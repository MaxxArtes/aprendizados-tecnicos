# Glossário

Os termos que aparecem nas lições, em uma linha cada. Não é para decorar — é para desbloquear
a leitura quando você esbarra numa palavra que todo mundo usa como se você já soubesse.

Cada verbete leva para a lição onde o conceito aparece com contexto completo.

---

## Web e navegador

**CORS** — regra do **navegador** que impede um site de ler dados de outro domínio sem
permissão explícita. Exibir uma imagem de outro domínio é permitido; *ler os bytes* dela
exige CORS. Por isso um teste com `curl` nunca reproduz erro de CORS: `curl` não é navegador.
↳ [apis-e-integracoes.md](apis-e-integracoes.md)

**Preflight** — antes de certas requisições cross-origin, o navegador manda sozinho um
`OPTIONS` perguntando "posso?". Se a resposta não autorizar, a requisição real nem sai.
↳ [apis-e-integracoes.md](apis-e-integracoes.md)

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

---

## Servidor, deploy e infraestrutura

**Build-time versus runtime** — *build* é quando o código vira o pacote que será publicado;
*runtime* é quando esse pacote atende requisições. Variável lida no build fica congelada no
pacote: trocá-la depois não muda nada até um novo build.
↳ [deploy-e-build.md](deploy-e-build.md)

**Serverless** — seu código roda em processos que nascem quando chega uma requisição e morrem
logo depois. Consequências: nada guardado em memória sobrevive, e há teto de duração.
↳ [deploy-e-build.md](deploy-e-build.md)

**Cold start** — a demora da primeira requisição depois de um tempo parado, porque o ambiente
precisa ser criado do zero.
↳ [ferramentas-de-ia.md](ferramentas-de-ia.md)

**Pooler** — intermediário que reaproveita poucas conexões reais de banco entre muitos
clientes. Ótimo para requisições curtas; fatal para transações longas, porque ele arranca a
conexão no meio.
↳ [banco-de-dados.md](banco-de-dados.md)

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
construída para uma não roda na outra — o erro é `exec format error`.
↳ [deploy-e-build.md](deploy-e-build.md)

**Swap** — espaço em disco usado como memória de emergência. Troca "o processo morre" por "o
processo fica muito lento".
↳ [infraestrutura-e-containers.md](infraestrutura-e-containers.md)

**CDN** — rede que guarda cópias do seu conteúdo perto do usuário. Acelera arquivo estático;
**não** acelera consulta a banco nem página gerada na hora.
↳ [deploy-e-build.md](deploy-e-build.md)

**Egresso** — tráfego que *sai* da sua máquina. Muitos bloqueios são de egresso, não de
entrada — e o sintoma é idêntico ao de um serviço fora do ar.
↳ [apis-e-integracoes.md](apis-e-integracoes.md)

---

## Dados e concorrência

**Atômico** — operação que acontece inteira ou não acontece; ninguém observa o meio dela. Ler,
calcular e escrever em três passos **não** é atômico, e é aí que dois clientes compram o mesmo
último item.
↳ [banco-de-dados.md](banco-de-dados.md)

**Idempotência** — a operação pode ser repetida sem efeito adicional. Apagar um arquivo já
apagado é idempotente; cobrar um cartão não é. Toda rotina que pode rodar duas vezes precisa
disso.
↳ [arquitetura-e-produto.md](arquitetura-e-produto.md)

**Migration** — script versionado que altera a estrutura do banco. O build da aplicação
**não** roda migration: são coisas separadas, e confundir isso derruba produção.
↳ [banco-de-dados.md](banco-de-dados.md)

**DDL** — comandos que mudam a *estrutura* (`CREATE`, `ALTER`, `DROP`), em oposição aos que
mudam os *dados* (`INSERT`, `UPDATE`).
↳ [banco-de-dados.md](banco-de-dados.md)

**Snapshot** — cópia congelada de um momento. Uma tabela criada por `CREATE TABLE AS SELECT`
é snapshot: a origem continua mudando, ela não.
↳ [banco-de-dados.md](banco-de-dados.md)

**Reconciliação** — rotina que compara dois lados e conserta o que ficou pela metade. É a rede
de segurança de qualquer fluxo que passa por várias etapas de rede.
↳ [arquitetura-e-produto.md](arquitetura-e-produto.md)

---

## Segurança

**Cifrar em repouso** — guardar o dado criptografado no banco, de modo que quem obtiver uma
cópia dele não leia nada. Decifre num ponto único do código, nunca espalhado.
↳ [seguranca-e-segredos.md](seguranca-e-segredos.md)

