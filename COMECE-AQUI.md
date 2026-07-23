# Comece aqui

Este repositório tem 235 lições. Você não vai ler todas, e não deve.

Ele é uma **referência**, não um curso: serve para quando você já está com um problema na
tela. O problema é que, para procurar, você precisa saber descrever o que está acontecendo —
e no começo você não sabe. Esta página existe para resolver isso.

Se você está na faculdade, estagiando, no primeiro emprego ou aprendendo por conta, comece
pelo trecho abaixo que descreve **onde você está**.

---

## "Funciona na minha máquina e quebra quando eu subo"

O erro de iniciante mais comum não é de código — é achar que o ambiente de desenvolvimento e
o de produção são a mesma coisa. Eles não são, e as diferenças são sempre as mesmas.

1. [O lint do CI é mais rigoroso que o dev local](deploy-e-build.md) — variável não usada
   passa no seu computador e derruba o build lá
2. [Variável de ambiente nova só vale no próximo deploy](deploy-e-build.md) — você mudou a
   chave no painel e nada aconteceu; é isso
3. [Build da aplicação não roda migração de banco](deploy-e-build.md) — deploy verde, site
   com erro 500
4. [`relation "x" does not exist`](banco-de-dados.md) — o mesmo problema visto do outro lado
5. [Exceção lançada em Server Action é sanitizada em produção](frontend-e-nextjs.md) — em
   desenvolvimento a mensagem aparece; em produção some, e você fica sem diagnóstico
6. [Imagem nginx com `USER` não-root entra em crash loop](infraestrutura-e-containers.md) —
   sobe na sua máquina, mas no ar o container reinicia sem parar e o proxy responde 404

> Se aparecer uma palavra que você não conhece, o [glossário](GLOSSARIO.md) tem todas em uma
> linha cada.

---

## "Botei no ar e agora tem gente usando"

O que muda quando existem usuários de verdade: coisas passam a acontecer **ao mesmo tempo**,
e o que dava certo com uma pessoa dá errado com duas.

1. [Leitura-modificação-escrita em estoque entrega o último item duas vezes](banco-de-dados.md)
   — o primeiro bug de concorrência que quase todo mundo escreve
2. [Autenticação por rota, não por página](seguranca-e-segredos.md) — esconder o botão não
   protege nada
3. [`include` de ORM para "só contar" carrega todas as linhas filhas](banco-de-dados.md) —
   por que o site fica lento sozinho, sem você mexer em nada
4. [Nunca confie no primeiro item de `X-Forwarded-For`](seguranca-e-segredos.md) — seu
   bloqueio por tentativas de login provavelmente não bloqueia nada
5. [Endpoint de seed ou bootstrap esquecido é backdoor ativa](seguranca-e-segredos.md)

---

## "Vou mexer com senha, chave e API de terceiro"

Aqui os erros custam caro e são difíceis de desfazer. Vale ler antes de precisar.

1. [`process.env.X || "<valor real>"` é vazamento disfarçado de robustez](seguranca-e-segredos.md)
   — o hábito que mais coloca chave real dentro de repositório
2. [Prefixo `NEXT_PUBLIC_` / `VITE_` é contrato, não convenção](seguranca-e-segredos.md) —
   uma letra de diferença no nome e a chave vai para o navegador
3. [`*.env` no `.gitignore` não pega `.env.bak`](git.md) — você achou que estava protegido
4. [Webhook falha fechada, sempre](apis-e-integracoes.md) — se qualquer um pode chamar a URL,
   qualquer um pode dizer que o pedido foi pago
5. [Rotação: crie o novo e propague antes de revogar o antigo](seguranca-e-segredos.md) — a
   ordem importa, e errá-la derruba produção

**Se você já cometeu algum destes:** não entre em pânico e não apague o histórico primeiro.
A ordem certa é rotacionar a credencial (o valor velho deixa de valer), depois limpar o
código. Reescrever histórico é o último passo, e nem sempre necessário.

---

## "Preciso automatizar alguma coisa que roda sozinha"

Script agendado é onde a falha silenciosa mora. Ele não avisa que parou — simplesmente para.

1. [Toda tarefa agendada precisa de um arquivo de status legível por humano](automacao-e-agendamento.md)
2. [O modo de morte mais comum de automação é a credencial expirar](automacao-e-agendamento.md)
3. [Teste a tarefa agendada **pelo agendador**, não pelo seu terminal](automacao-e-agendamento.md)
   — ambiente e permissões são outros
4. [Grave o novo, valide, só então substitua o bom](automacao-e-agendamento.md) — como não
   destruir um backup bom com um ruim
5. [Log vazio de processo em execução não é sinal de travamento](automacao-e-agendamento.md) —
   antes de matar o processo, leia isto

---

## "Estou usando IA para escrever código"

1. [Agente que narra sucesso não substitui verificação](ferramentas-de-ia.md) — "está 100%
   funcionando" não é evidência; código HTTP e log são
2. [Layout, contraste e ícone são o que build e linter nunca reprovam](ferramentas-de-ia.md)
3. [Valide contra o contexto todo ID que o modelo devolver](ferramentas-de-ia.md) — a saída do
   modelo é entrada não confiável, como qualquer dado de fora
4. [`middleware` com matcher amplo cria loop de redirect](frontend-e-nextjs.md) — erro que
   modelos repetem com frequência; peça explicitamente no prompt para evitar

---

## Se for ler só três coisas

Estas atravessam todas as categorias e valem mais que qualquer lição isolada:

**Ausência de erro não é evidência de sucesso.** Paginação que para na primeira página.
Remoção de diretório que não remove. Job que parou há semanas. Nenhum deles emite erro.
Verifique o **efeito** — o arquivo sumiu? a data avançou? — nunca o código de saída.

**O sintoma quase nunca aponta para a causa.** Antes de acusar a camada mais distante e cara
de investigar (rede, provedor, infraestrutura), prove que o que está perto está certo. A
maioria dos "problemas de firewall" é configuração errada.

**Confiança precisa de fronteira explícita.** O que vem do navegador, do usuário, de um
modelo de IA ou de uma API de terceiro é *alegação*, não fato. Toda vez que um dado atravessa
uma fronteira, alguém precisa decidir conscientemente se confia nele.

---

## Como usar daqui pra frente

Quando tiver um problema concreto, vá ao [índice por sintoma](INDICE-POR-SINTOMA.md) e
procure pelo que você está vendo na tela — não pelo nome da tecnologia.

Se topar com um termo desconhecido, o [glossário](GLOSSARIO.md) resolve em uma linha.

E se nada aqui servir, tudo bem: este repositório cobre os erros que **uma pessoa** cometeu.
O universo de formas de errar é bem maior.
