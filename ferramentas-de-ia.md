# LLMs e ferramentas de IA em produção

## Valide contra o contexto todo ID que o modelo devolver

**Sintoma:** o conteúdo gerado referencia um produto, cupom ou registro que não existe.

**Causa:** o modelo recebeu o catálogo como texto e devolveu identificadores plausíveis mas
inventados.

**Regra:** a saída estruturada do modelo é **entrada não confiável**. Cada ID retornado é
conferido contra o conjunto que foi enviado no prompt; o que não casa vira `null` com
fallback, nunca vai para o banco. Serializar o contexto de forma compacta e explícita
melhora muito a taxa de acerto.

---

## Retry com backoff não é opcional — 429 e 503 são rotina

**Causa:** endpoints de modelo generativo rejeitam por capacidade com frequência muito
maior que APIs comuns.

**Regra:** envolva toda chamada num wrapper com ~3 tentativas e backoff exponencial para
429, 5xx e erro de rede; nunca faça retry de 4xx de validação. Tenha um caminho de
degradação quando as tentativas esgotam.

**Ponto crítico de implementação:** trate `status == 429` **antes** de
`raise_for_status()`, senão vira exceção e pula a espera.

---

## Não chame o mesmo provedor duas vezes no mesmo pipeline

**Sintoma:** a segunda etapa falha com 429 de forma intermitente.

**Causa:** limite de taxa por conta, consumido pela primeira etapa.

**Regra:** distribua etapas entre provedores diferentes e monte uma cadeia de fallback
explícita. Verifique os identificadores de modelo contra o endpoint de listagem do provedor
antes de fixá-los no código — IDs mudam e somem.

---

## Cota tem duas dimensões independentes e contexto varia por modelo

**Sintoma:** erro de limite mesmo com poucas chamadas, ou truncamento inesperado do prompt
ao trocar de modelo dentro do mesmo provedor.

**Causa:** o limite é aplicado separadamente a **requisições** e a **tokens** (por minuto,
hora e dia) — estourar qualquer um bloqueia. E a janela de contexto pode variar em ordem de
grandeza entre modelos do mesmo catálogo.

**Regra:** dimensione o batch pelo limite de tokens por minuto, não pelo de requisições.
Implemente backoff lendo os headers de rate limit e tenha um modelo alternativo mapeado por
caso de uso.

---

## Corrida entre dois geradores vence fallback sequencial

**Sintoma:** encadear provedores em sequência com timeout generoso desperdiça minutos toda
vez que o primeiro trava — e o primeiro trava com frequência.

**Regra:** dispare os dois em paralelo, consuma com "primeiro que devolver resultado
**válido** vence" (validação de conteúdo, não apenas código de saída zero), e cancele as
tarefas restantes no `finally`. Custa o dobro de chamadas no pior caso e economiza minutos
em cada execução. Use quando o custo por chamada é baixo e a latência é o gargalo.

---

## O tempo de geração é dominado pelo **modelo**, não pelo volume de saída

**Sintoma:** você "enxuga" o prompt para gerar menos texto e a latência quase não muda.

**Causa:** medições reais — a mesma saída levou cerca de cinco minutos num modelo grande e
cerca de um minuto num pequeno; e uma saída quatro vezes menor no mesmo modelo não foi mais
rápida. Cold start e latência do provedor dominam.

**Regra:** para ganhar velocidade, troque o modelo ou o caminho, não o tamanho da saída.
Reduzir volume ainda vale — mas pelo peso entregue ao usuário, não pelo tempo de geração.

**Como verificar:** instrumente tempo **por fase** do pipeline. Sem isso, a atribuição de
culpa é chute.

---

## Modelo de raciocínio é a escolha errada para reescrita longa

**Causa:** modelos com cadeia de raciocínio pagam um custo fixo alto antes de emitir saída
longa.

**Regra:** para transformar ou reescrever texto longo, use a variante sem raciocínio; ou
substitua a etapa por validação heurística determinística. Raciocínio compensa em decisão,
não em transcrição.

---

## "Template determinístico + LLM preenchendo só JSON" ganha de "LLM gera tudo"

**Sintoma:** geração de documento leva minutos e a qualidade oscila a cada execução.

**Causa:** pedir ao modelo que produza o artefato final coloca o custo e a variância no
caminho crítico.

**Regra:** separe layout (código fixo, sem IA) de conteúdo (o modelo devolve só um JSON
estruturado pequeno). O render vira instantâneo e consistente. Reserve a geração livre para
os casos que não cabem em template.

---

## Um loop de typecheck antes do build caro derruba a taxa de falha

**Sintoma:** boa parte dos builds de código gerado por IA falha, cada um custando minutos.

