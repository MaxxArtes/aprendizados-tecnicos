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

## Chamada de rede sem timeout no loop de treino trava o treino inteiro, não só a telemetria

**Sintoma:** o painel de acompanhamento do treino para de atualizar por dezenas de
minutos; o job continua marcado como "rodando" e a conta continua sendo cobrada, mas
não há como saber se o modelo está progredindo ou travado.

**Causa:** uma chamada de rede síncrona dentro do laço principal de treino (upload de
telemetria, checkpoint, o que for) sem `connect_timeout`/`read_timeout` explícitos pode
ficar pendurada indefinidamente numa rede degradada, sem lançar exceção — e como está
na MESMA thread que faz o passo de treino, ela trava o treino junto, não só o aviso.
Um `try/except Exception` ao redor não ajuda: a exceção nunca chega, porque a chamada
nunca retorna.

**Exemplo concreto:** a função de publicar progresso subia primeiro pro storage próprio
(rápido, sempre funcionou) e depois, como reforço redundante, pro Drive do provedor de
GPU — um caminho que já era conhecido como quebrado para LEITURA. A chamada de escrita
nesse segundo caminho, sem timeout, ficou pendurada 90+ minutos numa rede instável. Só
não passou despercebido porque o checkpoint (que sobe por outro caminho, o storage
próprio) continuou avançando normalmente enquanto a telemetria ficava parada — a
discrepância entre os dois é que revelou o problema.

**Regra:** toda chamada de rede dentro do laço de treino tem `connect_timeout` e
`read_timeout` explícitos, curtos o bastante pra nunca competir com o próprio passo de
treino em duração. Caminho redundante que já é sabidamente instável não entra "só por
garantia" — reforço que pode travar é reforço negativo.

**Como verificar:** se dois caminhos de telemetria dependem de provedores diferentes,
compare os timestamps dos dois; se um avança e o outro trava, o travado é que está
bloqueando o laço, não um problema de rede aleatório.

## Retomar um checkpoint já decaído com LR no pico piora o modelo, não melhora

**Sintoma:** um teste rápido e barato de "será que mais treino ainda ajuda" — retomar
um checkpoint já treinado e rodar mais um pouco — devolve o modelo PIOR do que estava,
às vezes com geração de texto degenerada.

**Causa:** um cronograma WSD (warmup-stable-decay) reiniciado do zero (LR volta ao
PICO total de propósito — é assim que o mecanismo funciona) desconverge o modelo
esperando que a fase de decaimento reconverja pra um ponto melhor que o anterior. Se
o novo ciclo for curto, a fase de decaimento (uma fração fixa do NOVO orçamento, não
do original) também é curta — curta demais pra sequer voltar aonde o modelo já
estava, quanto mais melhorar. O teste barato mede "o quanto o ciclo curto disruptiu
e recuperou parcialmente", não "quanto o modelo ainda tem de margem".

**Exemplo concreto:** um checkpoint com val loss 2,55 (decaído ao fim de um treino de
8h) retomado com `RESET_SCHEDULE=1` e orçamento de 15 minutos: o LR voltou ao pico,
a val loss saltou pra ~3,0-3,09 (nível de antes do decaimento original), a fase de
decaimento desse ciclo durou só ~180s contra ~5.760s do decaimento original, e o
treino terminou com val 2,81 — pior que o ponto de partida — e uma amostra de texto
degenerada (sequência de sublinhados).

**Regra:** um teste de extensão barato só é informativo se a fase de decaimento do
NOVO ciclo for proporcional (em tempo absoluto, não em fração) à do ciclo original —
ou se o LR de retomada for bem mais baixo que o pico original, evitando o
re-aquecimento disruptivo. Testar a hipótese "ainda tem margem" com um orçamento
pequeno demais não economiza dinheiro — devolve um resultado sem informação (nem
confirma nem refuta) e ainda soa como confirmação de que "não tem mais margem" pra
quem não souber a causa.

**Como verificar:** compare a duração da fase de decaimento do teste (fração do
orçamento × orçamento) contra a duração da fase de decaimento do treino original em
segundos absolutos — se for uma fração pequena dela, o teste não vai reconvergir a
tempo, e o resultado não deve ser usado pra decidir nada.

## Medir duplicata por amostragem é estatisticamente cego

