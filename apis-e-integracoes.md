# APIs e integrações

## Headers HTTP são case-insensitive — `dict(response.headers)` quebra isso

**Sintoma:** você pagina uma API, recebe 100 itens, o laço termina sem erro nenhum e
você conclui que existem 100. Na verdade existem 189.

**Causa:** a especificação diz que nome de header é case-insensitive, e servidores
HTTP/2 costumam mandar tudo em minúsculo (`x-next-page`). O objeto de resposta da
biblioteca respeita isso, mas ao converter para `dict` comum vira busca literal:
`d.get("X-Next-Page")` devolve `None` mesmo com o header presente.

**Regra:** nunca converta headers para `dict`. Use o objeto original da biblioteca.

**Como verificar:** compare o total acumulado com o header de contagem que a API
devolve. Um número redondo exato (100, 500, 1000) é suspeito por definição — é o tamanho
da página, não a realidade.

---

## Valide o total da paginação contra um número independente

**Sintoma:** um inventário "completo" que está truncado — e como nada falhou, o erro se
propaga para todas as decisões seguintes.

**Causa:** laços de paginação falham silenciosamente por vários motivos: header não
lido, limite de offset do servidor, filtro implícito, permissão que esconde itens.

**Regra:** todo laço de paginação termina com uma conferência contra o total declarado.
Melhor ainda: peça a mesma coleção por um segundo endpoint e compare.

---

## Webhook falha **fechada**, sempre

**Sintoma:** nenhum — esse é o problema. Qualquer um consegue postar um evento forjado e
mudar estado: marcar pedido como pago, ativar plano.

**Causa:** implementação típica é "se houver token configurado, valide; senão, siga".
Em ambiente onde a variável não foi propagada, o webhook passa a aceitar tudo.

**Regra:** sem token ou assinatura válida, responda 401 **antes** de ler o corpo. Campos
que selecionam comportamento privilegiado (plano, valor, papel) passam por allowlist,
nunca são refletidos direto do payload.

**Como verificar:**
```bash
curl -s -o /dev/null -w "%{http_code}" -X POST https://HOST/api/webhook -d '{}'   # 401
```

---

## Rotacionar token de webhook exige atualizar os dois lados na ordem certa

**Sintoma:** após trocar o segredo, o provedor acumula 401 e **pausa a fila de
entrega**; eventos param de chegar mesmo depois do token certo entrar em produção.

**Causa:** rotação feita só no receptor. Provedores costumam suspender a assinatura após
N falhas consecutivas e não retomam sozinhos.

**Regra:** aceite token novo e antigo → atualize no painel do provedor → remova o antigo.
Depois de qualquer sequência de 401, verifique e reative manualmente a fila e reenvie os
eventos perdidos.

---

## Conta compartilhada entre produtos: responda 200 ao evento que não é seu

**Sintoma:** a fila de entrega do provedor entra em backoff e eventos legítimos atrasam;
ou erros 500 esporádicos num endpoint estável.

**Causa:** quando duas aplicações compartilham a mesma conta no provedor, cada uma recebe
os eventos da outra. Devolver 500 para o evento alheio é interpretado como falha e
penaliza a fila inteira.

**Regra:** prefixe o identificador externo com um namespace da aplicação
(`app::tipo::parâmetro`), ignore silenciosamente com **200** o que não for seu, e mantenha
compatibilidade com referências antigas sem namespace. Use atualização em massa tolerante
a zero linhas em vez de "buscar e falhar se não achar".

---

## Webhook sozinho não é confirmação — tenha reconciliação

**Sintoma:** alguns eventos nunca chegam; o estado local fica "pendente" para sempre
enquanto o provedor já considera concluído. Ou: pagamento entra na conta e o pedido
continua pendente porque o cliente fechou a aba.

**Causa:** entrega de webhook é best-effort — cai por indisponibilidade momentânea, fila
pausada, deploy no meio do caminho. E polling só roda com a página aberta.

**Regra:** três camadas — polling (feedback imediato) + webhook (tempo real) +
reconciliação por consulta ao abrir a tela de gestão ou por job periódico (rede de
segurança). A reconciliação tem que ser idempotente. Com múltiplas abas em polling, troque
`update` por `updateMany` filtrando pelo status anterior e notifique só quando o contador
for 1 — senão o aviso sai duplicado.

---

