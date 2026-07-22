# Frontend, React e Next.js

## `params`, `searchParams`, `cookies()` e `headers()` são assíncronos no Next 15

**Sintoma:** após atualizar de 14 para 15, rotas quebram com erros de tipo ou valores
`undefined` ao ler parâmetros em Server Components e route handlers.

**Causa:** breaking change — essas APIs passaram a devolver `Promise`. Em Client
Components, `useParams()` continua síncrono, o que faz o erro parecer aleatório.

**Regra:** declare `params: Promise<{...}>` e use `await params`. Para escrever cookie em
route handler, use `res.cookies.set/delete` no objeto de resposta.

**Como verificar:** `npx tsc --noEmit` acusa a maioria.

---

## Exceção lançada em Server Action é sanitizada em produção

**Sintoma:** em desenvolvimento a mensagem de erro aparece corretamente; em produção o
usuário vê "ocorreu um erro inesperado" e você fica sem diagnóstico.

**Causa:** o framework remove a mensagem original em produção para não vazar detalhes de
servidor. O bug só se manifesta no ar.

**Regra:** não use `throw` para comunicar erro de negócio a partir de Server Action.
**Retorne** um objeto (`{ ok: false, error: "motivo legível" }`) e trate na UI. Reserve
`throw` para falhas realmente inesperadas.

**Como verificar:** rode o build de produção localmente e provoque o erro — o
comportamento de dev mente aqui.

---

## Aviso de "Dynamic server usage" no build não é erro

**Sintoma:** o log exibe `Route /api/... couldn't be rendered statically because it used
headers()`.

**Causa:** o framework tenta pré-renderizar estaticamente; ao encontrar acesso a dados por
requisição, desiste e trata a rota como dinâmica — comportamento esperado.

**Regra:** ignore. Só declare `dynamic = 'force-dynamic'` para silenciar ruído ou tornar
a intenção explícita.

---

## Comentário `//` dentro de uma tag JSX quebra o build

**Causa:** entre `<Componente` e `>` o parser está em contexto de atributos, onde `//`
não é comentário.

**Regra:** use `{/* ... */}` fora dos atributos, ou comente acima do elemento.

---

## `middleware` com matcher amplo cria loop de redirect na própria página de login

**Sintoma:** `ERR_TOO_MANY_REDIRECTS` ao acessar a área administrativa.

**Causa:** o matcher `/area/:path*` inclui `/area/login`, então o middleware redireciona a
página de login para ela mesma.

**Regra:** primeira linha do middleware: liberar explicitamente as rotas públicas do
prefixo protegido. Se o código é gerado por IA, coloque isso como requisito explícito no
prompt — é um erro que ela repete.

---

## `navigator.share` precisa ser chamado no mesmo tick do clique

**Sintoma:** no Android o botão "Compartilhar" nunca abre a bandeja e cai no fallback;
no desktop funciona.

**Causa:** havia um `await fetch(imagem)` **entre** o clique e a chamada. O navegador
perde a ativação transitória durante o await e rejeita a API.

**Regra:** prepare tudo o que a API precisa **antes** — no mount ou num efeito — e no
handler chame de forma síncrona. Vale para qualquer API que exija gesto: abrir janela,
fullscreen, clipboard, áudio.

---

## `AbortError` em `navigator.share` é o usuário cancelando

**Regra:** no `catch`, cheque `err.name === 'AbortError'` e saia silenciosamente. Só o
erro real dispara o fallback.

---

## `<img>` exibe imagem cross-origin que `fetch` não consegue baixar

**Sintoma:** a imagem aparece na tela normalmente, mas `fetch(url).then(r => r.blob())`
falha com "Failed to fetch" — e o CORS do servidor está correto.

**Causa:** duas coisas distintas, ambas custosas de diagnosticar. Primeira: exibir imagem
não exige CORS, ler os bytes exige. Segunda, mais traiçoeira: a tag `<img>` busca em modo
`no-cors` e guarda no cache HTTP uma resposta **opaca**; o `fetch` seguinte casa com essa
entrada e não consegue ler o corpo — mesmo depois de o bucket ser configurado
corretamente.

**Regra:** use `fetch(url, { cache: 'no-store' })`, ou `crossOrigin="anonymous"` na `<img>`
desde o início. Quando precisar dos bytes de forma confiável, sirva por uma rota de mesma
origem que busca no servidor — com guarda anti-SSRF validando que a URL é do seu bucket.

---

## `fetch` não reporta progresso de **envio**

**Regra:** para barra de progresso de upload, use `XMLHttpRequest` e
`upload.onprogress`. Não é legado — é a única API que entrega esse dado hoje.

---

## Imagem vinda do cache não dispara `onLoad` no React

**Sintoma:** fotos somem (ficam em `opacity-0`) quando o usuário volta para uma página já
visitada.

**Causa:** se a imagem já está completa quando o React monta o elemento, o evento `load`
já passou.

**Regra:** no `ref`, cheque `img.complete` e dispare o mesmo handler manualmente, além de
manter o `onLoad`.

---

