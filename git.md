# Git

## `*.env` no `.gitignore` não pega `.env.bak`

**Sintoma:** um arquivo de backup cheio de credenciais fica versionável — ou versionado —
apesar do `.gitignore` "cobrir env".

**Causa:** o padrão casa arquivos **terminados** em `.env`. `.env.bak`, `.env.local.old` e
`.env.producao` não terminam assim.

**Regra:** use padrões que cubram sufixos: `.env*`, `*.env*`, e explicitamente `*.bak`.
Nunca confie na leitura visual do `.gitignore`.

**Como verificar:**
```bash
git check-ignore -v .env.bak      # se não imprimir regra, NÃO está ignorado
git status --ignored
```

---

## Regra genérica no `.gitignore` pode engolir uma pasta de código

**Sintoma:** a rota funciona local e responde 404 em produção; o arquivo "existe" na sua
máquina.

**Causa:** uma linha como `storage/` casa **qualquer** diretório com esse nome em qualquer
profundidade — inclusive uma pasta de rota de API.

**Regra:** ancore as regras (`/storage/`) em vez de deixá-las soltas. Depois de criar
arquivo em pasta de nome genérico, confirme que ele foi versionado.

---

## `.gitignore` com `*.json` global é armadilha silenciosa

**Sintoma:** um arquivo de configuração novo simplesmente "não aparece" no `git status`, e
ninguém percebe até o deploy quebrar.

**Causa:** padrão nascido de um susto legítimo (credenciais em JSON) resolvido com força
bruta: ignorar tudo e adicionar exceções `!package.json`, `!tsconfig.json`… A lista de
exceções nunca acompanha o projeto.

**Regra:** ignore por **nome específico** dos arquivos sensíveis (`client_secret_*.json`,
`*-service-account.json`) e mantenha um comentário dizendo por quê. Se precisar de rede de
segurança, use um hook de pré-commit com varredura de segredos — não uma regra que também
esconde arquivos legítimos.

---

## Branch local rastreando upstream antigo inventa commits "não enviados"

**Sintoma:** `git status` acusa dezenas de "commits ahead" num repositório que você sabe
estar sincronizado — e você quase reescreve o remoto por causa disso.

**Causa:** o branch local ainda rastreava um branch remoto congelado meses atrás. A
contagem é real, só que contra a referência errada.

**Como verificar:**
```bash
git rev-parse --abbrev-ref --symbolic-full-name @{u}
git log --oneline origin/main..HEAD
git branch -u origin/main          # corrige
```

---

## Antes de limpar uma máquina, varra todos os repositórios

**Sintoma:** trabalho de meses desaparece junto com o disco.

**Causa:** o que nunca virou commit não existe em lugar nenhum, e "eu acho que estava tudo
enviado" nunca é verdade em todos os repositórios.

**Regra:** script de varredura que, para cada repositório, imprime `git status --porcelain`,
`git log @{u}..HEAD`, `git stash list` e os arquivos ignorados relevantes. Resolva item a
item antes de apagar qualquer coisa. Atenção a segredos em arquivos locais não versionados
— esses vão para o gerenciador de segredos, não para o git.

---

## Repositório vazio não se detecta pelo branch padrão

**Sintoma:** rotina de clone ou backup falha em alguns repositórios, ou gera artefato
inválido.

**Causa:** APIs de hospedagem git reportam um `default_branch` (`main`) mesmo em
repositórios que nunca receberam commit.

**Regra:** use a flag explícita de repositório vazio que a API oferece; nunca infira do
branch padrão.

---

## Backup de repositório: `git bundle`, não zip da pasta

**Causa:** um bundle é um arquivo único com o histórico completo — commits, branches e
tags — restaurável com `git clone arquivo.bundle`. Zipar a pasta de trabalho guarda o
`.git` inteiro **mais** os arquivos já materializados: o mesmo conteúdo duas vezes.

**Regra:** `git clone --mirror` → `git bundle create --all` → validar → descartar o mirror.
Bundle já sai comprimido (o packfile é zlib): embrulhar em zip rende quase nada. O ganho
real de espaço vem de **escolher o que não guardar** — artefato de CI, cache e binário
grande —, não de comprimir mais forte.

**Como verificar:** `git bundle verify arquivo.bundle` antes de substituir o backup
anterior.

---

## Chave SSH por repositório vai em `core.sshCommand`

**Sintoma:** você precisa de uma deploy key específica sem afetar os demais acessos SSH da
máquina.

**Causa:** editar `~/.ssh/config` é global e, em ambientes com controle de mudanças, costuma
ser bloqueado.

**Regra:**
```bash
git config core.sshCommand "ssh -i ~/.ssh/<chave> -o IdentitiesOnly=yes"
```
Escopo local, reversível, não interfere em nada fora dali.

---

## Deploy por cópia de arquivo para diretório que não é repositório é dívida

**Sintoma:** o servidor tem uma versão que não corresponde a nenhum commit e ninguém sabe
qual.

**Regra:** se precisa ser assim, copie **apenas** de código já commitado, guarde backup
datado do estado anterior no próprio servidor, e valide o health check depois do restart.
Melhor ainda: transforme o diretório em checkout, mesmo que o deploy continue manual.

---

## Dois arquivos que só diferem em maiúsculas colidem em Windows e macOS

**Sintoma:** criar `ARQUIVO.md` falha ou sobrescreve silenciosamente porque já existe
`arquivo.md`; no CI Linux o repositório passa a ter os dois e a aplicação lê o errado.

**Causa:** NTFS e APFS são case-insensitive por padrão; ext4 não é. O Git registra a
diferença de caixa que o sistema local não enxerga.

**Regra:** padronize a caixa dos nomes de arquivo de convenção e nunca dependa de distinção
por maiúsculas. Ao exportar em lote de uma origem case-sensitive para arquivos, verifique
colisão **antes** de escrever.

**Como verificar:**
```bash
git ls-files | tr 'A-Z' 'a-z' | sort | uniq -d
```
