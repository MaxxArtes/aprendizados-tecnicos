# Deploy, build e plataforma

## Variável de ambiente nova só vale no próximo deploy

**Sintoma:** você adiciona a variável no painel, a feature continua desligada, e a
suspeita cai sobre o código.

**Causa:** o valor é injetado no artefato durante o build. Sem novo build, o runtime
continua com o valor anterior. Pior: se o código tem fallback silencioso, ele cai no
caminho degradado sem avisar.

**Regra:** toda mudança de variável exige redeploy. Um commit vazio
(`git commit --allow-empty`) é o gatilho mais barato. Documente isso junto da feature
flag, senão o próximo a mexer perde a mesma hora.

**Como verificar:** exponha uma rota de diagnóstico que devolva apenas
`Boolean(process.env.X)` — nunca o valor — e chame após o deploy.

---

## Deploy `BLOCKED` não é build quebrado — é o e-mail do autor do commit

**Sintoma:** todo push para de deployar. Não há logs de build para investigar, porque o
build nunca começou.

**Causa:** a plataforma recusa o deploy quando o e-mail do autor do commit não
corresponde a alguém com acesso ao projeto. Acontece quando a máquina tem um
`git config user.email` herdado de outro contexto.

**Regra:** configure `user.email` por repositório com o e-mail da conta que tem acesso.
Não caia na armadilha de procurar erro de build.

**Como verificar:** `git log -1 --format='%ae'` e o campo de estado do deployment — não
os build logs.

---

## Com integração git ativa, push na branch principal **é** o deploy de produção

**Sintoma:** um commit "só de ajuste" vai para produção sem ninguém rodar comando de
deploy.

**Regra:** saiba se o projeto tem integração git ativa **antes** de commitar. Trabalho em
progresso vai para branch (que gera preview). Se o fluxo exigir aprovação humana,
desligue a integração ou proteja a branch.

---

## O lint do CI é mais rigoroso que o dev local

**Sintoma:** `npm run dev` roda liso, mas o build na plataforma falha com
`'x' is defined but never used` ou `prefer-const`.

**Causa:** o servidor de desenvolvimento não roda a checagem completa; o build de
produção roda. Sobras de refatoração só aparecem lá.

**Regra:** rode `npm run build` localmente antes de qualquer push. Remova o parâmetro em
vez de deixá-lo, use `catch { }` sem binding, e use `const` sempre que a referência não
for reatribuída — mutar propriedades de um objeto **não** é reatribuição.

---

## `ignoreDuringBuilds` / `ignoreBuildErrors` compram velocidade a crédito

**Sintoma:** o build passa mesmo com erro de tipo, e o erro reaparece como falha em
runtime, em produção.

**Regra:** desligue apenas quando há portão de qualidade **em outro lugar** (CI rodando
`tsc --noEmit` e lint separadamente, bloqueando o merge). Sem esse portão, você não
removeu a verificação — só a moveu para produção.

---

## `grep` que encontra "Failed" retorna sucesso e empurra build quebrado

**Sintoma:** pipeline do tipo `npm run build | grep -E "Compiled|Failed" && git push`
empurra código que não compila.

**Causa:** `grep` sai com 0 quando **acha** o padrão. Achar "Failed" é sucesso para o
shell, e o `&&` dispara o push.

**Regra:** nunca use presença de string como portão. Use o exit code do build, ou conte
ocorrências: `test "$(... | grep -c Failed)" -eq 0`.

---

## Em comando composto, `&&` e `;` decidem qual código de saída sobrevive

**Sintoma:** um build que falhou é reportado como sucesso porque um `cp` de limpeza rodou
depois.

**Causa:** o retorno do comando composto é o do **último** comando executado.

**Regra:** encadeie o passo de persistência com `&&` e feche o bloco acessório com
`; true` para que um erro cosmético não mascare o retorno real.

---

## Build pesado no runner do CI, montagem nativa no servidor de destino

**Sintoma:** ou o build direto na VPS consome toda a RAM e derruba a aplicação, ou o build
no CI gera uma imagem que morre no boot com `exec format error`.

