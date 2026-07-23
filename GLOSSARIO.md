# Glossário

As lições usam o vocabulário técnico de verdade, sem simplificar. Este glossário existe para
que isso não seja barreira.

**Por que não trocar os termos por linguagem simples:** porque você vai encontrá-los assim.
Na documentação, na mensagem de erro, na revisão de código e na reunião em que alguém diz
"isso não é idempotente" e todo mundo assente. Aprender o termo junto com o conceito é o que
permite participar dessa conversa. Explicação mastigada resolve o problema de hoje e deixa
você de fora amanhã.

Não é para decorar. É para desbloquear a leitura quando você esbarra numa palavra que todo
mundo usa como se você já soubesse — e para você reconhecê-la da próxima vez.

---

## Fundamentos que aparecem em todo lugar

**Endpoint** — um endereço específico da sua API que faz uma coisa (`POST /pedidos`,
`GET /usuarios/42`). "Expor um endpoint" é disponibilizar essa URL.

**Handler** — a função que efetivamente responde a uma requisição num endpoint. Quando se diz
"o handler engoliu o erro", é essa função.

**Payload** — o conteúdo que vai no corpo de uma requisição ou de um evento, normalmente JSON.
"Não confie no payload" significa: quem manda pode mandar qualquer coisa.

**Token** — string que prova quem você é ou o que você pode fazer. Funciona como uma chave
física: quem tiver, usa. Por isso não se coloca token em log, URL ou repositório.

**Sessão** — o estado "esta pessoa está logada", geralmente guardado num cookie no navegador e
conferido a cada requisição.

**Cookie** — pedacinho de dado que o navegador guarda por site e reenvia automaticamente a
cada requisição. É onde a sessão normalmente vive.

**Escopo** — o alcance de algo. Escopo de um token é o que ele pode fazer; escopo de uma
variável é onde ela vale. "Escopo amplo demais" é quase sempre um problema de segurança.

**Middleware** — código que roda **entre** a chegada da requisição e o seu handler. Bom para
autenticação e redirecionamento; perigoso quando o filtro pega rotas demais.
↳ [frontend-e-nextjs.md](frontend-e-nextjs.md)

**Timeout** — tempo máximo de espera antes de desistir. Sem timeout explícito, uma chamada
travada pendura o seu processo inteiro.

**Retry** — tentar de novo depois de falhar. Só faz sentido para erro transitório (rede, 429,
5xx); repetir um erro de validação é desperdício.

**Fallback** — o plano B quando o principal falha. "Degradação" é o mesmo com outro nome:
entregar menos, em vez de quebrar.

**Efeito colateral** — qualquer coisa que a função faz além de devolver um valor: gravar no
banco, cobrar um cartão, mandar e-mail. É o que torna repetir uma operação perigoso.

---

## Web e navegador

**CORS** — regra do **navegador** que impede um site de ler dados de outro domínio sem
permissão explícita. Exibir uma imagem de outro domínio é permitido; *ler os bytes* dela
exige CORS. Por isso um teste com `curl` nunca reproduz erro de CORS: `curl` não é navegador.
↳ [apis-e-integracoes.md](apis-e-integracoes.md)

**Preflight** — antes de certas requisições cross-origin, o navegador manda sozinho um
`OPTIONS` perguntando "posso?". Se a resposta não autorizar, a requisição real nem sai.

**Hidratação** — o servidor manda HTML pronto (rápido de mostrar, morto) e o JavaScript
"acorda" essa página no navegador, ligando os botões. Enquanto não hidratou, a tela aparece
mas nada funciona.
↳ [frontend-e-nextjs.md](frontend-e-nextjs.md)

**Ativação transitória** — permissão temporária que o navegador dá logo depois de um clique
real. Algumas APIs (compartilhar, abrir janela, tela cheia) só funcionam dentro dessa janela.
Um `await` no meio a consome, e a API passa a ser recusada.
↳ [frontend-e-nextjs.md](frontend-e-nextjs.md)

**Resposta opaca** — quando o navegador busca um recurso sem CORS, ele guarda no cache uma
resposta que consegue *exibir* mas não deixa o código *ler*. O `fetch` seguinte reaproveita
essa entrada e falha, mesmo com o servidor configurado certo.
↳ [frontend-e-nextjs.md](frontend-e-nextjs.md)