## O descritor `w` do `srcset` é a **largura** do arquivo, não o lado maior

**Sintoma:** em galeria com fotos em retrato, o navegador escolhe consistentemente um
arquivo de resolução menor do que precisaria; imagens saem levemente borradas.

**Causa:** ao gerar derivadas com redimensionamento "caber dentro de N×N", uma imagem em
retrato de lado maior 1600 tem largura real ~900. Declarar `1600w` faz o navegador achar
que o candidato é ~1,8x maior do que é.

**Regra:** guarde largura e altura do original e calcule
`round(largura × min(1, ladoMáx / max(largura, altura)))`. Sem as dimensões originais o
cálculo é impossível — grave-as no momento em que as derivadas são geradas.

**Como verificar:** compare `img.currentSrc` com a largura real do arquivo servido em cada
breakpoint. Cuidado: com `srcset`, `img.naturalWidth` vem dividido pela densidade
atribuída ao candidato e **não** devolve os pixels reais do arquivo.

---

## `loading="lazy"` não adia slides de carrossel

**Causa:** os slides estão **dentro** da viewport, apenas deslocados por transform. O
navegador considera todos visíveis.

**Regra:** carregue explicitamente só o slide atual e o próximo; controle por estado.

---

## Processamento de imagem em canvas congela a interface

**Sintoma:** ao selecionar algumas centenas de fotos para upload, a página fica minutos sem
responder.

**Causa:** o canvas roda na mesma thread que pinta a tela.

**Regra:** mova a compressão para `Worker` + `OffscreenCanvas`, com um pool pequeno
(ordem de 4). O código do worker pode ser embutido como Blob para evitar arquivo extra no
build.

---

## Variável CSS de fonte precisa estar no `<html>`, não no `<body>`

**Sintoma:** o site inteiro cai para uma fonte serifada padrão. Tipagem, lint e build
passam limpos.

**Causa:** o preflight de frameworks utilitários aplica `font-family` no `<html>`. Se a
regra referencia um `var()` **indefinido** naquele escopo, o CSS invalida a **declaração
inteira** e o navegador usa o default.

**Regra:** aplique a classe da fonte no `<html>`. E valide sempre a `font-family`
**computada**, nunca o CSS gerado.

**Como verificar:** `getComputedStyle(document.body).fontFamily` no navegador real.

---

## Variantes `dark:` empatam com `hover:` e vencem pela ordem

**Sintoma:** no modo escuro o hover para de funcionar em quase todos os botões — exceto
num que usa gradiente.

**Causa:** `dark:bg-*` e `hover:bg-*` têm a mesma especificidade; decide a ordem no CSS
gerado. O gradiente "funciona" porque pinta `background-image`, que fica por cima de
`background-color`.

**Regra:** use o modificador de importância ou reorganize as camadas. Ao conferir CSS
compilado, **grepe o seletor, não a cor** — a declaração vira `rgb(...)` e o hex some.

---

## Use `dvh` em vez de `vh` para altura útil no celular

**Causa:** `vh` congela a viewport maior; não acompanha a retração da barra do navegador.

**Regra:** `calc(100dvh - <alturas fixas>)`. Em iOS, `safe-area-inset-*` só funciona com
`viewport-fit=cover` — sem isso o inset é 0.

---

## Alvo de toque é área, não desenho

**Sintoma:** bolinhas de carrossel de 6 px, links de topo de 18 px, botão de fechar
minúsculo — tudo bonito e impossível de acertar.

**Regra:** envolva o desenho num botão de ~44 px (mínimo 40 no celular) mantendo o visual
idêntico, com `aria-label`. Em telas sem hover, ações que no desktop aparecem no hover
precisam ficar **sempre visíveis** — e por isso maiores.

**Como verificar:** percorra `document.querySelectorAll('button,a,input')` medindo
`getBoundingClientRect()` e liste tudo abaixo de 40 px.

---

## Cores com alpha da mesma matiz convergem para contraste 1.0 no modo escuro

**Sintoma:** uma etiqueta de destaque fica **invisível** no tema escuro.

**Causa:** fundo `cor-da-marca/10` sobre superfície escura e texto `cor-da-marca` acabam
com quase a mesma luminância.

**Regra:** em tema escuro use fundo sólido para badges e recalcule contraste real (alvo
4.5:1). Cinza "secundário" costuma ser o segundo pior ofensor.

---

## Menu absoluto dentro de card com `overflow-hidden` é cortado

**Regra:** renderize o menu num portal para o `<body>` e posicione por coordenadas.
Cheque também z-index de widgets flutuantes: um balão de ajuda fixo pode cobrir o botão
principal de checkout.

---

## O botão de sair some justamente para quem está logado

**Sintoma:** em tela estreita a barra de topo quebra e o penúltimo item é o primeiro a
ser empurrado para fora.

**Regra:** não deixe ação crítica de sessão depender do espaço remanescente do header.
Coloque identidade e sair dentro da própria área autenticada.

---

## Emoji é fonte: dimensione com `font-size`

**Causa:** emoji é glifo tipográfico, não SVG. Classes utilitárias de `width/height` não
controlam o desenho.

