# Arquitetura, modelagem e decisões de produto

## Prove a arquitetura com uma fatia vertical antes de escalar

**Sintoma:** meses construindo camadas e, no final, o gargalo real — acesso à rede interna,
autenticação, latência — inviabiliza o desenho.

**Causa:** construir camada por camada só valida a integração no fim.

**Regra:** primeira entrega é uma fatia de ponta a ponta com um único caso: uma tabela
agregada → um endpoint → um gráfico → uma incorporação. Ela exercita exatamente as
restrições que podem matar o projeto. Quando a fonte de dados vive em rede privada, o padrão
é expor **só a API** por túnel com HTTPS — nunca o banco — e servir de tabelas pré-calculadas
em vez de consultar o modelo cru a cada requisição.

---

## O registro nasce antes do arquivo, e a reconciliação é obrigatória

**Sintoma:** em lotes longos, quando a aba é fechada ou a rede cai, ficam arquivos no
storage sem registro no banco: invisíveis, sem processamento, ocupando espaço para sempre.
E registros sem arquivo, pendentes eternamente.

**Causa:** no fluxo com URL assinada, o byte vai do navegador direto para o storage e o
backend só é avisado **depois**. Esse "depois" pode nunca acontecer. A operação está quebrada
em várias etapas de rede que não compartilham transação.

**Regra:** inverta — ao assinar a URL, já grave o registro com marcador de "não pronto"; a
listagem pública filtra por "pronto". Uma rotina de reconciliação varre os pendentes: se o
objeto existe no storage, conclui e publica; se não existe, descarta o registro. Assim
nenhum estado intermediário é irrecuperável, em qualquer ponto de falha.

**Como verificar:** suba um lote e mate a aba no meio; a reconciliação seguinte deve zerar
os pendentes sem intervenção manual.

---

## Reprocessamento idempotente: proteja só o passo caro e não-idempotente

**Sintoma:** a rotina de conclusão roda duas vezes e a mesma pessoa aparece duplicada na
busca, ou a fatura é contada duas vezes.

**Causa:** regerar derivados é inofensivo (sobrescreve). Já chamadas a serviços externos
pagos que **acumulam estado** — indexação, cobrança, envio — não são idempotentes.

**Regra:** torne a função inteira segura para repetir, mas coloque uma guarda explícita
imediatamente antes do passo com efeito acumulativo: "só executa se ainda não houver
registro do resultado". A guarda vive no banco, não na memória do processo.

---

## Valor cobrado se recalcula no servidor; total vindo do cliente é sugestão

**Causa:** é natural o front já ter o total calculado para exibir e mandá-lo junto no envio.

**Regra:** uma única função de cálculo, no servidor, usada tanto pela tela quanto pela
geração da cobrança. O payload do cliente traz apenas parâmetros — o quê, quanto —, nunca o
resultado financeiro.

---

## Status de pedido é um extrato: cancelar e estornar não são a mesma coisa

**Sintoma:** relatórios de faturamento divergem do dinheiro real, e o histórico "mente".

**Causa:** um único estado "cancelado" usado tanto para desistência antes do pagamento
quanto para devolução depois.

**Regra:** modele transições explícitas (`pendente → pago | cancelado`, `pago → estornado`)
numa função `podeTransicionar`; derive de funções puras quem **segura** estoque, quem
**conta** como receita e quais botões aparecem. Não permita "cancelar" um pedido pago.
Estornar deve devolver o dinheiro **antes** de mudar o status: se o provedor recusar, o
pedido continua pago e o operador lê o motivo. E limpar histórico não é desfazer vendas —
não devolva estoque nessa operação.

---

## Ligar um recurso não pode desligar o negócio

**Sintoma:** o usuário clicou num botão novo "só para ver o que era" e tirou a própria
operação do ar por uma hora.

**Causa:** o botão "ativar controle de estoque" gravava **zero**, e zero significa esgotado,
que significa produto oculto na loja. Sem confirmação e sem aviso.

**Regra:** ativar um recurso nunca deve gravar um valor com efeito destrutivo. Peça o valor,
só grave quando informado, e avise em texto claro qual é a consequência visível.

---

## CRUD sem "update" força um fluxo destrutivo

**Sintoma:** para mudar o papel de um usuário só existia apagar e recriar — o que obriga a
combinar uma senha nova com a pessoa.

**Regra:** ao modelar permissões, garanta a operação de alteração desde o começo, com
travas: o papel de dono nunca é oferecido na lista (um segundo dono pode apagar o primeiro)
e ninguém altera o próprio papel.

---

## "Papel fixo" costuma ser palpite sobre a realidade do cliente

**Sintoma:** requisito de "criar conta de funcionário e marcar quais telas ele vê".

**Causa:** confundir navegação com autorização, e presumir que os papéis do seu modelo
correspondem à realidade daquele negócio.

**Regra:** lista de áreas liberadas costuma modelar melhor que papéis fixos. Identifique de
antemão as travas **não-delegáveis** — assinatura, credenciais de pagamento, domínio, gestão
de usuários e operações que movem dinheiro — e o requisito estrutural: se hoje existe relação
1:1 entre conta e organização, isso vira tabela de vínculo e refatoração de **toda** a
verificação de posse.