**Regra:** rode a checagem de tipos num container leve com dependências pré-instaladas,
monte o código como volume somente-leitura, filtre apenas erros de sintaxe e devolva ao
modelo para correção, com teto de iterações. Medido: a taxa de falha caiu de mais da metade
dos builds para uma fração pequena, e o tempo total quase pela metade.

---

## Caminho especializado sem fallback derruba a funcionalidade inteira

**Sintoma:** o chat quebra por completo quando o usuário cola um link.

**Causa:** a presença de URL desviava para um caminho exclusivo de modelos de visão. Sem
nenhum deles disponível, não havia rota de texto puro.

**Regra:** todo caminho especializado precisa de degradação para o caminho genérico. E toda
chamada auxiliar de rede precisa de timeout curto com cancelamento, senão trava a função
inteira.

---

## Regex de detecção estreita demais faz a feature nunca rodar

**Sintoma:** a métrica de uso do recurso fica em zero e você conclui que "ninguém quer
isso".

**Causa:** o gatilho exigia uma palavra-chave **mais** uma palavra de contexto; as frases
naturais dos usuários não casavam.

**Regra:** detecte por padrão amplo e monitore a taxa de acionamento. Zero uso de uma
feature entregue é sintoma de bug de roteamento até prova em contrário.

---

## Quando o modelo escolhe a ferramenta errada, ponha rede de segurança no cliente

**Sintoma:** pedidos claramente de edição ("muda a cor") fazem o sistema recriar o artefato
do zero.

**Causa:** o modelo, mesmo com regra prioritária no prompt, às vezes emite a ferramenta
errada.

**Regra:** camada dupla — (1) regra prioritária no prompt quando o estado indica "já existe
artefato"; (2) verificação determinística no cliente que sobrepõe a decisão do modelo.
Prompt é probabilístico; estado é fato.

---

## Agente que narra sucesso não substitui verificação

**Sintoma:** sequência de "está 100% funcionando agora" seguida imediatamente do mesmo erro
em produção, várias vezes seguidas.

**Causa:** o agente declara conclusão ao terminar de escrever o código ou disparar o deploy,
sem exercitar o caminho real. E "espera" por tempo em vez de consultar o estado do build.

**Regra:** fechamento de tarefa só com evidência — código HTTP da rota real, linha do log,
ou contagem no banco. Consulte o status pelo comando da plataforma em vez de dormir e
presumir. **O mesmo vale para você: deploy verde ≠ feature funcionando.**

---

## Layout, contraste e ícone são o que build e linter **nunca** reprovam

**Sintoma:** build verde, revisão aprovada, e a tela sai ilegível ou quebrada em produção.

**Causa:** nenhuma ferramenta estática avalia render. Ícone desenhado à mão, contraste em
modo escuro e alvo de toque só existem em pixels.

**Regra:** para qualquer mudança visual, adicione captura de tela como etapa de
verificação. E **meça, não olhe**: overflow por `scrollWidth > clientWidth`, alvos por
`getBoundingClientRect`, tamanho por `font-size` computado, contraste calculado.

---

## `--headless --screenshot` **não** emula mobile

**Sintoma:** a captura "de celular" mostra layout desktop cortado — falso positivo de
layout quebrado.

**Causa:** a flag só define o tamanho da janela; não muda user agent, densidade de pixels
nem flag de dispositivo móvel.

**Regra:** use o protocolo de depuração com override de métricas de dispositivo
(`mobile: true`, `deviceScaleFactor`) antes de capturar.

---

## Mudar só o hash da URL não recarrega — o estado vaza entre passadas de teste

**Sintoma:** uma captura de celular aparece com um estado de UI que só foi ativado na
passada de desktop.

**Regra:** em automação de captura, force reload usando querystring diferente a cada
passada, ou recarregue com cache desativado.

---

## Clique programático e clique por coordenada mentem de formas diferentes

**Sintoma:** o teste "clica" e nada acontece; ou clica no elemento errado depois de uma
correção de layout.

**Causa:** clique programático num filho não dispara o handler registrado no pai por
delegação; clique por coordenada erra quando o layout deslocou.

**Regra:** para validar handler, dispare o evento de mouse real via protocolo no elemento
certo; para elementos estáveis, prefira clique programático. Detalhe: `dispatchEvent(new
Event('blur'))` **não** aciona `onBlur` do React — ele escuta `focusout`.

---

## Servidor de desenvolvimento pode não hidratar por causa do websocket de HMR

**Sintoma:** uma rota fica presa em "Carregando…", o efeito de autenticação nunca roda, e a
rota vizinha hidrata normalmente — parece bug da página.