**Bundler / empacotador** — ferramenta que junta e transforma seu código-fonte no pacote que
o navegador recebe. É quem substitui variáveis de ambiente por valores literais.

**JSX** — a sintaxe que permite escrever marcação parecida com HTML dentro de JavaScript.
Dentro de uma tag JSX as regras de comentário mudam, o que quebra builds.

**`srcset` e o descritor `w`** — lista de versões da mesma imagem com a largura de cada
arquivo, para o navegador escolher a adequada à tela. O `w` é a **largura real em pixels**,
não o lado maior.
↳ [frontend-e-nextjs.md](frontend-e-nextjs.md)

**Especificidade CSS** — critério de desempate quando duas regras atingem o mesmo elemento.
Empatou na especificidade, ganha a que vier **por último** no arquivo gerado.
↳ [frontend-e-nextjs.md](frontend-e-nextjs.md)

**`vh` versus `dvh`** — `1vh` é 1% da altura da tela considerando a barra do navegador sempre
visível; `1dvh` acompanha a barra entrando e saindo. No celular, `100vh` estoura.
↳ [frontend-e-nextjs.md](frontend-e-nextjs.md)

**Polling** — perguntar repetidamente "já mudou?" em vez de ser avisado. Simples e confiável,
porém caro e com atraso.

**Heartbeat** — sinal periódico enviado só para dizer "ainda estou aqui". Usado para medir
tempo de sessão e detectar quem sumiu.

---

## APIs e integrações

**Webhook** — o serviço de terceiro chama **você** quando algo acontece, em vez de você ficar
perguntando. Como qualquer um pode chamar essa URL, ela precisa validar quem chamou.
↳ [apis-e-integracoes.md](apis-e-integracoes.md)

**OAuth** — protocolo em que o usuário autoriza seu app a agir em nome dele num outro serviço,
sem entregar a senha. É o "entrar com…".

**Callback** — a URL para onde o serviço externo devolve o usuário (ou o resultado) ao fim de
um fluxo. Precisa estar registrada no provedor, ou o fluxo falha.

**SDK** — biblioteca oficial de um fornecedor que embrulha a API dele. Facilita, mas esconde:
quando quebra, o erro vem do SDK e não da sua lógica.

**Rate limit** — teto de quantas requisições você pode fazer num período. Costuma ter duas
dimensões independentes: número de chamadas **e** volume de dados.
↳ [ferramentas-de-ia.md](ferramentas-de-ia.md)

**Backoff exponencial** — ao receber erro, espere antes de tentar de novo, dobrando a espera a
cada tentativa. Evita que sua repetição piore a sobrecarga que causou o erro.
↳ [ferramentas-de-ia.md](ferramentas-de-ia.md)

**Falha aberta / falha fechada** — quando um mecanismo de proteção quebra, ele libera (aberta)
ou bloqueia (fechada)? Proteção contra abuso costuma falhar aberta; controle de gasto,
fechada.
↳ [apis-e-integracoes.md](apis-e-integracoes.md)

**Fila** — lista de coisas a processar, consumida por um ou mais trabalhadores. Provedores de
webhook mantêm uma fila de entrega e a **pausam** se você errar demais.

**Worker** — processo (ou thread) que consome trabalho de uma fila.

**Namespace** — prefixo que separa coisas de origens diferentes que dividem o mesmo espaço.
`app1::pedido::42` não colide com `app2::pedido::42`.

**Latência** — o tempo que a mensagem leva para ir e voltar. Diferente de throughput, que é
quanto passa por segundo: uma conexão pode ser rápida e lenta ao mesmo tempo.

**Round-trip** — uma ida e volta completa até o servidor. Vinte consultas em série significam
vinte round-trips somados; o mesmo trabalho em paralelo custa um.

---

## Banco de dados

**Transação** — bloco de operações que acontece inteiro ou é desfeito (`BEGIN` … `COMMIT` /
`ROLLBACK`). É como você evita deixar dados pela metade.

**Atômico** — operação que acontece inteira ou não acontece; ninguém observa o meio dela. Ler,
calcular e escrever em três passos **não** é atômico, e é aí que dois clientes compram o mesmo
último item.
↳ [banco-de-dados.md](banco-de-dados.md)

**Lock** — trava que impede dois processos de mexerem na mesma coisa ao mesmo tempo. Necessário
e perigoso: lock em tabela grande trava produção.

