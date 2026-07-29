# Treino de modelos de IA

Lições de treinar um modelo de linguagem próprio do zero (pré-treino, avaliação,
quantização e a infraestrutura em volta). O contexto de origem é um modelo pequeno
(centenas de milhões de parâmetros) treinado com orçamento de dezenas de dólares —
regime em que cada erro custa dinheiro contado e nenhum desperdício se esconde.

## O pico de GPU do datasheet tem um asterisco que dobra o seu denominador

**Sintoma:** o MFU (aproveitamento da GPU) calculado dá ~15% e você gasta dias
perseguindo otimizações de throughput; nenhuma move o número de forma coerente.

**Causa:** o datasheet anuncia o pico de TFLOPs **com esparsidade estruturada** — um
recurso que treino denso não usa. O pico real (denso) é a metade. Dividir o throughput
medido pelo número de marketing reporta metade do MFU verdadeiro e faz um treino
saudável parecer quebrado.

**Exemplo concreto:** uma H100 anuncia 1979 TFLOPs bf16; o rodapé diz "*With
sparsity". O denso é 989. Um treino a 277 TFLOPs reais é 28% de MFU (normal para
modelo pequeno) — mas dividido por 1979 vira "14%", disparando uma caça a um problema
que não existe. O alvo também engana: modelo pequeno rende MENOS MFU em GPU nova
(o pico de cálculo cresceu 3,2× entre gerações; a banda de memória, só 1,7×), então
comparar com o MFU de um modelo grande em GPU antiga é outra fonte de pânico falso.

**Regra:** MFU usa o pico DENSO da própria GPU, e a referência de "MFU bom" tem que
vir de medição publicada com modelo de tamanho parecido no mesmo hardware.

**Como verificar:** procure o rodapé de esparsidade no datasheet; confira se
`tokens/s × FLOPs_por_token` bate com o TFLOPs que você está reportando.

## Hiperparâmetro copiado de outro projeto se converte pela escala do tensor

**Sintoma:** um otimizador que a literatura mede como 1,3-1,5× melhor perde do
baseline no seu treino; a loss dispara no início e depois estagna.

**Causa:** learning rate de métodos que normalizam a atualização só faz sentido
relativo à **escala de inicialização do tensor**. Se o projeto de origem inicializa o
embedding com desvio 1,0 e o seu usa 0,02, a mesma taxa "0,7" significa mover o seu
tensor **35 vezes a própria escala por passo** — destruição, não treino.

**Exemplo concreto:** multiplicadores de Muon copiados de um benchmark público
(embedding LR 0,7 com init std 1,0) aplicados a um GPT-2 com init std 0,02: o Muon
"perdeu" do AdamW (3,99 contra 3,64 de val loss) e o modelo empacava no passo 600.
Convertidos pela razão das escalas (multiplicador 23 em vez de 1000), o Muon venceu
com folga (2,95 contra 3,60). A leitura do código-fonte da origem — não do README —
foi o que revelou o init std 1,0.

**Regra:** antes de transplantar hiperparâmetro, leia o init do projeto de origem e
converta pela razão das escalas. O que transfere entre projetos são as RAZÕES entre
grupos de parâmetros, nunca os valores absolutos.

**Como verificar:** imprima `lr / peso.std()` por grupo nos dois projetos; se as
razões diferem por ordens de grandeza, a cópia está errada.

## Acurácia significativamente ABAIXO do acaso é bug do medidor, não modelo ruim

**Sintoma:** num benchmark de múltipla escolha com 4 alternativas, o modelo marca 19%
— e você anota "modelo fraco" e segue em frente.

**Causa:** modelo fraco fica **no** acaso (25%), porque chutar é o pior caso honesto.
Ficar vários desvios padrão abaixo exige um sinal sistemático — tipicamente o método
de pontuação lendo o prior das alternativas (comprimento, frequência) em vez da
resposta à pergunta, e escolhendo sistematicamente errado.

**Exemplo concreto:** ARC-Challenge em português com pontuação por soma de log-prob:
19,0% com n=400 está a 3 desvios abaixo do acaso — probabilidade menor que 0,2% de
ser azar. Trocando a pontuação para PMI (descontar a probabilidade incondicional da
alternativa), o mesmo modelo foi para 25,8%: no acaso, que é o valor honesto de um
modelo pequeno nessa tarefa.

**Regra:** abaixo do acaso com significância = investigar o scorer, nunca aceitar o
número. Para múltipla escolha, reportar também a variante PMI.

**Como verificar:** `z = (acc - acaso) / sqrt(acaso*(1-acaso)/n)`; z < -2 é alarme
de medidor.

## Comparar dois modelos por médias soltas esconde qualquer ganho pagável

**Sintoma:** você gasta um orçamento de treino, a acurácia vai de 36,8% para 39,5%,
e não há como afirmar se melhorou ou se foi sorte da amostra.

**Causa:** com n itens e acurácia ~40%, o menor ganho detectável entre duas medições
independentes é grande (com 400 itens, ~9,7 pontos). Mas os dois modelos podem
responder **os mesmos itens** — e aí só os desacordos carregam informação (teste de
McNemar), derrubando o limiar de detecção por 5-7×.

**Exemplo concreto:** avaliação com 400 itens não pareados: cego para tudo abaixo de
9,7 pontos — a faixa inteira onde vivem os ganhos de uma rodada incremental. Os
mesmos 2.076 itens avaliados par a par, guardando o vetor de acerto por item:
resolução de ~1,3 ponto, custo idêntico.