**Regra:** renderize num `<span>` com `font-size` em px. Vantagem colateral: cada aparelho
mostra o emoji do próprio sistema.

---

## Cota "mensal" calculada em UTC reseta cedo para o usuário

**Causa:** o início do período foi calculado com o relógio UTC do servidor, enquanto o
usuário vive em outro fuso.

**Regra:** limites, cotas e agrupamentos por dia/mês são calculados no fuso do **negócio**.
Defina numa constante única e use em toda parte — inclusive nos rótulos gerados no front,
senão o gráfico desloca em relação à consulta.

---

## Tracking de visita curta registra 0s se o primeiro ping demora

**Causa:** o heartbeat só disparava a cada 15s; quem saía antes nunca gerava um segundo
evento, e a duração (`último - primeiro`) dava zero.

**Regra:** mande um primeiro ping cedo (~4s) e um evento final no
`pagehide`/`visibilitychange`, usando `navigator.sendBeacon` — o `fetch` normal é cancelado
quando a página é descarregada. Calcule a duração no **servidor**.

---

## O caminho do cron precisa repetir as validações do caminho interativo

**Sintoma:** um usuário que perdeu o plano pago continua recebendo o benefício, só pelo
agendamento.

**Causa:** a checagem de permissão foi implementada no endpoint da interface, e o job
agendado chama a função de negócio direto.

**Regra:** coloque a validação **dentro** da função de domínio, não no controller.

---

## Push web no iOS só funciona com o app instalado

**Sintoma:** notificações funcionam no Android e desktop e nunca chegam no iPhone.

**Causa:** no iOS a permissão de push só existe para PWA adicionado à tela de início. Se
só a área pública tem manifesto, a área administrativa não é instalável — e é justamente
ela que precisa do aviso.

**Regra:** manifesto e metadados por área instalável. Na UI, seja honesto sobre os canais:
e-mail (sempre ativo), push (com o motivo quando indisponível) e a instrução específica do
iOS.

---

## `iframe srcDoc` com documento completo não pode ser embrulhado

**Causa:** envolver um documento que já tem `<!doctype>`/`<html>` em outro HTML produz
DOCTYPE duplicado.

**Regra:** use o documento como veio; valide a presença de `<!doctype`/`<html>` e recuse
ou limpe a saída em vez de embrulhar.

---

## Buffer apoiado em `SharedArrayBuffer` é recusado pelo SDK de storage

**Sintoma:** erro "The input argument must be ArrayBuffer. Received SharedArrayBuffer" ao
enviar uma imagem processada; funciona local e só quebra ao subir.

**Causa:** bibliotecas nativas de processamento de imagem podem devolver um `Buffer` sobre
memória compartilhada; o SDK precisa hashear o corpo para assinar a requisição e recusa
esse tipo.

**Regra:** copie antes de enviar —
`const out = new Uint8Array(buf.length); out.set(buf);`. Dica: a presença de metadados do
SDK no objeto de erro denuncia quem lançou.

---

## Binário nativo em função serverless precisa ser declarado externo ao bundle

**Sintoma:** erro de módulo não encontrado apenas no ambiente de deploy, com
processamento de imagem, PDF ou planilha.

**Causa:** o bundler tenta empacotar pacotes com dependências nativas (`.node`), que não
sobrevivem à transformação.

**Regra:** liste-os em `serverExternalPackages` no config do framework.

---

## Cliente de banco em singleton global sobrevive ao hot reload

**Sintoma:** em desenvolvimento, esgotamento de conexões após alguns salvamentos.

**Causa:** o hot reload reavalia o módulo e cria um pool novo a cada recarga, sem fechar o
anterior.

**Regra:** guarde a instância em `globalThis` e reutilize — só fora de produção, onde cada
instância serverless deve ter seu próprio cliente.

---

## Renomear entidade que gera slug quebra todo link já compartilhado

**Causa:** o slug é derivado do nome e regenerado na atualização, sem histórico.

**Regra:** guarde histórico de slugs e responda com redirect permanente do antigo para o
atual. Custa uma tabela e evita perda de tráfego e de autoridade de busca.

---

## O alfabeto padrão de geradores de ID tem caracteres inválidos em hostname

**Causa:** geradores populares de ID curto incluem `_` e maiúsculas; hostname aceita
apenas `[a-z0-9-]`.

**Regra:** use alfabeto customizado minúsculo e sem sublinhado para slugs que virarão
hostname, e faça as buscas por slug case-insensitive em todos os pontos.

---

## Rota de `sitemap` dinâmica congela no build sem revalidação

**Regra:** declare `export const revalidate = <segundos>`. Exclua conteúdo de demonstração
por um critério robusto (dono ou flag), não por slug — slug muda.

---

## `title.template` faz sua marca "vampirizar" o nome do cliente

**Sintoma:** toda página de inquilino aparece no buscador como "Nome do Cliente — Sua
Marca".

**Regra:** use template no seu próprio conteúdo e título **absoluto** nas páginas de
inquilino. Adicione dados estruturados — é o que habilita resultado enriquecido.