**Race condition (condição de corrida)** — bug que só aparece quando duas coisas acontecem ao
mesmo tempo, na ordem errada. Difícil de reproduzir, fácil de causar.

**Idempotência** — a operação pode ser repetida sem efeito adicional. Apagar um arquivo já
apagado é idempotente; cobrar um cartão não é. Toda rotina que pode rodar duas vezes precisa
disso.
↳ [arquitetura-e-produto.md](arquitetura-e-produto.md)

**Schema** — o desenho do banco: quais tabelas existem, com quais colunas e tipos. Em
PostgreSQL também é o nome de um "namespace" que agrupa tabelas (`vendas.pedidos`).

**Migration** — script versionado que altera a estrutura do banco. O build da aplicação
**não** roda migration: são coisas separadas, e confundir isso derruba produção.
↳ [banco-de-dados.md](banco-de-dados.md)

**DDL** — comandos que mudam a *estrutura* (`CREATE`, `ALTER`, `DROP`), em oposição aos que
mudam os *dados* (`INSERT`, `UPDATE`).

**Chave primária** — a coluna que identifica cada linha de forma única. **Chave estrangeira**
é a coluna que aponta para a chave primária de outra tabela; ela é o que impede um comentário
de existir sem dono.

**UUID** — identificador único de 128 bits (`a3f1…`), gerado sem precisar consultar o banco.
Alternativa ao id sequencial quando você não quer expor "quantos registros existem".

**Índice** — estrutura extra que o banco mantém para achar linhas rápido, como o índice de um
livro. Acelera leitura, encarece escrita.

**View** — consulta salva com nome, que se comporta como tabela mas é recalculada a cada uso.
Sempre fresca, potencialmente lenta.

**Snapshot** — cópia congelada de um momento. Uma tabela criada por `CREATE TABLE AS SELECT`
é snapshot: a origem continua mudando, ela não.
↳ [banco-de-dados.md](banco-de-dados.md)

**Tabela materializada** — o resultado de uma consulta gravado como tabela física, para leitura
rápida. É um snapshot, e por isso precisa de um plano de atualização.
↳ [banco-de-dados.md](banco-de-dados.md)

**Catálogo** — as tabelas internas onde o banco guarda informação sobre si mesmo: que tabelas
existem, que colunas, que dependências. É onde você consulta antes de apagar algo.

**JOIN** — combinar linhas de duas tabelas por uma coluna em comum. É onde caracteres
invisíveis silenciosamente impedem o casamento.
↳ [banco-de-dados.md](banco-de-dados.md)

**NULL** — ausência de valor, diferente de zero e de string vazia. `NULL = NULL` é falso, o
que surpreende quase todo mundo uma vez.

**Offset** — "pule as primeiras N linhas". Forma comum e traiçoeira de paginar: se os dados
mudam entre as páginas, você pula ou repete registros.

**Pooler** — intermediário que reaproveita poucas conexões reais de banco entre muitos
clientes. Ótimo para requisições curtas; fatal para transações longas, porque ele arranca a
conexão no meio.
↳ [banco-de-dados.md](banco-de-dados.md)

**ORM** — biblioteca que traduz objetos do seu código em linhas de tabela, para você não
escrever SQL. Conveniente; esconde consultas caras, como carregar mil linhas para contar.

**Reconciliação** — rotina que compara dois lados e conserta o que ficou pela metade. É a rede
de segurança de qualquer fluxo que passa por várias etapas de rede.
↳ [arquitetura-e-produto.md](arquitetura-e-produto.md)

**OLAP (processamento analítico)** — banco ou consulta feita para *analisar* grandes volumes
(somar, agrupar, cruzar milhões de linhas), em oposição ao **OLTP**, o transacional do dia a dia
(inserir um pedido, ler um cadastro). Bancos OLAP, como ClickHouse ou DuckDB, existem para
relatório e dashboard, não para gravar cada clique.

**Banco colunar** — banco que guarda os dados por **coluna** em vez de por linha. Para "somar o
valor de 10 milhões de vendas" ele lê só a coluna `valor`, não a linha inteira — por isso é tão
rápido em agregação, e ruim em alterar registro a registro. É a tecnologia por trás da maioria
dos bancos OLAP.

**Pré-agregação** — calcular o resultado (o total por mês, a média por rota) **antes** de a tela
pedir, gravando numa tabela pequena que é lida pronta. Troca "processar na hora da consulta" por
"processar uma vez, no ETL". É o que faz um painel responder em milissegundos sem banco especial.