## Falha silenciosa em efeito colateral "best-effort" produz buraco invisível

**Sintoma:** o registro principal existe e a interface parece correta, mas a busca ou
derivação nunca encontra nada. Nenhum erro no log.

**Causa:** o fluxo foi desenhado para não travar o caminho feliz: se o enriquecimento
(indexação, vetorização, notificação) falha, é engolido num `catch` vazio. Com a
credencial inválida por horas, todo o período fica sem enriquecimento.

**Regra:** efeito colateral opcional pode não travar a operação, mas **precisa** persistir
estado (`pendente`/`falhou` + timestamp + motivo) e ter um reprocessador que consulte esse
estado. Contadores de falha devem gerar alerta.

**Como verificar:**
```sql
SELECT count(*) FROM registros r
LEFT JOIN derivados d ON d.registro_id = r.id WHERE d.id IS NULL;
```

---

## `.catch(() => {})` numa remoção remota vira cobrança órfã

**Sintoma:** cobranças que ninguém consegue explicar continuam existindo no provedor
depois de o pedido ser apagado.

**Causa:** engolir o erro da chamada de remoção externa.

**Regra:** distinga os códigos — **404 é sucesso** (já não existe); **400 costuma ser
recusa legítima** ("cobrança já recebida não pode ser removida") e precisa chegar ao
operador. Se a limpeza remota falha, **não apague o registro local**, senão some o rastro.
Em limpeza em lote, preserve os que falharam.

---

## "Criar cliente" a cada transação duplica cadastro no provedor

**Sintoma:** o mesmo documento aparece com três cadastros distintos na conta do provedor.

**Causa:** `POST /customers` chamado em toda venda, sem busca prévia.

**Regra:** busque por identificador único primeiro e reuse; só crie se não achar. Se a
busca falhar por indisponibilidade, crie assim mesmo (não bloqueie a venda) e reconcilie
depois.

---

## Não monte URL de terceiro por concatenação — busque o link canônico

**Sintoma:** link gerado pelo sistema leva a "não encontrado" no site do provedor, mesmo
com o identificador correto.

**Causa:** a URL foi construída a partir de um ID interno seguindo um padrão observado. O
provedor usa outro identificador ou rota para acesso público, e o formato pode mudar sem
aviso.

**Regra:** use o campo de URL que a própria API retorna no recurso.

---

## Endpoint que consome recurso escasso precisa de limite de taxa **fora** do processo

**Sintoma:** estoque zerado, cobranças fantasma ou custo explodindo depois de uma rajada.

**Causa:** a rota muta recurso finito sem limite. Contadores em memória não funcionam em
serverless: cada invocação pode ser uma instância nova, sem estado compartilhado.

**Regra:** limite de taxa com estado externo (cache distribuído) ou na camada de borda da
plataforma. Além disso, só reserve o recurso escasso na confirmação, não na criação.

---

## Escolha a direção da falha pelo custo do erro

**Sintoma:** dilema recorrente — se o controle auxiliar (rate limit, cota,
contabilização) falhar, libera ou bloqueia?

**Causa:** bloquear por falha de mecanismo secundário derruba o produto; liberar por
falha de mecanismo de custo abre a torneira financeira.

**Regra:** controles de **proteção contra abuso** falham abertos; controles de **consumo
de recurso pago** falham fechados. Deixe a decisão explícita e comentada no código, com
log dos dois lados.

---

## Dado externo volátil: cache diário + último valor conhecido + padrão manual

**Sintoma:** uma API de terceiros fora do ar quebra uma tela inteira de relatório.

**Causa:** valor que muda devagar (câmbio, tabela de preços, feriados) buscado a cada
render, sem plano B.

**Regra:** três degraus — (1) cache com chave de data; (2) falhou a rede? devolve o
último valor cacheado; (3) sem cache nenhum? devolve o padrão configurável. Sempre com
timeout explícito: sem ele, `fetch` pode pendurar a função até o limite da plataforma.

---

## Corrida entre dois provedores elimina o pior caso de timeout

**Sintoma:** a operação ocasionalmente leva minutos porque um provedor externo está
lento, mesmo havendo alternativa disponível.

**Causa:** chamada sequencial com fallback: só depois de estourar o timeout do primeiro é
que o segundo é tentado, somando as latências.