**Sintoma:** a taxa de duplicatas entre duas fontes de corpus, medida por amostra, dá
~0% — e uma passagem completa depois encontra dezenas de milhares de documentos
repetidos.

**Causa:** a interseção de duas amostras é um estimador viciado para baixo: um documento
presente nas duas fontes só é detectado se cair nas **duas** amostras ao mesmo tempo.
Amostrando 3% de cada lado, a chance de detectar cada duplicata é ~0,1% — a medição
inteira é ruído.

**Exemplo concreto:** duas fontes derivadas de Common Crawl, amostra de 160 mil
documentos por fonte: 0,062% de sobreposição medida. Passagem completa com tabela única
de hashes: **1,49%** — 24× mais, 72 mil documentos. A mesma amostragem também subestimou
a duplicata interna de uma fonte de PDFs: 16,6% medido contra 23,9% real.

**Regra:** taxa de duplicata se mede com passagem completa e tabela única. Amostra serve
para estimar propriedades de documentos individuais, nunca de PARES de documentos.

**Como verificar:** se a medição por amostra deu quase zero, rode a passagem completa em
um subconjunto pequeno mas INTEIRO (uma fonte contra outra) e compare as ordens de
grandeza.

## Lista Python para acumular tokens custa 20× a RAM do dado

**Sintoma:** um processamento de corpus morre por falta de memória acumulando um buffer
que "deveria" ter algumas centenas de MB.

**Causa:** cada `int` de token numa lista Python é um objeto de 28 bytes mais 8 de
ponteiro (IDs acima de 256 não são cacheados) — 36-40 bytes por token que no arquivo
final ocupa 2 (`uint16`). O buffer de 80 MB no disco são ~1,5 GB na lista.

**Exemplo concreto:** buffer de 40 milhões de tokens antes de gravar: ~1,4 GB em lista,
estourou o teto de memória do processo e matou a montagem do corpus — silenciosamente,
porque o código de saída foi mascarado por outro defeito. Trocado por `array("H")`: os
mesmos 40 milhões em 80 MB.

**Regra:** buffer de tokens é `array` do módulo `array` (ou `np.ndarray`), nunca lista.
`array("H")` para vocabulário até 65k; a gravação vira `np.frombuffer(...).tofile()`.

**Como verificar:** `sys.getsizeof` de uma lista de 1 milhão de tokens contra
`array("H")` com o mesmo conteúdo — a razão é ~20×.

## Tokenizar documento a documento usa um núcleo; o lote usa todos

**Sintoma:** máquina de 16 núcleos tokenizando corpus com carga ~1,0; pagou-se por
paralelismo que não acontece.

**Causa:** `tokenizer.encode(texto)` da biblioteca `tokenizers` é monothread. O
paralelismo da biblioteca (Rust + rayon) só é acionado por `encode_batch(lista)`.

**Exemplo concreto:** montagem de corpus em máquina de 16 vCPU: carga 0,95, 200 milhões
de tokens em 5 minutos. Trocando o laço documento-a-documento por `encode_batch` sobre o
lote já filtrado: 3,5× mais rápido, sem mudar mais nada.

**Regra:** filtre e acumule documentos primeiro, tokenize em lote depois. `encode` um a
um só para depuração.

**Como verificar:** `uptime` durante a tokenização — carga próxima de 1 numa máquina
multi-núcleo denuncia o laço serial.

## Sob DDP, preparar arquivo compartilhado sem eleger um rank é corrida armada

**Sintoma:** treino multi-GPU morre no arranque com `FileNotFoundError` num arquivo
`.tmp` que o próprio código acabou de criar; em uma GPU funciona sempre.

**Causa:** os N processos do DDP executam o MESMO código de preparação (concatenar
shards, gerar cache, converter dado). Todos escrevem o mesmo arquivo temporário; o
primeiro `os.replace` leva o arquivo, os demais falham — ou pior, sobrescrevem-se em
silêncio.

**Exemplo concreto:** concatenação de 123 shards de corpus num `.cat` único: 4 ranks
construíram o mesmo `.tmp` ao mesmo tempo e o lançamento inteiro caiu. O teste de
bancada não pegou porque rodou em processo único — teste que não cobre a concorrência
real aprova código quebrado.

