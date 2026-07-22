# Infraestrutura, containers e rede

## Porta 25 de saída é bloqueada pelo provedor de nuvem

**Sintoma:** a fila do servidor de e-mail cresce com `connect to ...:25: Connection timed
out`. Enviar de dentro do servidor não funciona, mas o serviço de e-mail parece saudável.

**Causa:** a maioria dos provedores de VPS bloqueia a porta 25 de saída por padrão
(antispam). O gargalo é a entrega final, não a autenticação — por isso "logar com uma conta
válida" não resolve.

**Regra:** configure relay para um serviço de e-mail transacional na porta 587 com SASL, ou
use a API HTTPS do provedor. Pedir desbloqueio da 25 ainda deixa a entregabilidade ruim por
reputação de IP. Verifique também se a conta está fora do modo sandbox antes de projetar o
fluxo.

**Como verificar:** compare `nc -vz -w 5 <mx-externo> 25` com `nc -vz -w 5 <relay> 587`.

---

## Entregabilidade exige DKIM + **um único** SPF + DMARC

**Sintoma:** e-mails caem em spam, ou a verificação de domínio não conclui.

**Causa:** configuração parcial, ou dois registros SPF separados no apex — o que invalida os
dois.

**Regra:** publique os CNAMEs de DKIM do provedor, **mescle** todos os `include:` num só
registro SPF, e publique DMARC. Leia a zona antes de escrever para não derrubar registros
existentes.

**Como verificar:** `dig TXT dominio` deve mostrar exatamente um `v=spf1`.

---

## Domínio sem registro MX não recebe e-mail

**Regra:** ao prometer um endereço em domínio próprio, valide o MX antes de divulgar o
endereço em cadastros e materiais. `dig +short MX exemplo.tld`.

---

## SSL curinga em subdomínio geralmente exige delegar a subzona por NS

**Sintoma:** subdomínios dinâmicos sobem sem certificado.

**Causa:** a plataforma só emite certificado curinga se ela controlar o DNS daquele nível.

**Regra:** crie um registro NS para a subzona apontando para os servidores da plataforma.
**Apague antes o registro A ou curinga anterior daquele nome** — ele conflita e a API recusa
com erro do tipo "abaixo de delegação".

---

## Update de conjunto de registros DNS pode não ser `PUT`

**Sintoma:** `PUT` no recurso do registro devolve 422 sem explicação útil.

**Causa:** algumas APIs de DNS expõem a substituição como endpoint de **ação**, não como
update REST.

**Regra:** leia a referência do provedor antes de assumir semântica REST. E sempre leia a
zona antes de escrever, para não apagar registros de outros serviços.

---

## `docker inspect` dizendo "Running" não é health check

**Sintoma:** o pipeline declara sucesso e o site está fora do ar.

**Causa:** um container em restart loop aparece como "Running" durante cada tentativa.
Checar poucos segundos depois do start pega exatamente essa janela.

**Regra:** valide com requisição HTTP real esperando 200, contra endereço IPv4 literal, com
algumas tentativas espaçadas.

---

## Nunca chame o CLI do ORM via `npx` no entrypoint de um container

**Sintoma:** o container entra em restart loop em produção; local funciona. Ou: roda por
meses e um dia quebra com "flag desconhecida".

**Causa:** o CLI não está no runtime, então `npx` baixa a **última major** publicada — que
pode ter removido a flag que seu comando usa.

**Regra:** copie o CLI para a imagem e invoque o binário local por caminho explícito. Fixe
versões no manifesto. Regra geral: `npx` em entrypoint é dependência de rede mais versão
flutuante.

**Como verificar:** `docker logs --tail 50 <container>` mostra o mesmo erro em ciclo.

---

## Módulo nativo precisa de toolchain de compilação na imagem

**Sintoma:** a instalação de dependências falha em ambiente limpo com erro de compilação,
mesmo funcionando na máquina do desenvolvedor.

**Causa:** pacotes que falam com o SO em baixo nível não têm binário pré-compilado para
toda combinação de plataforma e versão, e caem para compilar do fonte.

