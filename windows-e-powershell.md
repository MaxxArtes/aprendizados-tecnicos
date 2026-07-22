# Windows e PowerShell

Quase tudo aqui é a mesma classe de problema: um comando que existe no mundo Unix existe
no PowerShell com **outro significado**, ou um padrão de código que funciona em Linux
falha no Windows por causa de permissão, codificação ou case-insensitivity.

---

## `shutil.rmtree` falha em diretório `.git` — e com `ignore_errors=True` falha em silêncio

**Sintoma:** a limpeza "roda sem erro" e mesmo assim sobram gigabytes de diretórios
temporários no disco.

**Causa:** o git grava os objetos como somente-leitura. No Windows, arquivo somente-leitura
não pode ser apagado direto — a exclusão levanta `PermissionError`. Com
`ignore_errors=True`, a exceção é engolida e o diretório fica pela metade.

**Regra:** passe um handler que ajuste a permissão e tente de novo. Nunca use
`ignore_errors=True` em limpeza que você precisa que realmente aconteça — ele transforma
falha em mentira.

```python
shutil.rmtree(p, onerror=lambda f, path, e: (os.chmod(path, stat.S_IWRITE), f(path)))
```

**Como verificar:** confirme com medição de tamanho da pasta, não com ausência de exceção.

---

## Erro não-terminante deixa a variável do loop com o valor da iteração anterior

**Sintoma:** uma edição em lote "reporta OK" para todos os itens, mas metade das tarefas
passa a apontar para o alvo errado — o alvo da última iteração que deu certo.

**Causa:** um cmdlet `New-*` que falha com erro **não-terminante** não interrompe o loop e
**não reatribui** a variável; a linha seguinte usa o valor velho como se fosse novo. No
caso real, um parâmetro vazio fez o cmdlet recusar a criação e o `Set-` seguinte gravou o
objeto da iteração anterior.

**Regra:**
1. `$var = $null` no topo de cada iteração, ou cheque sucesso antes de usar;
2. construa cada item a partir de dados **explícitos** (hashtable nome→valor), nunca a
   partir do estado do objeto que está sendo mutado;
3. não passe parâmetros com valor vazio;
4. considere `-ErrorAction Stop` dentro de `try/catch`.

**Como verificar:** depois de qualquer mutação em lote, liste as propriedades-chave de
**todos** os itens e confira uma a uma. `Write-Output "OK"` dentro do loop imprime mesmo
quando o comando anterior falhou.

---

## `curl` e `dir /s /b` não são o que você pensa

**Sintoma:** `curl -s URL` retorna um objeto de resposta em vez do corpo, ou erra em flags
válidas; `dir /s /b` devolve erro de parâmetro.

**Causa:** no PowerShell, `curl` é alias de `Invoke-WebRequest` (flags POSIX não existem) e
`dir` é alias de `Get-ChildItem` (as flags são do `cmd.exe`).

**Regra:** chame `curl.exe` explicitamente para o cURL real, e use
`Get-ChildItem -Recurse -Filter` para busca de arquivos. Em scripts multiplataforma,
prefira o executável com extensão.

**Como verificar:** `Get-Command curl | Select-Object CommandType, Source`

---

## Redirecionamento de saída gera UTF-16 ou ANSI e quebra parsers

**Sintoma:** um JSON gerado por script aparece com um byte nulo entre cada caractere e
falha em `JSON.parse` ou `jq`; ou vem com BOM inesperado.

**Causa:** `>` e `Out-File` no PowerShell padrão usam UTF-16LE; `Set-Content` e
`Add-Content` usam a codepage ANSI do sistema. Ferramentas Unix esperam UTF-8 sem BOM.

**Regra:** sempre passe `-Encoding utf8` explicitamente ao escrever arquivo que outra
ferramenta vai ler. Em CI rodando em runner Windows, declare `shell: bash` nos passos que
geram arquivos de configuração — resolve a classe inteira.

**Como verificar:** `file arquivo.json` deve dizer "UTF-8", não "UTF-16".

---

## Alimentar CLI via PowerShell insere BOM e corrompe o valor

**Sintoma:** o segredo "está configurado" mas a autenticação falha com credencial
inválida; comparar visualmente não mostra diferença.

**Causa:** redirecionamentos e cmdlets escrevem UTF-8 **com BOM** por padrão; a CLI grava o
byte extra dentro do valor.

**Regra:** use um shell POSIX para alimentar valores em CLIs, ou passe
`-Encoding utf8NoBOM`/`-NoNewline` explicitamente.

**Como verificar:** `head -c3 arquivo | xxd` — `efbbbf` é BOM.

---

## Para enviar arquivo via API, leia bytes crus

**Sintoma:** acentos e emojis chegam corrompidos no destino depois de um upload em base64.

**Causa:** `Get-Content -Raw` decodifica o arquivo usando a codificação **presumida** do
shell e devolve string; re-codificar essa string só acerta se o palpite estiver certo.

**Regra:** use `[System.IO.File]::ReadAllBytes($caminho)` e converta direto para base64. O
byte que saiu é o byte que entrou, independentemente de codificação, BOM ou locale.

---

## Ao achatar um namespace em nomes de arquivo, verifique colisão por maiúscula

**Sintoma:** você exporta N itens e recebe menos que N arquivos, sem erro. Um sobrescreveu
o outro.

**Causa:** o sistema de arquivos é case-insensitive por padrão. Origens que diferenciam
maiúsculas (URLs, APIs, Linux) podem ter dois itens que só diferem no caso.

**Regra:** antes de gravar em lote, agrupe os nomes por versão em minúsculas e verifique se
algum grupo tem mais de um. Cheque também caracteres inválidos e o comprimento total do
caminho.

**Como verificar:** conte os arquivos gerados e compare com o número de itens da origem.

---

## Disco cheio quebra a ferramenta e o erro não diz "disco cheio"

**Sintoma:** automação de navegador headless passa a falhar com `ENOSPC` e mensagens
inconsistentes.

**Causa:** perfis temporários de navegador acumulam gigabytes por execução, num disco
compartilhado com outros perfis de usuário.

**Regra:** aponte diretórios temporários para um volume com folga e limpe ao fim de cada
execução. Antes de investigar um erro estranho de I/O, cheque espaço livre.

**Como verificar:** `Get-PSDrive C | Select-Object Used,Free` como primeiro passo do
diagnóstico.

---

## Use `127.0.0.1`, não `localhost`, ao falar com serviços locais

**Sintoma:** `ECONNREFUSED` ou `curl` devolvendo código 000 contra um serviço
comprovadamente no ar.

**Causa:** `localhost` pode resolver para IPv6 (`::1`) enquanto o serviço escuta apenas em
IPv4.

**Regra:** em health check e automação, use o endereço literal IPv4. Vale também dentro de
container.
