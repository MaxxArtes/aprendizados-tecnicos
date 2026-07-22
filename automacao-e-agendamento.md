# Automação, jobs agendados e ETL

## Toda tarefa agendada precisa de um arquivo de status legível por humano

**Sintoma:** a automação parou de rodar semanas atrás e ninguém notou, porque "não
apareceu erro".

**Causa:** falha silenciosa é o modo de falha padrão de job agendado — quem não roda também
não avisa. E o agendador só registra o código de saída, que ninguém olha.

**Regra:** toda rodada grava um arquivo de status com timestamp, resultado e causa da
falha, e sai com código diferente de zero quando falha. O check de saúde vira "a data
avançou?" em vez de "achei algum erro?".

---

## O modo de morte mais comum de automação é a credencial expirar

**Sintoma:** 15 jobs vermelhos de uma vez; parece que a rede, o banco ou o servidor caiu.

**Causa:** falha de infraestrutura costuma ser parcial e heterogênea. Expiração de
credencial derruba tudo que compartilha o mesmo segredo, simultaneamente e com mensagem
idêntica.

**Regra:** antes de investigar rede, runner ou banco, leia o traceback de **um** job. Se
todos têm a mesma mensagem de login, é credencial. Se os jobs equivalentes em outro
ambiente (que usa outro segredo) seguem verdes, isso confirma.

---

## Teste a tarefa agendada **pelo agendador**, não pelo seu terminal

**Causa:** rodar pelo agendador usa outro usuário, outro ambiente e outras permissões.

**Regra:** dispare manualmente pelo agendador e confirme o resultado. É o único teste que
vale. Tarefa nunca executada é exatamente o tipo de coisa que falha em silêncio na
primeira vez que importa.

---

## Automação que aborta deve preservar o estado anterior

**Sintoma:** uma falha transitória de rede não só interrompe a atualização como destrói o
dado bom que já existia.

**Causa:** scripts costumam limpar o destino antes de repovoar. Se a origem responde vazio
— por token revogado, filtro errado ou API instável — o resultado é apagar tudo e não
escrever nada.

**Regra:** antes de sobrescrever, valide que a origem devolveu algo plausível. Resposta
vazia com sucesso HTTP é um caso a tratar explicitamente: aborte sem tocar no destino.

**Como verificar:** teste o caminho de erro de propósito, com credencial inválida. Se o
resultado for destino zerado, o desenho está errado.

---

## Grave o novo, valide, só então substitua o bom

**Sintoma:** uma execução com rede instável troca um backup íntegro por um corrompido.
Você só descobre no dia em que precisa restaurar.

**Regra:** grave como arquivo temporário, valide o formato, e só então substitua
atomicamente. Se a validação falhar, apague o temporário e mantenha o antigo — melhor um
backup velho e íntegro que um novo e quebrado.

---

## Todo coletor de lixo precisa de trava contra lista de referências vazia

**Sintoma:** uma rotina de limpeza de órfãos apaga o acervo inteiro.

**Causa:** a rotina monta o conjunto "o que deve existir" a partir do banco e apaga do
storage tudo que não está nele. Se a consulta falhar, voltar vazia ou a tabela ainda não
existir, **tudo** vira órfão.

**Regra:** antes de deletar, aborte se o conjunto de referências estiver vazio ou
implausivelmente pequeno. Adicione teto por execução e um modo dry-run que só lista.

---

## Reconciliação só deve tocar no que já esfriou

**Sintoma:** a rotina de conserto "conserta" itens que outro worker está processando neste
exato momento, duplicando trabalho e gastando chamadas pagas.

**Causa:** um item em processamento e um item abandonado têm o mesmo estado: "pendente".

**Regra:** filtre por idade — só reconcilie pendentes com mais de N minutos. A janela deve
ser maior que a duração máxima esperada de um lote normal.

---

## Grave um marcador sentinela para "processado, resultado vazio"

**Sintoma:** toda varredura reprocessa os mesmos itens indefinidamente, gastando chamadas
pagas.

