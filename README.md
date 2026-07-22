# Aprendizados técnicos

Armadilhas, causas-raiz e regras acumuladas construindo e operando software. Cobre
infraestrutura, backend, frontend, dados, IA e o que mais vier — o escopo é "qualquer coisa
técnica que custou tempo para descobrir e vale não redescobrir".

## 👉 [Comece aqui](COMECE-AQUI.md)

**Para quem isto foi escrito:** quem está na faculdade, estagiando, no primeiro emprego, ou
aprendendo por conta. A maior parte destes erros não está em tutorial nenhum — eles aparecem
só quando o código sai da sua máquina e encontra usuários, produção e outras pessoas.

Não leia na ordem, e não leia tudo. É uma **referência** para consultar quando você já está
com um problema na tela. O [Comece aqui](COMECE-AQUI.md) organiza as lições por **o que você
está fazendo** — "funciona na minha máquina e quebra quando eu subo", "botei no ar e agora
tem gente usando", "vou mexer com senha e chave de API" — em vez de por categoria técnica.

Se esbarrar num termo desconhecido, o [glossário](GLOSSARIO.md) explica em uma linha.

Cada lição segue o mesmo formato:

> **Sintoma** — como o problema aparece para quem está debugando
> **Causa** — por que acontece
> **Regra** — o que fazer
> **Como verificar** — o comando ou a checagem concreta

O formato é proposital. Quase toda lição aqui nasceu de horas gastas com um sintoma
enganoso — algo que parecia ser rede e era configuração, parecia ser bug do framework e
era cache do navegador, parecia estar funcionando e estava falhando em silêncio. Procurar
pelo **sintoma** é como isto se usa às 2 da manhã.

## Índice

**[Comece aqui](COMECE-AQUI.md)** — porta de entrada, organizada por situação. Se você
está começando, é por aqui.

**[Índice por sintoma](INDICE-POR-SINTOMA.md)** — 208 sintomas em ordem alfabética, cada um
linkado para a lição. Para quando você já sabe descrever o que está vendo.

**[Glossário](GLOSSARIO.md)** — os termos usados nas lições, em uma linha cada.

**Fundamentos e operação**

| Arquivo | Sobre |
|---|---|
| [seguranca-e-segredos.md](seguranca-e-segredos.md) | Credenciais, autenticação, autorização, o que vaza para o navegador |
| [infraestrutura-e-containers.md](infraestrutura-e-containers.md) | Docker, DNS, e-mail, rede, servidores |
| [automacao-e-agendamento.md](automacao-e-agendamento.md) | Jobs agendados, ETL, falha silenciosa, diagnóstico |
| [git.md](git.md) | `.gitignore`, backup de repositório, branches, histórico |
| [windows-e-powershell.md](windows-e-powershell.md) | Diferenças que quebram scripts vindos do mundo Unix |

**Aplicação**

| Arquivo | Sobre |
|---|---|
| [apis-e-integracoes.md](apis-e-integracoes.md) | Webhooks, pagamentos, provedores externos, paginação, reconciliação |
| [frontend-e-nextjs.md](frontend-e-nextjs.md) | React, Next.js, imagens responsivas, CSS, APIs do navegador |
| [deploy-e-build.md](deploy-e-build.md) | Build, variáveis de ambiente, regiões, limites de plataforma |
| [banco-de-dados.md](banco-de-dados.md) | Postgres, ORM, concorrência, migrações, modelagem |
| [encoding-e-midia.md](encoding-e-midia.md) | UTF-8, mojibake, compressão de imagem, formatos |

**Transversal**

| Arquivo | Sobre |
|---|---|
| [ferramentas-de-ia.md](ferramentas-de-ia.md) | LLMs em produção, agentes de código, custo e latência |
| [arquitetura-e-produto.md](arquitetura-e-produto.md) | Decisões com trade-off, modelagem de estado, defaults perigosos |

## Três coisas que se repetem

Depois de organizar tudo isto, três padrões apareceram em praticamente toda categoria.
Se você levar só isto daqui, já valeu:

**1. Falha silenciosa é o modo de falha padrão.**
Paginação que para na primeira página. Remoção de diretório que não remove. Job agendado
que parou semanas atrás. Webhook que aceita qualquer coisa. Nenhum deles emite erro. A
lição operacional: *ausência de erro não é evidência de sucesso*. Verifique o efeito — o
arquivo sumiu? a data avançou? o objeto apareceu no destino? — nunca o código de saída.

**2. O sintoma quase nunca aponta para a causa.**
Timeout de TCP que parecia firewall e era porta errada na configuração. Deploy bloqueado
que parecia build quebrado e era o e-mail do autor do commit. Imagem que a tag `<img>`
exibe e o `fetch` não baixa, por causa de uma resposta opaca no cache. A regra: antes de
acusar a camada mais distante e mais cara de investigar — rede, provedor, infraestrutura —
prove que o que está perto está certo.

**3. Confiança precisa de fronteira explícita.**
Saída de LLM é entrada não confiável. Total calculado no cliente é sugestão. O primeiro IP
de `X-Forwarded-For` é fornecido pelo atacante. Esconder um botão não é controle de acesso.
Toda vez que um dado atravessa uma fronteira, alguém precisa decidir conscientemente se ele
é fato ou alegação.

## Como isto cresce

Área nova vira arquivo novo, com uma linha no índice acima. O formato das lições não muda —
é ele que torna o conteúdo pesquisável e comparável entre domínios muito diferentes.

Áreas ainda não cobertas, que devem entrar conforme aparecerem: engenharia de dados e
orquestração, machine learning (treino, avaliação, deriva de modelo), observabilidade e
testes.

Uma lição só entra aqui se passar em dois testes: **custou tempo real para descobrir** e
**vale para alguém que não conhece o contexto onde ela apareceu**. Trivialidade e
preferência de estilo ficam de fora.

## Sobre a origem

Extraído de anotações técnicas, documentação de projeto e histórico de sessões acumulados ao
longo de aproximadamente um ano.

Nomes de empresa, cliente, produto, servidor, endereço e credencial foram removidos, e o
contexto foi reduzido ao tipo de sistema — nunca a qual sistema. As lições estão separadas
da história que as gerou de propósito: o valor está na armadilha, não em quem caiu nela.

Dito isso, este repositório é assinado e público, então vale ser explícito: **ele foi
escrito assumindo que colegas, clientes e empregadores podem lê-lo.** Não é um documento
anônimo e não pretende ser. Cada lição passou pelo teste de "eu diria isto numa conversa
técnica com a pessoa envolvida presente" — e o que não passou ficou de fora, em vez de ser
disfarçado.