**Regra:** preparação de arquivo compartilhado é trabalho do rank 0; os demais esperam o
arquivo COMPLETO aparecer (existência + tamanho esperado), com timeout. Nome temporário
leva o PID por segurança.

**Como verificar:** rode a preparação com 2 processos locais (`torchrun
--nproc_per_node 2`, backend gloo, em CPU) antes de pagar GPU — a corrida aparece de
graça.

## Ganho de recusa medido só do lado negativo pode ser limiar, não aprendizado

**Sintoma:** depois de um fine-tuning, a taxa de recusa em perguntas SEM resposta sobe
(ex.: 34% → 48%) e parece aprendizado — mas o modelo continua inútil na prática.

**Causa:** medir só os negativos não distingue "aprendeu a julgar respondibilidade" de
"ficou mais conservador em tudo". Um deslocamento global do limiar sobe a recusa nos
negativos E nos positivos ao mesmo tempo — e a segunda metade fica invisível se o
conjunto de avaliação não tem positivos do MESMO domínio dos negativos.

**Exemplo concreto:** recusa em negativos humanos subiu de 34,00% para 47,75% e foi
celebrada. O controle com respondíveis do mesmo corpus mostrou 42,0% de recusa indevida
contra 46,7% de correta — discriminação de 4,7 pontos, p=0,41. O modelo recusava ~44% de
TUDO. Duas hipóteses mecanicistas (familiaridade de domínio; confiança sensível a
domínio) foram refutadas pela mesma tabela.

**Regra:** avaliação de recusa/abstenção exige os quatro quadrantes: negativos e
positivos do MESMO domínio. O número que importa é a DIFERENÇA entre recusa nos
negativos e recusa nos positivos, com teste de significância — não a taxa isolada.

**Como verificar:** se o seu conjunto de avaliação de recusa não tem positivos da mesma
fonte dos negativos, o resultado é inconclusivo por construção — monte o controle antes
de celebrar.

## SFT de tarefa sem replay do pré-treino apaga a língua que pagou caro

**Sintoma:** depois de um fine-tuning de tarefa (SFT), a métrica-alvo fica ótima — e o
benchmark de língua desaba (no nosso caso, CALAME de ~50 para ~17; noutra base, 53 para
12). Ninguém nota, porque só a métrica da tarefa foi medida.

**Causa:** esquecimento catastrófico. O SFT só apresenta a distribuição estreita da
tarefa; os pesos que sustentavam a competência geral são sobrescritos sem nada que os
defenda. O mesmo mecanismo destrói a tarefa ANTERIOR num segundo SFT sequencial
(intenção 86 → 44).

**Exemplo concreto:** mesmo SFT, mesma dose de dado, num modelo de 269M: sem replay,
CALAME 18,11 e 5-shot 19,33; misturando 25% de dado do pré-treino ao lote do SFT,
CALAME 42,97 e 5-shot 41,33 — com a execução da tarefa intacta (intenção 87,2 → 87,4,
F1 74,9 → 75,5). O replay não custou nada mensurável na tarefa.

**Regra:** todo SFT leva replay do dado de pré-treino (25% funcionou; trate como piso a
ajustar), e a bateria de avaliação do SFT inclui SEMPRE um benchmark de competência
geral, não só a métrica da tarefa.

**Como verificar:** rode o benchmark de língua no modelo antes e depois do SFT. Queda
de dezenas de pontos com a tarefa "ótima" é o sintoma exato.

## Modelo base avaliado em tarefa de turno responde e depois "vira o usuário"

**Sintoma:** um modelo base (sem SFT) parece incapaz numa tarefa que ele domina: F1
na metade do esperado, respostas que começam certas e terminam em lixo.

**Causa:** base é completador de texto: ela responde e SEGUE completando — inventa o
próximo turno do usuário, outra pergunta, outra resposta. Se o medidor pontua a geração
inteira, o apêndice inventado dilui a resposta certa. Não é limitação do modelo; é o
medidor sem critério de parada.

**Exemplo concreto:** F1 de extração de campos de um 269M sem SFT: 37,96 medido cru.
Cortando a geração no marcador do próximo turno: 73,65. O número dobrou sem tocar no
modelo — e três conclusões de projeto tiradas do número cru estavam erradas.

**Regra:** avaliação de modelo base em tarefa de turno exige corte explícito no
marcador de próximo turno (ou terminador equivalente) antes de pontuar. Registrar na
régua COMO a parada foi feita; números com e sem corte são incomparáveis.