**Causa:** "sem resultado" e "nunca processado" são indistinguíveis quando ausência de
linha é o único sinal.

**Regra:** ao processar e não encontrar nada, grave uma linha sentinela. As consultas de
negócio filtram a sentinela; as consultas de "o que falta processar" a respeitam.

---

## Log vazio de processo em execução não é sinal de travamento

**Sintoma:** tarefa marcada como "em execução" há horas com log de duas linhas; a tentação
é matar o processo.

**Causa:** scripts bufferizam a saída — enquanto o processo vive, o arquivo fica vazio ou
truncado. Tamanho de log não é medida de progresso.

**Regra:** antes de declarar travamento, junte três evidências independentes: (1) tempo de
**CPU acumulado** do processo — poucos segundos de CPU em muitos minutos significa bloqueio
em I/O, não travamento; (2) os artefatos de saída esperados já existem? (3) a duração
comparada com a média histórica da mesma tarefa. Só com as três, proponha o kill.

**Como verificar:** para log em tempo real, rode o interpretador em modo não-bufferizado.

---

## Condição de espera por "nenhum processo X rodando" nunca se satisfaz

**Sintoma:** um watcher fica preso para sempre esperando algo que já terminou.

**Causa:** máquina de trabalho tem processos do mesmo interpretador rodando o tempo todo.
Usar "não existe processo com esse nome" como sinal confunde o seu processo com os dos
outros.

**Regra:** espere pelo artefato que o trabalho produz, ou pelo PID específico que você
lançou. Nunca por ausência genérica de processo.

---

## Timeout no processo não mata o container

**Sintoma:** após um timeout, o processo local morre mas o container continua rodando,
segurando CPU, disco e o volume montado.

**Causa:** matar o cliente `docker run` não mata a carga no daemon.

**Regra:** sempre passe `--name <nome-único>` e, no tratamento de timeout, execute
`docker rm -f <nome>`. Gere o nome com sufixo aleatório para permitir execuções
concorrentes.

---

## Variável de CI com host de rede interna não resolve em job comum

**Sintoma:** pipeline **verde**, mas o passo de espelhamento não gravou nada; o erro é um
`EndpointConnectionError` perdido no meio do log.

**Causa:** a variável foi definida no nível de grupo com o nome de serviço do compose (que
só resolve dentro daquela rede). Em job que roda o processo direto no runner, esse nome não
existe. E o passo era best-effort, então não derrubou o build.

**Regra:** variáveis com endereço de rede interna nunca vão no nível de grupo. Crie
variável de **projeto** com a URL externa — a de projeto sobrescreve a de grupo. E qualquer
passo não-fatal precisa logar erro visível.

**Como verificar:** confira o **efeito** (o objeto apareceu no destino?), nunca a cor do
pipeline.

---

## Variável errada em nível de grupo continua sendo uma bomba, mesmo sobrescrita

**Sintoma:** tudo funciona, porque a variável de projeto correta tem precedência.

**Causa:** precedência esconde o valor ruim; quem apagar a de projeto no futuro cai
silenciosamente no valor errado.

**Regra:** corrija o valor no nível de grupo para que o fallback também esteja certo. Antes
de mexer, mapeie quem realmente consome aquela variável.

---

## Isole automação nova do pipeline crítico

**Sintoma:** um passo novo, útil porém secundário, é enxertado no pipeline que sustenta o
relatório mais importante — e passa a poder derrubá-lo.

**Regra:** rotina auxiliar (backup, espelhamento, exportação) roda **fora** do pipeline
crítico. Documente explicitamente que é exceção consciente à convenção e por quê. Antes,
confirme que ninguém consome o artefato que você vai sobrescrever.

---

## Priorize no backup o dado digitado à mão

**Sintoma:** um desastre destrói uma tabela pequena de parâmetros e a única forma de
recuperar é perguntar às pessoas o que estava lá.

**Causa:** o plano de backup cobria as tabelas grandes de pipeline — justamente as que são
reconstruíveis rodando a extração de novo.

