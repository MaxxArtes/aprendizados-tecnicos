# Banco de dados

## Integrações gerenciadas criam seis connection strings quase iguais

**Sintoma:** a aplicação funciona, mas a migration falha com erro de conexão ou timeout de
transação. Ou o inverso.

**Causa:** integrações de Postgres gerenciado criam meia dúzia de URLs para o mesmo banco,
variando em dois eixos: passa pelo pooler ou não, e quais parâmetros de query carrega
(`sslmode`, `channel_binding`, `connect_timeout`). Nomes diferentes podem ter o mesmo
valor, e **nomes sinônimos podem ter valores diferentes** — `DATABASE_URL_UNPOOLED` e
`POSTGRES_URL_NON_POOLING` não são a mesma coisa, apesar do nome sugerir.

**Regra:** runtime da aplicação usa a URL **com** pooler; migrations e DDL usam a **sem**
pooler — pooler em modo transaction quebra transação longa e statement preparado. A
variante sem `sslmode` só serve para debug local. Documente qual é qual, porque não dá
para deduzir pelo nome.

**Como verificar:** diferencie pelo host (o do pooler tem sufixo próprio) e pela query
string, nunca pelo nome da variável.

---

## Em Postgres serverless, prefira o adaptador HTTP ao WebSocket

**Sintoma:** erro de adapter no login e, nos logs, um crash fatal do Node
(`ERR_INVALID_ARG_TYPE`) dentro da classe `WebSocket`; a interface mostra apenas um erro
genérico de configuração.

**Causa:** o adaptador padrão usa a biblioteca `ws` para falar com o pooler. Em runtime
serverless, essa combinação (WebSocket não-nativo + pooler + conexões efêmeras) é frágil.

**Regra:** use a variante HTTP do adaptador, que trafega em 443 via `fetch` nativo.
Aponte-o para a string **não pooled** quando disponível.

---

## `relation "x" does not exist` significa que a migração não faz parte do deploy

**Causa:** o deploy publicou o código sem aplicar o DDL. Criar a tabela por um endpoint
improvisado só transfere o problema para o próximo ambiente.

**Regra:** esquema versionado e aplicado no pipeline. Rota administrativa de setup, se
existir, é idempotente, autenticada e temporária.

**Como verificar:** `SELECT to_regclass('public.minha_tabela');` antes de liberar a feature.

---

## DDL idempotente em tempo de requisição: atalho legítimo com prazo de validade

**Sintoma:** rotas chamando `CREATE TABLE IF NOT EXISTS` no caminho quente.

**Causa:** sem ferramenta de migração no projeto, é a forma mais rápida de garantir o
esquema logo após o deploy — e realmente evita a classe de bug "deployei o código antes da
migração".

**Regra:** aceitável em projeto pequeno **desde que** a DDL seja estritamente idempotente,
esteja isolada num `try/catch` que não derrube a requisição, e rode em um ponto único —
não espalhada por dezenas de handlers. Com mais de um ambiente ou mais de um dev, migre
para migrações versionadas: o custo por requisição e o risco de lock em tabela grande
crescem rápido.

**Como verificar:** se `git grep -c "IF NOT EXISTS"` cresce a cada sprint, passou da hora.

---

## `operator does not exist: uuid = text` é erro de modelagem

**Causa:** a tabela nova foi criada com a coluna de referência como `TEXT` enquanto a PK
referenciada é `UUID`. PostgreSQL não faz coerção implícita.

**Regra:** confira o tipo real da PK antes de criar tabelas dependentes. **Nunca** remova
a chave estrangeira só para o erro sumir — isso mascara a incompatibilidade e deixa órfãos
entrarem.

---

## Leitura-modificação-escrita em estoque entrega o último item duas vezes

**Causa:** `SELECT` → subtrai em memória → `UPDATE` não é atômico.