**Regra:** para operações idempotentes e de custo baixo, dispare em paralelo e aceite a
primeira resposta **válida** — validação de conteúdo, não apenas "respondeu". Cancele as
demais explicitamente. Para operações caras ou com efeito colateral, mantenha sequencial
com timeout curto.

---

## Serviço gratuito de terceiro vira pago sem aviso

**Sintoma:** funcionalidade que rodava há meses começa a devolver 402/403 em massa.

**Regra:** para dependências não críticas, monte uma cascata explícita de 3-4 provedores
terminando num recurso local que nunca falha (placeholder gerado, degradação visual) —
melhor sair torto do que quebrar o layout. Cada nível registra qual foi usado, para você
perceber a degradação.

---

## Limite de tamanho em base64 é 33% mais permissivo do que você acha

**Sintoma:** uma imagem "dentro do limite" é recusada pelo provedor por exceder o
tamanho máximo.

**Causa:** base64 infla ~4/3. Validar o comprimento da string valida caracteres, não bytes.

**Regra:** valide sempre em bytes (`Buffer.byteLength(b64, 'base64')` ou o tamanho
reportado pelo storage).

---

## Serviço de visão com limite de tamanho: indexe uma derivada, não o original

**Sintoma:** reconhecimento passa a falhar em toda foto nova depois que você para de
degradar os uploads.

**Causa:** APIs de visão costumam ter teto rígido por imagem (ordem de 5 MB). Originais
de câmera passam disso com folga.

**Regra:** separe o arquivo **guardado** do arquivo **analisado** — guarde o original
intacto e mande para a API uma versão reduzida. Também respeita o limite de taxa e
barateia a conta.

---

## SSE atrás de proxy precisa de `X-Accel-Buffering: no`

**Sintoma:** o stream de progresso entrega tudo de uma vez, no final; ou funciona local e
trava em produção.

**Causa:** proxies reversos bufferizam a resposta por padrão.

**Regra:** envie `Content-Type: text/event-stream`, `Cache-Control: no-cache` e
`X-Accel-Buffering: no`. Além disso, um dicionário de jobs em memória só funciona com uma
instância — atrás de load balancer o cliente pode cair em outro processo. Documente a
restrição e adicione limpeza periódica dos jobs finalizados, senão o processo vaza memória.

---

## Timeout de TCP não prova firewall

**Sintoma:** conexão a um banco remoto dá timeout de ~75s no servidor de automação
enquanto o cliente gráfico do desenvolvedor conecta normalmente. Diagnóstico natural: "o
firewall bloqueia".

**Causa:** a configuração estava errada — porta e usuário diferentes dos reais. Pacote
para porta onde não há serviço produz timeout, exatamente a mesma assinatura de pacote
dropado por firewall.

**Regra:** antes de abrir chamado de rede, extraia host, porta, usuário e base do
**cliente que funciona** e compare campo a campo. Erro de credencial é ótima notícia:
prova que rede e IP estão liberados.

---

## Diagnóstico de egresso: teste destinos e portas cruzados

**Sintoma:** "não conecta" — e não se sabe se o bloqueio é da porta, do destino, do host
de origem ou do container.

**Causa:** um único teste falho é compatível com quatro causas diferentes.

**Regra:** monte uma matriz — (a) o destino em outra porta que costuma estar aberta (443);
(b) a mesma porta contra um serviço público de teste; (c) destinos neutros para provar
internet; (d) o IP público de saída via serviço de eco, em portas diferentes. Rode tudo de
dentro do container **e** do host para descartar o Docker.

---

## Teste de upload feito com `curl` no servidor não exercita CORS

**Sintoma:** o teste ponta a ponta passou, mas no navegador o envio direto ao storage é
bloqueado.

**Causa:** CORS é política aplicada pelo **navegador**; qualquer cliente HTTP fora dele
ignora completamente.

**Regra:** todo fluxo que sai do browser precisa de teste no browser ou de um preflight
`OPTIONS` explícito.

---

## `PutBucketCors` substitui a configuração inteira — leia antes de escrever

**Sintoma:** depois de habilitar CORS para um app novo, os uploads dos outros apps que
compartilham o bucket param de funcionar.

**Causa:** a API de CORS de storage S3-compatível é *replace*, não *merge*.

**Regra:** sempre `Get` → acrescentar a origem → `Put` com o conjunto completo. Mesmo
raciocínio para políticas de bucket e lifecycle.