**Regra:** instale compilador, `make` e Python na imagem **antes** do install. Melhor
ainda: pré-construa uma imagem base com as dependências pesadas já instaladas e rebaseie a
aplicação nela — corta ordens de magnitude do tempo de build.

---

## Swap resolve o congelamento de build, mas é mitigação

**Sintoma:** build na VPS trava a máquina, derruba o SSH e mata a aplicação.

**Causa:** build de aplicação moderna é o pico de RAM do ciclo de vida do projeto, e VPS
baratas dimensionam RAM para o regime permanente.

**Regra:** crie swap como rede de segurança — mas entenda que ele troca "morrer" por "ficar
muito lento", ainda estressando a CPU que deveria atender clientes. A solução real é não
compilar na máquina de produção.

---

## Estado de job só em memória significa que restart destrói trabalho

**Sintoma:** um deploy ou restart apaga jobs em andamento sem rastro; o usuário fica
esperando para sempre.

**Regra:** persista estado de job (fila, progresso, resultado) fora do processo. Enquanto
isso não existir, documente em local visível "não reiniciar com job ativo" e verifique antes
de reiniciar.

---

## Métrica que a ferramenta não expõe exige um segundo comando

**Sintoma:** um campo aparece sempre zerado no painel de monitoramento de containers.

**Causa:** o comando de estatísticas em tempo real não inclui uso de disco; é uma métrica
cara, coletada por outro caminho.

**Regra:** verifique qual comando expõe a métrica e mescle os dois resultados por
identificador, em vez de estimar.

---

## Saída de CLI vem com unidade grudada e zera gráfico

**Sintoma:** barras do gráfico não renderizam altura nenhuma; os valores aparecem certos
nas tabelas.

**Causa:** ferramentas de linha de comando devolvem strings formatadas (`12.5%`, `1.2GiB`).
Bibliotecas de gráfico precisam de número; string vira `NaN` ou zero.

**Regra:** normalize na camada de API — remova símbolo, converta para uma base única e
devolva número puro. A unidade volta só no rótulo da interface.

---

## Antes de debugar, confirme qual serviço realmente guarda o dado

**Sintoma:** horas perdidas olhando um painel vazio, concluindo que "nada foi salvo".

**Causa:** arquiteturas multi-provedor separam armazenamento de arquivos, processamento e
metadados em serviços diferentes.

**Regra:** mantenha um diagrama de uma tela por sistema dizendo qual provedor guarda o quê.
Ao investigar, primeiro identifique o dono do dado.

---

## Quando "tudo ficou estranho de uma vez", vá aos logs antes de caçar bug

**Sintoma:** vários registros mudam ao mesmo tempo e a hipótese natural é escrita em massa
defeituosa.

**Causa:** foi clique humano. Em aplicações com ações de servidor, cada clique aparece como
uma requisição na rota da própria página, com timestamp.

**Regra:** consulte os logs de runtime primeiro. Três requisições no mesmo minuto provam a
origem em um minuto e evitam horas depurando código correto. Retenção curta de log é motivo
real para manter observabilidade paga.

---

## Meça antes de escolher em qual servidor rodar

**Sintoma:** você monta um ambiente novo no servidor que já está sufocado de memória,
arriscando derrubar produção.

**Causa:** o documento de contexto citava só um servidor — o mais antigo — e a existência de
outro, mais folgado, não estava documentada.

**Regra:** antes de recomendar onde rodar qualquer coisa nova, liste os hosts existentes e
meça memória e disco em **todos** os candidatos. Depois, atualize o documento: a lacuna que
enganou você vai enganar o próximo.

---

## Franquia gratuita pode valer só nos 12 primeiros meses da conta

**Sintoma:** a conta do fornecedor vem maior do que a calculadora interna previa.

**Causa:** muitos "X mil requisições grátis por mês" são benefício de conta nova, não
permanente — e frequentemente não cobrem o armazenamento associado.

**Regra:** ao repassar custo, use **zero** de franquia por padrão e deixe configurável.
Errar cobrando a menos é pior que cobrar a mais. E meça o custo em vez de estimar: registre
cada chamada paga como evento de uso.