**Causa:** o runner gratuito do CI é x86_64; VPS baratas de bom desempenho por watt
costumam ser ARM64. Emular ARM via QEMU leva dezenas de minutos. Já compilar na máquina de
produção rouba CPU e RAM de quem está sendo atendido.

**Regra:** quebre em dois jobs. Job 1 (runner do CI): instala dependências, roda o build e
publica um artefato com a saída compilada + manifesto + Dockerfile. Job 2 (runner na
máquina de destino): baixa o artefato e roda `docker build`, que só instala dependências
nativas de produção — segundos, na arquitetura certa. O Dockerfile do job 2 nunca roda o
build da aplicação.

**Como verificar:**
```bash
docker image inspect <img> --format '{{.Architecture}}'   # deve bater com uname -m
```

---

## `docker builder prune` incondicional + `--no-cache` torna todo build frio

**Causa:** reflexo de "limpar disco" aplicado a cada execução, matando o cache de camadas
que existe exatamente para isso.

**Regra:** pode o cache **condicionalmente**, só quando o disco aperta. Com o manifesto de
dependências constante, a camada de instalação vira cache hit permanente.

**Como verificar:**
```bash
df --output=pcent /var/lib/docker | tail -1 | tr -dc '0-9'   # pode acima de ~85%
```

---

## Pré-valide o build real em container quente

**Sintoma:** erros que só aparecem no `docker build`, com log truncado, depois de dez
minutos. `tsc --noEmit` passou.

**Causa:** checagem de tipos não detecta erros específicos do framework (props
assíncronas, diretiva de componente cliente faltando, import inválido em contexto de
servidor). E `--no-cache` reinstala tudo antes de falhar.

**Regra:** mantenha uma imagem base com dependências já instaladas e rode o build real
dentro dela montando o código como volume (30–60s). Persista a saída para a imagem final
reaproveitar: `if [ -d .next/standalone ]; then echo reaproveitando; else npm run build; fi`.

---

## Backfill em função serverless morre no timeout

**Sintoma:** a rota de reprocessamento retorna sucesso parcial, ou é cortada pela
plataforma; o operador acaba apertando F5 dezenas de vezes.

**Causa:** funções serverless têm teto de duração e memória.

**Regra:** a rota processa um lote e devolve `{processados, falhas, restantes}`. Quem
itera é um loop no cliente, um cron ou uma fila — nunca o humano. Selecione sempre pelos
itens **ainda não processados**, não por offset, para não pular registros.

---

## A plataforma rejeita corpos grandes antes da sua função rodar — e devolve HTML

**Sintoma:** upload "às vezes funciona": foto de celular falha, print passa. O cliente
mostra mensagem genérica porque `res.json()` estourou.

**Causa:** o limite de payload (tipicamente ~4,5 MB) é do runtime da plataforma. A
resposta 413 é uma página HTML, não JSON — e o `.json()` lança antes de você ler o status.

**Regra:** (1) comprima no navegador antes de enviar; (2) melhor ainda, tire o binário do
caminho da função com upload assinado direto para o storage; (3) no cliente,
`await res.json().catch(() => ({}))` e trate 413 com mensagem específica.

---

## Região do compute longe da região do banco é o gargalo real

**Sintoma:** uma página administrativa leva 2,5 s. O instinto diz "roteamento".

**Causa:** as funções rodavam num continente e o banco em outro (~120 ms por ida e volta).
Uma página com 14 consultas, várias em série, multiplica isso.

**Regra:** (1) fixe a região das funções ao lado do banco, e faça isso **em arquivo de
configuração versionado** — ele vence o painel, que pode continuar mostrando o valor
antigo; (2) paralelize as ondas de `await`, inclusive as do layout, que rodam em toda
navegação; (3) páginas dinâmicas não são pré-buscadas por links — um esqueleto de
carregamento evita a sensação de congelamento.

---

## CDN não resolve latência de SSR

**Sintoma:** plano de migrar de plataforma serverless para VPS mais barata "sem perder
velocidade".

**Causa:** um CDN acelera estático; SSR, rotas de API e consultas continuam na origem. Se
a origem muda de continente e o banco fica, cada consulta paga a travessia.

**Regra:** classifique a aplicação antes de migrar. **Dominada por round-trip de banco**
(listagens, painéis com muitas consultas) exige compute e banco na mesma região.
**Dominada por chamada externa lenta** (geração por LLM, que custa dezenas de segundos)
tolera origem distante.