**HMAC** — assinatura calculada com uma chave secreta. Prova que a mensagem não foi adulterada
e veio de quem tem a chave. Barata de verificar, impossível de forjar sem a chave.
↳ [seguranca-e-segredos.md](seguranca-e-segredos.md)

**SSRF** — ataque em que você faz o **servidor** buscar uma URL escolhida pelo atacante, e ele
usa isso para alcançar a rede interna. Toda rota que aceita URL do cliente precisa de
allowlist.
↳ [seguranca-e-segredos.md](seguranca-e-segredos.md)

**URL pré-assinada** — link temporário que autoriza enviar ou baixar um arquivo direto do
storage, sem passar pelo seu servidor. É uma credencial: quem tem o link, tem a permissão.
↳ [seguranca-e-segredos.md](seguranca-e-segredos.md)

**Falha aberta / falha fechada** — quando um mecanismo de proteção quebra, ele libera (aberta)
ou bloqueia (fechada)? Proteção contra abuso costuma falhar aberta; controle de gasto,
fechada.
↳ [apis-e-integracoes.md](apis-e-integracoes.md)

**Inlinear** — o empacotador substitui `process.env.X` pelo **valor literal** dentro do código
enviado ao navegador. Por isso prefixo público em variável secreta vaza a chave.
↳ [seguranca-e-segredos.md](seguranca-e-segredos.md)

**Webhook** — o serviço de terceiro chama **você** quando algo acontece, em vez de você ficar
perguntando. Como qualquer um pode chamar essa URL, ela precisa validar quem chamou.
↳ [apis-e-integracoes.md](apis-e-integracoes.md)

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

**Backoff exponencial** — ao receber erro, espere antes de tentar de novo, dobrando a espera a
cada tentativa. Evita que sua repetição piore a sobrecarga que causou o erro.
↳ [ferramentas-de-ia.md](ferramentas-de-ia.md)

**Rate limit** — teto de quantas requisições você pode fazer num período. Costuma ter duas
dimensões independentes: número de chamadas **e** volume de dados.
↳ [ferramentas-de-ia.md](ferramentas-de-ia.md)

---

## Texto e arquivos

**Encoding** — a regra que traduz bytes em letras. UTF-8 é o padrão hoje; problema quase
sempre é alguém gravando em um e lendo em outro.
↳ [encoding-e-midia.md](encoding-e-midia.md)

**Mojibake** — texto UTF-8 lido com o encoding errado e regravado: "produção" vira
"produÃ§Ã£o". O arquivo cresce sem ganhar conteúdo, e é reversível.
↳ [encoding-e-midia.md](encoding-e-midia.md)

**BOM** — três bytes invisíveis no começo do arquivo indicando o encoding. Inofensivo em
editor, fatal quando outra ferramenta lê o arquivo como valor puro.
↳ [windows-e-powershell.md](windows-e-powershell.md)

**Com perda / sem perda** — compressão com perda descarta informação para reduzir tamanho
(JPEG); sem perda preserva tudo. Recomprimir um arquivo já comprimido costuma **aumentá-lo**.
↳ [encoding-e-midia.md](encoding-e-midia.md)

**Greedy / não-greedy** — em expressões regulares, `.*` pega o máximo possível e `.*?` o
mínimo. Escolher errado trunca dados aninhados.
↳ [encoding-e-midia.md](encoding-e-midia.md)

---

## Git

**Upstream** — o branch remoto que o seu branch local acompanha. Fica gravado na configuração,
**não** é deduzido pelo nome — por isso `git status` pode comparar com o branch errado e
inventar commits.
↳ [git.md](git.md)

**Bundle** — arquivo único com o histórico completo de um repositório, restaurável com
`git clone arquivo.bundle`. É a forma certa de fazer backup de repositório.
↳ [git.md](git.md)

---

## IA

**Janela de contexto** — quanto texto o modelo consegue considerar de uma vez. Varia em ordem
de grandeza entre modelos do mesmo fornecedor, e estourar trunca sem avisar.
↳ [ferramentas-de-ia.md](ferramentas-de-ia.md)

**Modelo de raciocínio** — modelo que "pensa" antes de responder. Compensa em decisão difícil;
é escolha ruim para reescrever texto longo, porque paga um custo fixo alto.
↳ [ferramentas-de-ia.md](ferramentas-de-ia.md)

**Saída estruturada** — pedir ao modelo que responda em JSON com formato fixo. Continua sendo
**entrada não confiável**: todo identificador que ele devolver precisa ser conferido.
↳ [ferramentas-de-ia.md](ferramentas-de-ia.md)