**Regra:** classifique cada tabela em "reproduzível pelo pipeline" e "digitada
manualmente". A segunda é a que precisa de backup, mesmo tendo poucas linhas. Snapshot
manual **não é backup**: se não tem agendamento, envelhece.

**Como verificar:** faça o dump por leitura **somente-leitura** do banco, garantindo que a
rotina de backup nunca escreve na origem.

---

## Troca de fonte de dados se valida em tabelas paralelas

**Sintoma:** você troca a origem de um relatório e não tem como provar que melhorou.

**Causa:** escrever direto nas tabelas vivas destrói a base de comparação.

**Regra:** rode o pipeline novo gravando em tabelas de staging e meça a mesma métrica de
negócio nas duas versões. Só promova com o número na mão — e esse número é o que se
comunica ao stakeholder.

---

## Dado congelado: descubra qual fonte dirige a data antes de culpar o código

**Sintoma:** uma tabela de conciliação para de receber registros; a suspeita cai sobre o
pipeline que acabou de mudar.

**Causa:** a tabela era dirigida por **outra** fonte (um arquivo que alguém deposita numa
pasta), que parou de ser alimentada. As demais estavam frescas e só faziam enriquecimento.

**Regra:** num build com várias fontes, identifique qual define a **contagem de linhas e a
data máxima**; as outras só preenchem colunas. Quando a causa é operacional, o registro
precisa dizer isso claramente — é cobrança, não bug.

---

## Leitor de CSV novo pode trocar string vazia por NULL

**Sintoma:** depois de migrar a leitura para outro motor, JOINs e filtros por `= ''` param
de casar.

**Regra:** ao trocar o motor, compare o resultado campo a campo numa amostra e preserve
explicitamente a convenção anterior até que os consumidores sejam ajustados. Ler CSV
desconhecido com todas as colunas como texto evita inferência de tipo instável entre
arquivos.

---

## `str(asyncio.TimeoutError())` é string vazia

**Sintoma:** o job termina com `error: ""` e você não sabe o que aconteceu.

**Causa:** algumas exceções não têm argumentos.

**Regra:** em logging genérico, sempre `str(e) or type(e).__name__`.

---

## Sempre use execução destacada para tarefas longas em servidor

**Regra:** `nohup cmd &`, `tmux` ou `screen` — e redirecione a saída para arquivo, senão
você perde o log junto com a sessão.

---

## Serviço que manda a saída para `/dev/null` esconde prompt bloqueante

**Sintoma:** o serviço está "ativo", o processo existe, a conta é a certa — e mesmo assim
ele nunca faz o que deveria.

**Causa:** o programa parou numa pergunta interativa que nunca seria respondida. Como a
unit descartava a saída, o travamento foi mudo.

**Regra:** processo vivo **não** prova serviço funcional; a prova é o efeito observável. Ao
diagnosticar, redirecione temporariamente a saída para um arquivo, reinicie e leia. Quando
a causa for um diálogo de confiança, a solução é marcar o consentimento no arquivo de
configuração antes de subir o serviço.

---

## Só escreva em arquivo de configuração compartilhado sabendo como a outra automação o manipula

**Sintoma:** sua entrada some sozinha depois de um tempo.

**Causa:** outro processo regenera o arquivo inteiro em vez de fazer append.

**Regra:** leia o código do outro automatismo antes de editar à mão. Se ele reescreve o
arquivo, a edição precisa ir para a fonte dele.

---

## `rsync` sem `-c` torna o dry-run inútil depois de um clone novo

**Sintoma:** o `--dry-run` lista **todos** os arquivos como alterados.

**Causa:** por padrão o rsync compara tamanho e mtime; um clone recente reescreve o mtime
de tudo, mesmo com conteúdo idêntico.

**Regra:** em deploy por rsync a partir de um clone, use `-c` (checksum). Mais lento, porém
é o que dá um dry-run legível. Peça confirmação entre o dry-run e o envio, e valide o
destino ao final.