---

## Deploy, build e infraestrutura

**Pipeline** — a sequência automatizada que pega seu commit e o transforma em software no ar:
instalar, testar, construir, publicar. "O pipeline está verde" quer dizer que passou.

**Runner** — a máquina que executa os passos do pipeline. Costuma ser diferente da sua e da de
produção — origem de muita surpresa.

**Artefato** — o resultado empacotado de um build (um `.tar.gz`, uma imagem, uma pasta
compilada), passado adiante entre etapas. Cuidado: costuma ser baixável por quem tem acesso ao
repositório.

**Build-time versus runtime** — *build* é quando o código vira o pacote que será publicado;
*runtime* é quando esse pacote atende requisições. Variável lida no build fica congelada no
pacote: trocá-la depois não muda nada até um novo build.
↳ [deploy-e-build.md](deploy-e-build.md)

**Staging** — ambiente que imita produção, para testar antes de valer. Só ajuda se for
parecido de verdade.

**Rollback** — voltar para a versão anterior depois de um deploy ruim. Ter rollback fácil vale
mais que ter deploy perfeito.

**Serverless** — seu código roda em processos que nascem quando chega uma requisição e morrem
logo depois. Consequências: nada guardado em memória sobrevive, e há teto de duração.
↳ [deploy-e-build.md](deploy-e-build.md)

**Cold start** — a demora da primeira requisição depois de um tempo parado, porque o ambiente
precisa ser criado do zero.

**Cache de camadas (Docker)** — cada instrução do Dockerfile vira uma camada reaproveitável.
Mudou uma camada, todas as seguintes são refeitas — por isso instalar dependências vem antes
de copiar o código.
↳ [deploy-e-build.md](deploy-e-build.md)

**Restart loop** — container que sobe, quebra, é reiniciado, quebra de novo. Durante cada
tentativa ele aparece como "rodando" — por isso checar o estado engana.
↳ [infraestrutura-e-containers.md](infraestrutura-e-containers.md)

**Módulo nativo** — biblioteca com parte escrita em C/C++, que precisa ser compilada para a
combinação exata de sistema e arquitetura. É uma das origens clássicas de "funciona na minha
máquina".
↳ [infraestrutura-e-containers.md](infraestrutura-e-containers.md)

**x86 versus ARM** — duas famílias de processador com instruções incompatíveis. Imagem
construída para uma não roda na outra — o erro é `exec format error`. **QEMU** é o emulador
que permite construir para a outra arquitetura, muito lentamente.
↳ [deploy-e-build.md](deploy-e-build.md)

**POSIX** — o conjunto de convenções que sistemas tipo Unix (Linux, macOS) compartilham.
"Shell POSIX" quer dizer bash/sh, em oposição ao PowerShell.

**Swap** — espaço em disco usado como memória de emergência. Troca "o processo morre" por "o
processo fica muito lento".
↳ [infraestrutura-e-containers.md](infraestrutura-e-containers.md)

**CDN** — rede que guarda cópias do seu conteúdo perto do usuário. Acelera arquivo estático;
**não** acelera consulta a banco nem página gerada na hora.
↳ [deploy-e-build.md](deploy-e-build.md)

**Proxy reverso** — servidor na frente da sua aplicação que recebe as requisições e repassa.
Faz TLS, cache e balanceamento — e às vezes bufferiza sua resposta sem você pedir.

**Egresso** — tráfego que *sai* da sua máquina. Muitos bloqueios são de egresso, não de
entrada — e o sintoma é idêntico ao de um serviço fora do ar.
↳ [apis-e-integracoes.md](apis-e-integracoes.md)

**Socket** — a ponta de uma conexão de rede. "Montar o socket do Docker" significa dar ao
container o canal de controle do próprio Docker — na prática, root na máquina.

**Apex e subdomínio** — `exemplo.com` é o apex (raiz); `loja.exemplo.com` é um subdomínio.
Certificado do apex **não** cobre subdomínio automaticamente.

**SMTP** — o protocolo de envio de e-mail. **SPF, DKIM e DMARC** são três registros de DNS que
provam que o e-mail veio mesmo de você: SPF autoriza quem pode enviar, DKIM assina a mensagem,
DMARC diz o que fazer quando falha.
↳ [infraestrutura-e-containers.md](infraestrutura-e-containers.md)