---

## Meça antes de otimizar: o gargalo costuma ser payload, não roteamento

**Sintoma:** "navegação lenta" numa galeria.

**Causa:** medindo em produção — a rota de listagem era dinâmica sem cache e refeita a cada
volta (~1,4 s de spinner), e cada miniatura de ~300 px baixava o arquivo original de
~470 KB.

**Regra:** gere derivadas no servidor (miniatura de ~500 px costuma cortar mais de 95% dos
bytes), sirva a lista com cache de borda, e ofereça um parâmetro de "recarregar sem cache"
para a área administrativa.

---

## Desligue a otimização de imagem da plataforma quando o CDN já serve derivadas

**Causa:** se você já gera variantes no upload e as serve por um CDN, a otimização
on-the-fly é trabalho duplicado, pago e com padrões remotos a manter.

**Regra:** `images: { unoptimized: true }` + `srcset` próprio. Faça o oposto quando as
imagens são poucas, de origem heterogênea e você não quer manter pipeline de derivadas.

---

## Não fixe a URL base de autenticação quando existem deploys de preview

**Sintoma:** OAuth falha com `invalid_grant: Invalid code verifier` ou
`redirect_uri_mismatch` só nos previews; produção funciona.

**Causa:** cada preview tem host diferente. Uma variável fixa de URL base faz a biblioteca
montar o callback com o domínio errado, e o cookie do verifier fica preso a outro host.

**Regra:** cadastre no provedor OAuth os domínios estáveis e **remova** as variáveis de
URL base fixa, deixando a plataforma resolver. Mantenha o segredo de assinatura de sessão
idêntico entre ambientes que compartilham cookies. Para depurar, teste em aba limpa.

---

## Redirect e rota baseados em **host** não podem ser testados em preview

**Sintoma:** a regra "funciona" em preview porque simplesmente não é acionada.

**Causa:** a condição casa o host de produção; o domínio de preview é outro.

**Regra:** regras host-gated (redirect de `www`, roteamento por subdomínio) e verificações
de SEO só se validam em produção. Implemente **em código versionado**, não no painel.

**Como verificar:** `curl -sI https://www.dominio` deve devolver 308 preservando o path.

---

## Subdomínio `www` não herda o certificado do apex

**Regra:** registre as duas variantes no projeto e redirecione uma para a outra com 308,
decidindo qual é canônica e refletindo isso no `canonical` e no sitemap.

---

## Build da aplicação não roda migração de banco

**Sintoma:** deploy verde, aplicação com erro de coluna ou tabela inexistente.

**Causa:** o script de build típico gera o client do ORM, mas não aplica schema.

**Regra:** deixe explícito no runbook que migração é passo separado. Alterações
**aditivas** são seguras antes do deploy do código; alterações destrutivas exigem duas
fases — deploy que tolera os dois formatos → migração → deploy que limpa.

---

## Nível "hobby" de plataforma proíbe uso comercial, inclusive doações

**Sintoma:** o projeto roda dentro dos limites técnicos do plano gratuito e ainda assim
está em violação.

**Causa:** a restrição é de **termos de uso**, não de quota. "Comercial" costuma abranger
qualquer ganho financeiro de qualquer envolvido — processar pagamento, ser pago para
manter, e até solicitar doação.

**Regra:** separe a decisão em duas — cabe nos limites? é permitido pelos termos? A segunda
decide. Note também que o plano gratuito costuma ter **teto rígido** (o projeto pausa)
enquanto o pago cobra excedente: o risco não é a conta, é o downtime.

---

## Distinga add-on **flat** de add-on **por uso** antes de cortar custo

**Sintoma:** a fatura está acima do valor do assento e ninguém sabe por quê.

**Causa:** add-ons de preço fixo por projeto (às vezes ligados sem intenção) somam dezenas
de dólares, enquanto ferramentas de observabilidade por uso podem custar centavos.

**Regra:** antes de desligar observabilidade "para economizar", verifique o modelo de
preço. Retenção estendida de log e latência por rota costumam ser baratíssimas e são
exatamente o que resolve incidente.
