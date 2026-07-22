# Lições técnicas

Armadilhas, causas-raiz e regras acumuladas construindo e operando aplicações web,
pipelines de dados e automação — em Windows, Linux e plataformas serverless.

Cada lição segue o mesmo formato:

> **Sintoma** — como o problema aparece para quem está debugando
> **Causa** — por que acontece
> **Regra** — o que fazer
> **Como verificar** — o comando ou a checagem concreta

O formato é proposital. Quase toda lição aqui nasceu de horas gastas com um sintoma
enganoso — algo que parecia ser rede e era configuração, parecia ser bug do framework
e era cache do navegador, parecia estar funcionando e estava falhando em silêncio.
Procurar pelo **sintoma** é como você vai usar isto às 2 da manhã.

## Índice

| Arquivo | Sobre |
|---|---|
| [seguranca-e-segredos.md](seguranca-e-segredos.md) | Credenciais, autenticação, autorização, o que vaza para o navegador |
| [apis-e-integracoes.md](apis-e-integracoes.md) | Webhooks, pagamentos, provedores externos, paginação, reconciliação |
| [frontend-e-nextjs.md](frontend-e-nextjs.md) | React, Next.js, imagens responsivas, CSS, APIs do navegador |
| [deploy-e-build.md](deploy-e-build.md) | Build, variáveis de ambiente, regiões, limites de plataforma |
| [banco-de-dados.md](banco-de-dados.md) | Postgres, ORM, concorrência, migrações, modelagem |
| [infraestrutura-e-containers.md](infraestrutura-e-containers.md) | Docker, DNS, e-mail, rede, servidores |
| [automacao-e-agendamento.md](automacao-e-agendamento.md) | Jobs agendados, ETL, falha silenciosa, diagnóstico |
| [git.md](git.md) | `.gitignore`, backup de repositório, branches, histórico |
| [windows-e-powershell.md](windows-e-powershell.md) | Diferenças que quebram scripts vindos do mundo Unix |
| [encoding-e-midia.md](encoding-e-midia.md) | UTF-8, mojibake, compressão de imagem, formatos |
| [ferramentas-de-ia.md](ferramentas-de-ia.md) | LLMs em produção, agentes de código, custo e latência |
| [arquitetura-e-produto.md](arquitetura-e-produto.md) | Decisões com trade-off, modelagem de estado, defaults perigosos |

## Três coisas que se repetem

Depois de organizar tudo isto, três padrões apareceram em praticamente toda categoria.
Se você levar só isto daqui, já valeu:

**1. Falha silenciosa é o modo de falha padrão.**
Paginação que para na primeira página. `rmtree` que não apaga. Job agendado que
parou semanas atrás. Webhook que aceita qualquer coisa. Nenhum deles emite erro.
A lição operacional: *ausência de erro não é evidência de sucesso*. Verifique o
efeito — o arquivo sumiu? a data avançou? o objeto apareceu no destino? — nunca o
código de saída.

**2. O sintoma quase nunca aponta para a causa.**
Timeout de TCP que parecia firewall e era porta errada. Deploy bloqueado que parecia
build quebrado e era o e-mail do autor do commit. Imagem que a tag `<img>` exibe e o
`fetch` não baixa, por causa de uma resposta opaca no cache. A regra: antes de acusar
a camada mais distante e mais cara de investigar (rede, provedor, infraestrutura),
prove que o que está perto está certo.

**3. Confiança precisa de fronteira explícita.**
Saída de LLM é entrada não confiável. Total calculado no cliente é sugestão.
O primeiro IP de `X-Forwarded-For` é fornecido pelo atacante. Esconder um botão
não é controle de acesso. Toda vez que um dado atravessa uma fronteira, alguém
precisa decidir conscientemente se ele é fato ou alegação.

## Como isto foi produzido

Extraído de anotações de trabalho, documentação de projeto e histórico de sessões
acumulados ao longo de aproximadamente um ano, e reescrito de forma genérica.

Nada aqui identifica empresa, cliente, servidor ou sistema. As lições foram separadas
da história que as gerou de propósito: o valor está na armadilha, não em quem caiu nela.
Onde o contexto era indispensável para a lição fazer sentido, ele foi substituído por
uma descrição neutra ("um ERP de transporte", "um portal de associados").

Aprendizados que não sobreviveram à anonimização — regra de negócio proprietária,
topologia de infraestrutura, precificação — foram descartados em vez de disfarçados.