**Observabilidade** — a capacidade de responder "o que está acontecendo agora?" com log,
métrica e rastreamento. É o que transforma um incidente de duas horas em um de dez minutos.

**Sandbox** — ambiente isolado onde código roda sem alcançar o resto do sistema. Também é o
modo restrito em que alguns provedores colocam contas novas.

**Multi-inquilino (multi-tenant)** — uma instalação só do sistema atendendo vários clientes,
com os dados separados logicamente. Cada cliente é um "inquilino".

**Reverse proxy** — servidor que fica **na frente** das suas aplicações e encaminha cada
requisição para a certa, decidindo pelo domínio ou pelo caminho; costuma cuidar também de HTTPS
e de filtros de acesso. Traefik, nginx e Caddy fazem esse papel. "Está atrás do proxy" quer
dizer que ninguém fala direto com a aplicação — tudo passa por ele.

**Crash loop (restart loop)** — quando um container ou processo morre logo ao iniciar e a
política de reinício o sobe de novo, que morre de novo, sem parar: fica "tentando subir" sem
nunca ficar de pé. No Docker o sintoma é o status `Restarting`; a causa está nos primeiros logs,
antes da morte.

---

## Segurança

**Hash** — função que transforma qualquer entrada numa string de tamanho fixo, sem volta.
Serve para guardar senha e para comparar dois valores sem revelá-los.

**HMAC** — assinatura calculada com uma chave secreta. Prova que a mensagem não foi adulterada
e veio de quem tem a chave. Barata de verificar, impossível de forjar sem a chave.
↳ [seguranca-e-segredos.md](seguranca-e-segredos.md)

**Cifrar em repouso** — guardar o dado criptografado no banco, de modo que quem obtiver uma
cópia dele não leia nada. Decifre num ponto único do código, nunca espalhado.
↳ [seguranca-e-segredos.md](seguranca-e-segredos.md)

**Allowlist** — lista do que é permitido, bloqueando todo o resto. O contrário de blocklist
(listar o proibido), que sempre esquece um caso.

**SSRF** — ataque em que você faz o **servidor** buscar uma URL escolhida pelo atacante, e ele
usa isso para alcançar a rede interna. Toda rota que aceita URL do cliente precisa de
allowlist.
↳ [seguranca-e-segredos.md](seguranca-e-segredos.md)

**URL pré-assinada** — link temporário que autoriza enviar ou baixar um arquivo direto do
storage, sem passar pelo seu servidor. É uma credencial: quem tem o link, tem a permissão.
↳ [seguranca-e-segredos.md](seguranca-e-segredos.md)

**Inlinear** — o empacotador substitui `process.env.X` pelo **valor literal** dentro do código
enviado ao navegador. Por isso prefixo público em variável secreta vaza a chave.
↳ [seguranca-e-segredos.md](seguranca-e-segredos.md)

**Rotação** — trocar uma credencial por uma nova. A ordem importa: crie e propague a nova
antes de revogar a velha, ou você derruba tudo que ainda usa a antiga.
↳ [seguranca-e-segredos.md](seguranca-e-segredos.md)

**Certificado (TLS)** — o arquivo que prova a identidade do seu domínio e permite HTTPS.
Emitido por domínio; um novo subdomínio precisa do seu.

**Token fine-grained (granular)** — credencial cujas permissões são recortadas por recurso e por
ação (ler *este* repositório, mas não listar todos; escrever aqui, não ali), em vez de um escopo
amplo do tipo "leitura de tudo". Mais seguro se vazar, mas exige conferir **cada** operação que a
automação faz — ter permissão de ler um item não implica poder listar os itens.

---

## Automação e operação

**Falha silenciosa** — o programa falha e nada avisa: sem erro, sem log, sem alerta. É o modo
de falha padrão de job agendado, e o tema que mais se repete neste repositório.
↳ [automacao-e-agendamento.md](automacao-e-agendamento.md)

**Bufferização de saída** — o programa acumula o que imprime e só grava em blocos. Log vazio
de processo rodando não significa travado — significa que o bloco ainda não encheu.
↳ [automacao-e-agendamento.md](automacao-e-agendamento.md)

**Dry-run** — modo que mostra o que *seria* feito, sem fazer. Obrigatório em qualquer rotina
que apaga coisas.
↳ [automacao-e-agendamento.md](automacao-e-agendamento.md)