**Regra:** atualização condicional atômica —
`UPDATE ... SET qtd = qtd - n WHERE id = ? AND qtd >= n` — e verifique o número de linhas
afetadas.

**Como verificar:** dispare N requisições concorrentes com o estoque em 1 e confirme que
só uma passa.

---

## `include` de ORM para "só contar" carrega todas as linhas filhas

**Sintoma:** feed fica progressivamente mais lento conforme o engajamento cresce, sem
nenhuma query aparentemente pesada.

**Causa:** o include trazia **todas** as reações apenas para exibir um contador.

**Regra:** substitua por `groupBy` agregando por chave + uma consulta única do que é do
usuário logado. Contadores devem ser agregação ou coluna materializada.

---

## Toda tabela materializada precisa nascer com plano de atualização

**Sintoma:** o relatório funciona no dia da entrega e envelhece silenciosamente.

**Causa:** diferente de uma view, uma tabela criada por `CREATE TABLE AS SELECT` é um
snapshot estático.

**Regra:** se a escolha for tabela — comum quando a ferramenta de BI trabalha melhor com
tabela física — a entrega **inclui** o mecanismo de refresh. Registre também a dependência
entre tabelas.

---

## Alias de schema com views reconecta painel legado sem tocar no painel

**Sintoma:** um relatório antigo quebra porque o pipeline novo grava em outro
schema, e reapontar dezenas de consultas é caro.

**Regra:** crie o schema com o nome antigo e, dentro dele, objetos apontando para os novos.
O painel resolve pelo nome que espera. Trate como ponte temporária e remova quando o painel
for oficialmente reapontado — ponte esquecida vira cópia redundante.

---

## Auditoria de objeto órfão exige três frentes — e busca por nome cru

**Sintoma:** você quase dropa uma tabela grande "sem uso" que alimenta um painel.

**Causa:** as referências vivem em lugares diferentes: código dos pipelines, dependências
no catálogo do banco, e **dentro dos arquivos de painel de BI**. O formato da referência
varia entre painéis do mesmo time, então buscar por delimitador dá falso negativo.

**Regra:** antes de aposentar um objeto, cheque (1) grep do nome em todos os repositórios,
(2) dependências no catálogo, (3) busca do nome **cru, como substring**, dentro dos
arquivos de painel — que são zips: extraia e busque nos membros decodificando tanto em
UTF-16LE quanto em UTF-8. Cuidado com nomes que são prefixo de outros.

---

## Coluna "que falta no banco" pode ser medida calculada no modelo de BI

**Sintoma:** o painel referencia dezenas de campos que não existem em nenhuma tabela.

**Regra:** ao reconectar um painel, separe as referências em "colunas físicas" (precisam
existir) e "medidas" (não exigem mudança). Reduz drasticamente o trabalho. Risco associado:
**regra de negócio que só existe dentro do arquivo do painel se perde junto com ele** — o
ideal é materializá-la no pipeline.

---

## "Quebrou o agrupar por mês" quase nunca é o banco

**Sintoma:** depois de repontar a fonte, os totais estão certos mas a quebra por período
some ou repete o mesmo valor.

**Causa:** a relação entre a dimensão de tempo e a tabela de fatos se perdeu no modelo.

**Regra:** após qualquer repontamento, revise as relações do modelo antes de investigar as
consultas.

---

## Dados legados vêm com espaço à esquerda e NBSP à direita

**Sintoma:** um JOIN que "deveria casar 100%" casa 0% — ou pior, casa 60% e você acha que
é problema de cobertura.

**Causa:** a camada nova preserva o valor cru do sistema de origem (padding e `\xa0`, o
espaço não separável vindo de HTML), enquanto a tabela antiga aplicava `TRIM`.

**Regra:** normalize os dois lados — substitua `chr(160)` por espaço comum **antes** do
trim: `btrim(replace(col, chr(160), ' '))`. Vale para qualquer dado que passou por HTML,
planilha ou copiar-colar.