**Regra:** avaliação de modelo guarda o **vetor de acerto por item** (não só a
média), fixa o conjunto (hash dos itens junto do resultado) e compara com teste
pareado.

**Como verificar:** se o relatório da sua avaliação não permite responder "em
quantos itens A acertou e B errou?", ele não permite comparar modelos.

## Filtrar log com grep no cano faz job morto parecer job sem resultado

**Sintoma:** um lote de experimentos "roda" e o log mostra só os cabeçalhos de cada
braço, sem resultados nem erros. Conclusão tentadora: os experimentos não acharam
nada.

**Causa:** `comando | grep padrao` descarta tudo que não casa — inclusive o
traceback do crash. Se o processo morre no import, o grep engole a evidência e o
código de saída se perde no meio do cano. O log fica idêntico a "rodou e não
imprimiu nada".

**Exemplo concreto:** cinco braços de uma varredura de hiperparâmetros morreram no
`import torch` (imagem de container sem a biblioteca). O log do job mostrava os
cinco cabeçalhos e nada mais — parecia varredura concluída sem achados. Custo: uma
rodada de máquina alugada e a conclusão errada quase registrada.

**Regra:** saída de job vai INTEIRA para arquivo; o console recebe só o resumo
filtrado; o código de saída é checado explicitamente e, em falha, as últimas linhas
do arquivo são impressas.

**Como verificar:** mate um braço de propósito (import inexistente) e confira se o
log do orquestrador denuncia a falha.

## Tokenizer é parte do checkpoint — separado, o modelo vira lixo ilegível

**Sintoma:** um checkpoint treinado com sucesso gera texto sem sentido ao ser
carregado em outra máquina, com perplexidade pior que a de um modelo aleatório.

**Causa:** os pesos do modelo só significam algo em relação ao vocabulário exato com
que foram treinados. Se o tokenizer foi treinado dentro do job e ficou no container
destruído (ou existe outro arquivo de mesmo vocabulário mas merges diferentes), os
IDs apontam para outros tokens e não há erro nenhum — só saída degenerada.

**Exemplo concreto:** um checkpoint de 219M params ficou irrecuperável porque o BPE
foi treinado dentro do job e o script de persistência só subia o diretório de pesos.
Dois tokenizers do mesmo projeto, ambos com vocabulário 16384: um decodificava o
corpus como português; o outro, como "Suúc melanc Quandora" — e nada além de
decodificar uma amostra revelaria qual era o certo.

**Regra:** tokenizer viaja SEMPRE junto do checkpoint (mesmo diretório, mesmo
upload) e vive versionado no repositório. Na dúvida entre dois, decodifique uma
amostra do corpus com cada um — não confie em nome de arquivo.

**Como verificar:** o script de persistência falha (não avisa: FALHA) se o
tokenizer não estiver ao lado dos pesos.

## Storage que devolve erro como conteúdo passa por download bem-sucedido

**Sintoma:** downloads "funcionam" (sem exceção), mas os arquivos têm 22 bytes; ou
um treino começa e quebra minutos depois com dado truncado.

**Causa:** alguns storages, em pane, respondem à API de download com um corpo JSON
de erro e status de sucesso. O cliente grava o JSON como se fosse o arquivo. Todo
código que só confere "download não lançou exceção" segue em frente com lixo.

**Exemplo concreto:** o storage de um provedor de GPU ficou 24h+ com a leitura
quebrada devolvendo `{"error":"not found"}` como conteúdo — inclusive para arquivos
baixados com sucesso horas antes. Um vigia ingênuo teria sobrescrito um checkpoint
bom de 1,1 GB com 22 bytes e reportado sucesso.

**Regra:** depois de todo download, validar o CONTEÚDO contra o esperado: tamanho
mínimo, magic bytes ou contagem derivada de um metadado independente. Preferir
storage sob seu controle (com egress zero) como fonte primária de artefatos de
treino; o storage do provedor de compute é cache, não casa.

**Como verificar:** `head -c 16` do arquivo baixado começa com `{"error"` ou tem
tamanho abaixo do plausível → a pane é do storage, e o seu código deveria ter
recusado.

## Dataset pequeno repetido muitas épocas vira decoreba, não estilo

**Sintoma:** depois do fine-tuning de conversa, o modelo responde qualquer pergunta
com uma resposta literal do conjunto de treino ("Roda `docker ps`" para "bom dia").

**Causa:** poucas centenas de exemplos repetidos dezenas de épocas não ensinam o
padrão — o modelo memoriza as sequências. A memorização verbatim dispara ANTES da
loss de validação denunciar, então a curva parece saudável enquanto o modelo decora.
As receitas publicadas para modelos pequenos usam centenas de milhares de exemplos
ÚNICOS por 2-3 épocas; ordens de magnitude menos diversidade não se compensam com
mais épocas — pioram com elas.

**Exemplo concreto:** fase de estilo com 474 diálogos × 20 épocas: o modelo passou a
cuspir respostas inteiras do treino para entradas não relacionadas. A referência de
mesma escala (SmolLM2-135M/360M) usa ~460 mil exemplos únicos × 2 épocas — mil vezes
mais diversidade, dez vezes menos repetição.

**Regra:** em SFT, diversidade ≫ repetição: mirar 10⁵+ exemplos únicos e ≤3 épocas;
monitorar sobreposição de n-gramas entre o que o modelo gera e o treino como teste
de regressão (não confiar na val loss para pegar decoreba).

**Como verificar:** gere 50 respostas e meça o maior n-grama compartilhado com o
conjunto de treino; sequências longas idênticas = decorou.
