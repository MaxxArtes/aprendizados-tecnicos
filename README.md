# Aprendizados técnicos

**Este repositório reúne o que eu aprendi construindo e operando software, e está aberto para
qualquer pessoa, em qualquer nível.**

São os erros que cometi, as causas que levei horas para encontrar e as regras que ficaram no
lugar. Publiquei porque quase nada disto aparece em tutorial: são problemas que só surgem
quando o código sai da sua máquina e encontra produção, usuários e outras pessoas — e quando
surgem, normalmente você está sozinho, com pressa, colando uma mensagem de erro na busca.

Se algo aqui poupar a madrugada de alguém, já valeu.

Cobre infraestrutura, backend, frontend, dados, IA e o que mais vier. O escopo é simples:
qualquer coisa técnica que custou tempo para descobrir e vale não redescobrir.

## Por onde entrar

**Está começando** — faculdade, estágio, primeiro emprego, aprendendo por conta?
👉 **[Comece aqui](COMECE-AQUI.md)**, que organiza as lições por situação ("funciona na minha
máquina e quebra quando eu subo", "botei no ar e agora tem gente usando").

**Já trabalha com isso?** O [índice por sintoma](INDICE-POR-SINTOMA.md) costuma ser o caminho
mais curto: procure pelo que está vendo na tela.

**Esbarrou num termo que não conhece?** O [glossário](GLOSSARIO.md) tem 122 verbetes. Os
termos técnicos foram mantidos como são usados no mercado, de propósito — a ideia é que
ninguém fique de fora por causa de vocabulário, não que o vocabulário desapareça.

Não leia na ordem e não leia tudo. É uma referência, não um curso.

---

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

| Guia | |
|---|---|
| [Comece aqui](COMECE-AQUI.md) | porta de entrada, organizada por situação |
| [Índice por sintoma](INDICE-POR-SINTOMA.md) | 208 sintomas, para quando você já sabe nomear o problema |
| [Glossário](GLOSSARIO.md) | 111 termos, em uma linha cada |

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
| [treino-de-modelos.md](treino-de-modelos.md) | Pré-treino do zero, avaliação honesta, quantização e a infra em volta |
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