**Como verificar:** leia 10 gerações cruas antes de aceitar qualquer métrica de base.
Se aparecer um segundo turno inventado, o medidor está pontuando lixo.

## Juiz-LLM é instrumento com ruído próprio — meça o ruído antes de ler o ponteiro

**Sintoma:** o score do juiz automático muda de uma rodada para outra com a MESMA
entrada e temperatura 0; diferenças de poucos pontos entre modelos são celebradas ou
lamentadas conforme a rodada.

**Causa:** julgamento por LLM tem variância própria (infra, desempate de tokens,
sensibilidade a formatação) mesmo "determinístico". Com n pequeno, a granularidade por
item soma-se ao ruído: com n=15, cada item vale 6,7 pontos.

**Exemplo concreto:** 5 rodadas do mesmo juiz binário sobre as mesmas saídas: ±9 pontos
percentuais de ruído típico, com excursão até 25. Uma diferença de 26 pp em
"honestidade" entre dois candidatos era só 3 aceites de distância (10 vs 13 em n=15) —
sinal fraco, não veredito.

**Regra:** antes de usar um juiz-LLM para decidir, rode-o 3–5 vezes sobre as mesmas
saídas e registre a excursão. Diferença menor que o ruído medido não é sinal; decisão
de treino não se toma por ela. Aumente o n antes de aumentar a convicção.

**Como verificar:** repita o juiz sobre um conjunto fixo. Se você não sabe o ruído do
seu juiz, você não sabe ler o placar dele.

## Dado raro repetido: aprende até ~3 épocas, memoriza aos ~5, deixa de valer aos ~40

**Sintoma:** repetir o dado mais valioso do corpus por muitas épocas parece de graça
("é o dado bom!") — o loss agradece, e o modelo piora onde importa.

**Causa:** o valor de um conjunto pequeno e raro se esgota rápido: depois de poucas
épocas o modelo decora superfícies em vez de aprender o padrão, e o excesso ainda
desloca o que o resto do treino tinha construído.

**Exemplo concreto:** curva de dose medida num 269M: aprendizado puro até ~2,7 épocas;
primeiros sinais de memorização em ~5 (o 5-shot cai primeiro); valor zera por volta de
~40. Dois runs que passaram disso (81 e 148 épocas do mesmo dado na fase de decay)
saíram piores em língua — desastre previsto pela curva e confirmado.

**Regra:** orce as ÉPOCAS do dado raro, não a fração da mistura: fração fixa com budget
de tempo diferente muda as épocas silenciosamente. Fique ≤ 3 épocas para aprender;
trate 5+ como zona de memorização; 40+ é desperdício com dano.

**Como verificar:** acompanhe uma métrica de generalização (few-shot, benchmark de
língua) junto com o loss durante a fase repetida — a queda do few-shot denuncia a
memorização antes do loss contar qualquer coisa.

## O cronograma de LR tem que ser dirigido pelo limite que aperta primeiro

**Sintoma:** o treino roda inteiro, termina no número de passos combinado, e a curva de
perda fica plana até o fim — sem a queda acentuada que o cronograma prometia. O log mostra
a taxa de aprendizado no valor de pico do começo ao fim, mas nenhuma configuração parece
errada, e o mesmo código já produziu a queda em treinos anteriores.

**Causa:** o laço de treino tem **dois** limites — um teto de passos e um orçamento de
tempo — e o cronograma foi escrito para se guiar por apenas um deles. Um cronograma
warmup-estável-decaimento decide entrar na fase de decaimento quando a fração de progresso
passa de um limiar. Se essa fração é calculada por *tempo decorrido ÷ orçamento de tempo*
mas o run termina por *teto de passos*, a fração nunca chega perto do limiar: o treino
inteiro acontece no platô e a fase onde a maior parte da queda acontece simplesmente não
ocorre. O código está correto quando o run para por tempo e silenciosamente errado quando
para por passos. Nada avisa: não há exceção, não há aviso, e o run "termina com sucesso".

