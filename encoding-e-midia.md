# Encoding, texto e mídia

## Arquivo que engordou sem ganhar conteúdo é mojibake

**Sintoma:** texto vira "produÃ§Ã£o" no lugar de "produção", e o arquivo cresce ~6% sem
nenhuma edição real.

**Causa:** conteúdo UTF-8 lido como cp1252/latin-1 e regravado como UTF-8: cada caractere
acentuado vira dois bytes visíveis, e um BOM costuma ser acrescentado no processo.

**Regra:** é uma transformação determinística e portanto **reversível** — codifique de
volta para cp1252 e decodifique como UTF-8. Cuidado com 5 bytes que o cp1252 não mapeia
(`0x81`, `0x8D`, `0x8F`, `0x90`, `0x9D`): precisam de tratamento explícito ou a reversão
quebra. Antes de sobrescrever, meça: se sobrar `U+FFFD` depois de reverter, houve perda e
você precisa de outra fonte. Melhor ainda: se existe uma fonte íntegra, regenere a partir
dela em vez de consertar.

**Como verificar:** crescimento inexplicado de tamanho é o detector barato. Mas **nem todo
`Ã` é mojibake** — em português, `NÃO` e `PRODUÇÃO` em maiúsculas são legítimos. Procure as
sequências que só existem corrompidas (`Ã§`, `Ã£`, `Ãµ`, `â€`), não a letra isolada.

---

## Declare o encoding sempre, na leitura e na escrita

**Regra:** `encoding='utf-8'` explícito em toda abertura de arquivo. A codificação padrão
varia por sistema operacional, por locale e por versão da linguagem — depender dela é
depender de algo que muda de máquina para máquina.

---

## Extração de JSON embutido em texto: regex não-greedy trunca em array aninhado

**Sintoma:** o JSON emitido por um modelo é cortado no meio quando contém uma lista.

**Causa:** um padrão não-greedy para no primeiro `]` ou `}` encontrado, que pertence ao
array interno.

**Regra:** use padrão guloso delimitado pelas chaves externas, ou — melhor — um contador de
chaves. Valide com `JSON.parse` e só então aceite.

---

## `canvas.toBlob('image/webp', 1.0)` ativa modo sem perda e engorda o arquivo

**Sintoma:** converter uma foto de 3 MB gera 7 MB e leva segundos por imagem.

**Causa:** qualidade 1.0 em WebP significa **lossless**. Sobre um JPEG, lossless gasta bits
reproduzindo até os artefatos da compressão anterior.

**Regra:** nunca use 1.0. Sem perda só compensa em imagem que nunca foi comprimida.

---

## Recomprimir JPEG frequentemente **aumenta** o arquivo

**Sintoma:** o painel de otimização mostra algo como "3 MB → 4 MB" e ninguém entende.

**Causa:** arquivos que saem de câmera ou editor já vêm comprimidos (tipicamente qualidade
~85). Recodificar com qualidade **maior** que a original decodifica para pixels crus e
gasta bits reproduzindo os defeitos da compressão anterior.

**Regra:** trate compressão como otimização **condicional** — comprima, compare com o
original e fique com o menor. Se a recompressão não ajudou, o original também é o de melhor
qualidade. Só force a conversão quando o formato de origem não for exibível na web.

**Como verificar:** logue `bytesEntrada`/`bytesSaída` por arquivo e alarme quando a razão
for maior que 1.

---

## AVIF pode ser descartado por custo de codificação, não por tamanho

**Sintoma:** o pipeline de imagens fica lento após adotar um formato mais moderno.

**Causa:** ganho de tamanho pequeno em miniaturas (ordem de 10%) contra tempo de encode uma
ordem de grandeza maior.

**Regra:** ao escolher formato, meça as duas pontas — bytes entregues **e** tempo/CPU de
geração. Em pipeline que codifica centenas de arquivos, o encode domina o custo.

---

## Não cape resolução quando o destino final é impressão em grande formato

**Sintoma:** o time quer "otimizar" um acervo de imagens, sem saber para que elas são
usadas.

**Causa:** faça a conta antes de decidir — pixels da maior dimensão divididos pela largura
impressa em polegadas. Uma foto de câmera ampliada para alguns metros cai facilmente para
algumas dezenas de dpi, sem folga nenhuma. E artefato de JPEG **cresce** com a ampliação:
invisível na tela, quadriculado no papel.

**Regra:** decida compressão pelo destino do arquivo, não pelo peso. Sirva uma escada de
tamanhos e mantenha o original intocado para quem vai imprimir. Métricas objetivas de
similaridade medem mal o defeito que aparece no papel — não as use sozinhas para justificar
qualidade baixa.

---

## Compressão dupla não rende quase nada

**Sintoma:** você embrulha um backup em zip esperando economia relevante e ganha uns poucos
por cento.

**Causa:** packfile de git, JPEG, PNG, vídeo e a maior parte dos formatos modernos já
aplicam compressão. Uma segunda passada só adiciona cabeçalho e tempo de CPU.

**Regra:** o ganho real vem de **escolher o que não guardar**. E desconfie de otimização
cara: uma passada agressiva de repack pode custar minutos de CPU para economizar frações de
por cento.

**Como verificar:** meça antes e depois numa amostra antes de aplicar em tudo.

---

## Não extrapole tamanho de acervo a partir de amostra pequena

**Sintoma:** você calibra com poucos itens, encontra uma razão em torno de 0,8, promete um
número — e o real vem cerca de 20% acima, fora da faixa que você deu como garantida.

**Causa:** taxa de compressão varia muito com o conteúdo. Se a amostra pega um item grande
que comprime bem, a razão fica enviesada. E acervos costumam ser dominados por poucos itens
grandes, então o erro não se dilui.

**Regra:** para dimensionar espaço, use o tamanho que a própria origem reporta para o
conjunto inteiro. Amostra serve para estimar **tempo**, não volume.

**Como verificar:** compare a razão item a item, não só a agregada. Se ela varia bastante
entre os itens da amostra, não é uma constante que você possa aplicar ao todo.