---

## Integração com provedor externo: Adapter + falha segura quando a chave não existe

**Sintoma:** trocar de fornecedor exige mexer em dezenas de arquivos, e um deploy com a
chave faltando expõe a feature quebrada para todo mundo.

**Regra:** defina um contrato e um adapter por fornecedor, escolhidos por uma factory —
trocar de provedor vira um arquivo novo. A factory retorna `null` quando a credencial não
está configurada, e a UI mostra "em breve" em vez de tentar. Complemente com allowlist por
variável de ambiente para lançamento gradual. Lembre que cada troca de provedor de OAuth
obriga todos os usuários a reconectar — por isso migrar cedo, enquanto a base é pequena.

---

## Dados de demonstração vazam para sitemap e listagens públicas

**Regra:** marque registros de demo com uma flag dedicada num único módulo e aplique esse
filtro em toda superfície pública — listagens, busca, sitemap, feeds. Um filtro só,
reutilizado, evita esquecer um lugar.

---

## Print de comprovante não é prova de pagamento

**Sintoma:** pedido de feature "o cliente manda o print e o sistema confirma".

**Causa:** print é trivialmente falsificável — é o golpe clássico de balcão. IA lendo valor
e data pega falsificação grosseira, e só.

**Regra:** quem prova é o extrato do provedor. Se a cobrança passa por um adquirente, deixe
a confirmação automática ser a fonte da verdade; se é chave estática, a confirmação é manual
e isso deve estar claro na UI.

---

## Recibo deve ser gerado ao vivo e não conter contato do cliente

**Sintoma:** um comprovante compartilhado continua dizendo "pago" depois de um estorno.

**Regra:** gere o comprovante a partir do estado atual — comprovante congelado após estorno
é documento falso circulando. Não inclua telefone nem endereço: o link é reencaminhável, e o
comprovante prova a compra, não expõe o comprador.

---

## Busca por similaridade tem dois cortes silenciosos

**Sintoma:** a busca "não encontra tudo" mesmo com os itens indexados.

**Causa:** o limiar de confiança alto derruba correspondências legítimas em condições ruins,
e o limite máximo de resultados corta o resto sem avisar.

**Regra:** ajuste limiar e teto conscientemente e registre-os como configuração, não como
número mágico. Antes de culpar a busca, confirme que a indexação terminou — resultado vazio
pode ser recall ruim **ou** base incompleta, e são diagnósticos opostos.

---

## Automação de mensageiro por biblioteca não-oficial bane o número do cliente

**Regra:** ou faz pelo caminho oficial (conta comercial verificada, template aprovado, custo
por conversa), ou usa outro canal. Nunca coloque o número do cliente em risco. Antes disso,
mapeie o buraco real: às vezes só um subconjunto dos casos precisa de aviso.

---

## APIs de autopreenchimento de plataformas de design exigem plano corporativo

**Sintoma:** plano de produto baseado em "preencher templates lindos automaticamente" morre
na integração.

**Causa:** essas APIs frequentemente exigem tier corporativo, autenticação por usuário final
e revisão do app antes de publicar.

**Regra:** valide tier comercial, modelo de autenticação e processo de aprovação **antes** de
desenhar o produto em cima. Alternativa controlada: renderizar HTML/CSS próprio e converter
para PDF.

---

## Só use base de código com licença permissiva

**Causa:** muitos repositórios populares são copyleft, não-comercial, ou simplesmente **sem
licença** — e sem licença significa "todos os direitos reservados", não "livre".

**Regra:** como base de código, só MIT/Apache/BSD/CC0, confirmada pela API de licença do
provedor — não pelo README. Qualquer outra coisa entra apenas como referência visual, com a
implementação recriada.

---

## Feature que depende de credencial de produção precisa de plano de diagnóstico em produção

**Sintoma:** um caminho de código nunca foi realmente exercitado porque, localmente, ele
para antes com erro de configuração.

**Regra:** assuma explicitamente o que só pode ser validado em produção e planeje como
observar: log estruturado, endpoint de diagnóstico protegido, ou um painel temporário
removido no commit seguinte. Quando o bug só reproduz em dispositivo real, isso resolve em
uma tentativa o que várias hipóteses no escuro não resolvem.

---

## Resumo para stakeholder é escrito em resultado, não em implementação

**Sintoma:** o relato técnico do que foi feito não é compreendido por quem precisa aprovar.

**Causa:** nome de tabela, contagem de linhas e jargão de código não significam nada para
quem é da área mas não é mão-na-massa.

**Regra:** duas a quatro frases dizendo o que passou a funcionar, o que foi corrigido, o que
falta e de quem depende. Evite nome de tabela, de repositório e número de linhas.

> Ruim: "criadas as duas dimensões, 652 mil linhas."
> Bom: "as bases de clientes e de fretes do relatório foram reconstruídas no novo ambiente e
> já estão prontas para uso."

Prepare esse texto **junto** com a entrega técnica — quem executou é quem consegue traduzir.