**Exemplo concreto:** um run configurado com orçamento de 69 horas mas com teto de passos
que se esgota em 9 horas termina com a fração de progresso em 0,13. O decaimento começaria
em 0,80. O run de comparação — mesmo código, mesma função — tinha orçamento de 18.000 s e
durou 18.024 s: parou por tempo, a fração chegou a 1,0 e o decaimento aconteceu inteiro.
Comparar a perda dos dois é comparar um modelo resfriado com um modelo ainda quente, e a
diferença (da ordem de 0,1 de perda de validação) seria creditada à mudança que se queria
medir.

**Regra:** quando o laço tem dois limites, a fração de progresso é o **máximo** das duas
frações — o limite que apertar primeiro assume sozinho. Não deixe isso depender de alguém
lembrar de passar uma variável de ambiente na linha de lançamento.

```python
frac_passo   = min(1.0, passo / max(1, MAX_PASSOS))
frac_relogio = min(1.0, decorrido / max(1.0, ORCAMENTO_S))
frac = max(frac_passo, frac_relogio)
```

**Como verificar:** imprima a taxa de aprendizado **efetiva** (a que foi para o otimizador)
e a fração de progresso em toda avaliação. Antes de lançar, calcule qual dos dois limites
será atingido primeiro e confira se a fração chega a 1,0 nesse ponto.

## Uma variável de estado com dois consumidores distantes: conserte o produtor

**Sintoma:** você identifica um valor calculado errado, aplica a correção no lugar onde o
sintoma aparece, confirma que o sintoma sumiu — e o experimento continua inválido por um
motivo que não estava sendo procurado.

**Causa:** a variável acumulou a semântica de "progresso geral" e foi injetada em domínios
diferentes do sistema. No caso de um treino: a mesma fração governa a fase de decaimento do
otimizador **e** o momento em que o carregador de dados troca a mistura do corpus. Corrigir
o valor pelo lado do otimizador é *shotgun surgery*: você conserta o efeito visível e o
outro consumidor continua lendo o valor errado, sem emitir sinal nenhum.

**Regra:** defeito preso a uma variável de estado exige enumerar **todos** os consumidores
dela antes de consertar (`grep` do nome, não do sintoma), e o conserto vai no **produtor**
do valor. Conserto aplicado a um consumidor é parcial por definição.

**Como verificar:** depois de consertar, procure a variável de novo e confirme que cada
ocorrência lê o valor corrigido. Se o conserto foi aplicado por fora (multiplicador,
sobreposição, remendo em tempo de execução), ele quase certamente alcança um consumidor só.

## Todo processo de execução longa nasce com superfície de controle

**Sintoma:** um job de horas — treino, indexação, migração — está com um parâmetro errado.
A única saída visível é matar o processo e recomeçar do último checkpoint, perdendo tempo
já pago e assumindo o risco da retomada.

**Causa:** parâmetros lidos do ambiente na importação viram constantes de módulo. Depois
disso não há como alterá-los de fora: mudar variável de processo vivo exige `gdb`/pyrasite
e a capacidade `CAP_SYS_PTRACE`, que a maioria dos contêineres de nuvem não concede e que
não se adiciona a contêiner já em execução.

**Regra:** todo laço longo relê um arquivo (local ou de bucket) a cada N iterações e aplica
um conjunto **fechado** de chaves. São ~15 linhas, e é a diferença entre um comando e uma
retomada. Três detalhes que só aparecem no uso:

- **chave ausente ≠ "não deu para ler"**: falha de rede tem que manter o que estava
  valendo, enquanto ordem removida restaura o padrão. Tratar os dois casos igual faz o
  "limpar" mentir — o parâmetro alterado continua valendo até o fim do run;
- **parâmetro de rota usa "não mexer" explícito** (um sentinela como `-1`), não padrão de
  fábrica: limpar a ordem não pode desfazer uma correção feita no meio do processo. E o
  valor "desligado" precisa ser distinguível de "ausente";
- **em execução distribuída, só o rank 0 lê e o resultado é transmitido** — e o vetor
  transmitido tem que ter o mesmo tamanho em todos os ramos do código, senão a barreira
  trava com as GPUs a 100% sem fazer nada.

**Como verificar:** confirme a ordem pelo log do **alvo** (uma linha do tipo
`{"controle_recebido": ...}`), nunca pela mensagem de sucesso da ferramenta que enviou.
E registre no log a **transição** "havia ordem → não há mais": sem ela, uma pausa cancelada
continua sendo a última notícia e quem monitora não distingue pausa de travamento.