**Como verificar:** compare a taxa de casamento antes e depois; diferenças residuais de
fração de 1% costumam ser frescor de dado, não encoding.

---

## Timestamp com offset precisa ser normalizado antes de comparar com hora de pipeline

**Sintoma:** a tabela parece desatualizada em algumas horas, ou "atualizada no futuro".

**Causa:** o banco grava com o offset local do servidor e o orquestrador de CI registra em
UTC.

**Regra:** compare sempre em UTC, convertendo explicitamente. Não faça aritmética de fuso
"na mão". Identifique a coluna de frescor de cada tabela antes de auditar — ela varia de
nome, e há tabelas que simplesmente não têm nenhuma.

---

## Migração de procedure legada: transforme por regex, valide, execute em transação

**Sintoma:** reescrever à mão uma procedure enorme introduz erros silenciosos de
mapeamento.

**Regra:** método que funcionou — (1) transformar o SQL programaticamente, remapeando os
identificadores por regex; (2) **validar que toda coluna referenciada existe na origem
antes de executar qualquer coisa**; (3) executar dentro de **uma transação**, com asserções
de sanidade (contagem esperada, unicidade da chave, distribuição dos ramos de `CASE`), e
dar `COMMIT` só se tudo passar; (4) guardar o SQL gerado como artefato.

Detalhe de ambiente legado: procedures que fazem `TRUNCATE` + `INSERT` sem `CREATE TABLE`
têm a estrutura implícita no `INSERT` — a tipagem precisa ser decidida explicitamente na
recriação.

---

## Renomeação inconsistente entre tabelas do mesmo pipeline

**Sintoma:** a migração falha no meio, depois de já ter alterado dados.

**Causa:** o pipeline novo normalizou os nomes de coluna em algumas tabelas e preservou os
antigos em outras. Não há regra global; há regra por tabela.

**Regra:** escreva a normalização como função mecânica e explícita, aplique
programaticamente, e valide a existência de todas as colunas **antes** de executar. Falhar
cedo, com a lista de colunas ausentes, custa minutos; falhar no meio custa horas.

---

## `DISTINCT ON` substitui o `ROW_NUMBER` de origens já deduplicadas

**Causa:** o `ROW_NUMBER() OVER (PARTITION BY ...) = 1` existia porque a origem antiga
trazia duplicatas; a origem nova já entrega uma linha por chave.

**Regra:** ao migrar, cheque se a duplicação ainda existe. Se existir,
`DISTINCT ON (chave) ... ORDER BY chave, criterio` é mais curto e barato. Se não existir,
remova a lógica — mas mantenha uma asserção de unicidade no build.

---

## Valor fixo por período cabe numa tabela por-item se for distribuído

**Sintoma:** um valor definido por dia precisa aparecer numa tabela cuja granularidade é
por documento — e qualquer soma dá o valor multiplicado pela quantidade de linhas.

**Regra:** divida o valor do período pela quantidade de linhas daquele período e grave a
fração em cada linha. A soma bate exatamente em **qualquer** agrupamento. Guarde o valor
original numa tabela de parâmetro com coluna de vigência, aplicando por data o registro
vigente mais recente — assim mudanças históricas não reescrevem o passado.

**Como verificar:** `SUM(coluna_distribuida)` agrupado por período deve dar exatamente o
valor do parâmetro vezes o número de períodos.

---

## Cotas e limites são regra de servidor, com reset preguiçoso

**Causa:** controle apenas na UI, ou job agendado de reset que falha silenciosamente.

**Regra:** guarde contador + período no registro do usuário e faça o reset **na leitura** —
se o período mudou, zera. O plano precisa ser argumento obrigatório da função de consumo:
assumir o plano mais generoso por omissão é o erro caro; assumir o mais restrito é o padrão
seguro. Devolva 429/402 com o motivo e trate no cliente oferecendo o caminho degradado.