**Regra:** para auditoria de UI, rode o build de produção localmente. A hidratação fica
determinística.

---

## Meça tráfego cross-origin pelos eventos do protocolo

**Sintoma:** sua auditoria diz que a página baixa 0 byte de imagem.

**Causa:** `encodedBodySize` é **0** em recursos cross-origin sem `Timing-Allow-Origin`; e
um service worker faz a requisição perder a classificação de tipo.

**Regra:** meça pelos eventos de rede do protocolo de depuração e filtre por URL, não por
tipo. Limpe qualquer cache de aplicação antes de medir, senão você mede o estado antigo.

---

## Instrução de agente centralizada com import evita cópia divergente

**Sintoma:** o agente A obedece uma regra que o agente B ignora; ninguém entende por quê.

**Causa:** cada ferramenta lê um arquivo de instrução diferente na raiz do projeto.

**Regra:** um único arquivo de contexto como fonte da verdade; o arquivo esperado por cada
ferramenta contém apenas um import mais o específico daquele projeto. Para ferramentas sem
suporte a import, symlink ou script de sincronização — **nunca cópia manual**, e se a cópia
for inevitável, trate-a como artefato gerado que nunca se edita à mão.

**Como verificar:** compare os hashes dos arquivos que deveriam ser idênticos. E audite a
cadeia de imports de **cada** projeto: import que não resolve não gera erro.

---

## Substituição por texto falha em arquivo com blocos duplicados

**Sintoma:** a ferramenta de edição recusa a operação por âncora ambígua — ou pior, edita a
ocorrência errada.

**Causa:** o arquivo acumulou conteúdo duplicado, comum em arquivos que vários agentes
editam por append.

**Regra:** deduplique antes de editar; se estiver muito duplicado, reescreva inteiro.
Prefira âncoras longas e únicas.

---

## Memória de agente não sincroniza entre máquinas

**Sintoma:** o agente na máquina A conhece a decisão que o agente na máquina B registrou
ontem — exceto que não conhece.

**Regra:** ao registrar algo que vale nos dois ambientes, avise onde foi gravado e
replique. Se virar incômodo, coloque o diretório de contexto num repositório privado
compartilhado. O mesmo vale para instruções globais: uma sessão em servidor começa **cega**
às suas regras.

---

## Ação bloqueada por política de segurança: prepare, não contorne

**Sintoma:** o agente é impedido de criar um serviço do sistema, escrever em configuração
global de SSH ou ler um arquivo de códigos de recuperação.

**Causa:** são ações de persistência ou de segredo que qualquer camada de proteção razoável
bloqueia — e o bloqueio geralmente está certo.

**Regra:** não procure o desvio. Deixe o artefato pronto com instruções de uma linha para o
humano executar. Prefira a alternativa de escopo local que **não** é bloqueada.

---

## Ao pedir aprovação, diga se a ação é reversível

**Sintoma:** aprovações concedidas no automático autorizam algo destrutivo.

**Causa:** o texto do pedido descreve o comando, não a consequência. Num contexto de
aprovação rápida, "sim" é o default.

**Regra:** formule destacando a **reversibilidade** — "isto apaga X permanentemente" versus
"isto cria Y e pode ser desfeito com Z".

---

## Contra risco de acesso amplo, a resposta é reversibilidade — não restrição

**Sintoma:** você propõe rodar a automação com usuário restrito e escopo mínimo, e o dono
da infraestrutura recusa porque isso derrota o propósito da ferramenta.

**Causa:** restringir é a mitigação correta para um usuário a ser contido; para o operador
que é dono de tudo, ela só transfere o trabalho de volta para ele.

**Regra:** quando o acesso amplo é o requisito, mitigue com **recuperabilidade** —
snapshot automático do servidor (a salvaguarda de maior valor: preserva toda a liberdade e
torna o erro reversível), contexto escrito na máquina dizendo quais credenciais ali
alcançam produção, e aprovação manual para ações irreversíveis.

---

## Não troque CLI com assinatura por chave de API sem contar o custo

**Sintoma:** a conta de uso dispara depois de "resolver" um problema de autenticação
configurando uma chave de API.

**Causa:** a assinatura já cobre o uso da ferramenta; a chave de API é um canal de cobrança
**separado**, por token.

**Regra:** quando a autenticação por assinatura falhar, ataque a causa (renovação de
sessão, serviço rodando continuamente) em vez de trocar pelo caminho que cobra de novo.
Antes de pedir reautenticação, simplesmente rode o comando e veja se ele se recupera:
`expiresAt` vencido não significa sessão perdida, porque o refresh token costuma continuar
válido.