**Best-effort** — passo que pode falhar sem derrubar o resto. O perigo é ele falhar em
silêncio e ninguém perceber por meses.
↳ [automacao-e-agendamento.md](automacao-e-agendamento.md)

**Orquestrador** — o sistema que decide o que roda, quando e em que ordem. Costuma registrar
tudo em UTC, o que confunde comparação com horário local.

**Checksum** — número calculado a partir do conteúdo de um arquivo, para detectar se ele mudou
ou corrompeu. Comparar checksum é mais confiável que comparar tamanho e data.

**UTC** — o fuso de referência mundial, sem horário de verão. Servidores registram em UTC;
usuários vivem em outro fuso, e é aí que a cota "mensal" zera cedo.
↳ [frontend-e-nextjs.md](frontend-e-nextjs.md)

---

## Texto e arquivos

**Encoding** — a regra que traduz bytes em letras. UTF-8 é o padrão hoje; problema quase
sempre é alguém gravando em um e lendo em outro. **UTF-16LE** e **cp1252** são outros
encodings, e o Windows ainda os usa por padrão em alguns comandos.

**Mojibake** — texto UTF-8 lido com o encoding errado e regravado: "produção" vira
"produÃ§Ã£o". O arquivo cresce sem ganhar conteúdo, e é reversível.
↳ [encoding-e-midia.md](encoding-e-midia.md)

**BOM** — três bytes invisíveis no começo do arquivo indicando o encoding. Inofensivo em
editor, fatal quando outra ferramenta lê o arquivo como valor puro.
↳ [windows-e-powershell.md](windows-e-powershell.md)

**`U+FFFD`** — o caractere "�" que aparece quando a decodificação falhou. É prova de perda:
se ele aparece depois de você consertar um encoding, o dado original não voltou inteiro.

**NBSP** — espaço "não separável" (`\xa0`), visualmente idêntico ao espaço comum mas um
caractere diferente. Vem de HTML e planilha, e faz JOIN falhar sem motivo aparente.
↳ [banco-de-dados.md](banco-de-dados.md)

**Com perda / sem perda** — compressão com perda descarta informação para reduzir tamanho
(JPEG); sem perda preserva tudo. Recomprimir um arquivo já comprimido costuma **aumentá-lo**.
↳ [encoding-e-midia.md](encoding-e-midia.md)

**Greedy / não-greedy** — em expressões regulares, `.*` pega o máximo possível e `.*?` o
mínimo. Escolher errado trunca dados aninhados.
↳ [encoding-e-midia.md](encoding-e-midia.md)

**NTFS / APFS / ext4** — sistemas de arquivos do Windows, do macOS e do Linux. Os dois
primeiros não diferenciam maiúsculas de minúsculas; o terceiro sim — e é por isso que um
arquivo "some" no CI.
↳ [git.md](git.md)

---

## Git

**Upstream** — o branch remoto que o seu branch local acompanha. Fica gravado na configuração,
**não** é deduzido pelo nome — por isso `git status` pode comparar com o branch errado e
inventar commits.
↳ [git.md](git.md)

**Bundle** — arquivo único com o histórico completo de um repositório, restaurável com
`git clone arquivo.bundle`. É a forma certa de fazer backup de repositório.
↳ [git.md](git.md)

**Packfile** — o formato comprimido em que o Git guarda os objetos. Já vem comprimido, e é por
isso que zipar um repositório não economiza quase nada.

---

## IA

**LLM** — modelo de linguagem grande, o tipo de modelo por trás dos assistentes de texto e
código.

**Janela de contexto** — quanto texto o modelo consegue considerar de uma vez. Varia em ordem
de grandeza entre modelos do mesmo fornecedor, e estourar trunca sem avisar.
↳ [ferramentas-de-ia.md](ferramentas-de-ia.md)

**Modelo de raciocínio** — modelo que "pensa" antes de responder. Compensa em decisão difícil;
é escolha ruim para reescrever texto longo, porque paga um custo fixo alto.
↳ [ferramentas-de-ia.md](ferramentas-de-ia.md)

**Saída estruturada** — pedir ao modelo que responda em JSON com formato fixo. Continua sendo
**entrada não confiável**: todo identificador que ele devolver precisa ser conferido.
↳ [ferramentas-de-ia.md](ferramentas-de-ia.md)
