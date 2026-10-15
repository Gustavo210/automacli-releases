---
url: https://gustavo210.github.io/automacli-releases/comandos.md
---

# Comandos

Todo comando aceita `--help`.

## Tasks

### `automacli init [DIR]`

Cria o perfil de um repositório a partir dos seus arquivos compose. Rode dentro de um checkout do repositório.

| Flag | Descrição |
|---|---|
| `--local` | Grava `.automacli.yml` na raiz do repositório em vez de `~/.config/automacli/repos/`. |
| `-y`, `--yes` | Aceita as sugestões sem perguntar. |
| `--force` | Sobrescreve um perfil existente. |
| `-g`, `--global` | Só a configuração da máquina, sem perfil de repositório. |

Enquanto o `config.json` não tiver token do GitHub (ou com `--global`), o `init` abre um assistente para a configuração da máquina: diretório das tasks, token do GitHub, estratégia de clone (entre as que a máquina suporta) e notificações. Grava em `~/.config/automacli/config.json`.

Para o repositório, o `init` pergunta quais arquivos compose formam a stack, quais serviços são compartilhados (pré-seleciona bancos, caches e filas pela imagem ou pelo nome do serviço) e, no Linux, se o isolamento por systemd deve ser ligado. Inclui `.env` em `rewrite` quando existe e `Makefile` em `compose.wireInto` quando ele existe e chama `docker compose`.

### `automacli doctor`

Mostra o que a máquina oferece (git, Docker e Compose; btrfs, reflink, systemd, journal, notificações, clipboard) e qual estratégia de clone será usada. Veja [Requisitos e ambiente](requisitos.md).

| Flag | Descrição |
|---|---|
| `-r`, `--repo` | `owner/repo`: avalia a estratégia com o repo base desse repositório. |
| `-j`, `--json` | Saída em JSON. |

### `automacli base convert OWNER/REPO`

Converte o repo base em subvolume btrfs, para que os próximos clones usem snapshot (instantâneo). Aceita também um caminho. Não gasta disco e mantém o compartilhamento com as tasks existentes; leva cerca de um minuto. `-y`, `--yes` pula a confirmação.

### `automacli base update [OWNER/REPO]`

Atualiza o repo base agora: `git pull --ff-only` da branch principal quando ela avançou, depois `baseRepo.regenerate` (se `autoUpdate.regenerate`), `baseRepo.autoUpdate.test` e, por último, `baseRepo.warmup` (se `autoUpdate.warmup`). `--force` regenera e testa mesmo sem commits novos. Sai com erro se um passo obrigatório falhar, mostrando as últimas linhas da saída; uma falha no pré-aquecimento só é registrada. Não mexe num repo base com alterações locais fora dos templates, nem num que outro processo esteja atualizando (`.automacli-update.lock` com menos de 2 h); nesses casos só avisa.

### `automacli base warmup [OWNER/REPO]`

Roda só os comandos de `baseRepo.warmup` no repo base, sem pull, regeneração nem testes. Serve para aquecer caches na hora (depois de ligar o perfil) e para medir quanto tempo o pré-aquecimento leva. Sai com erro se algum comando falhar.

### `automacli base status`

Últimas 5 atualizações (automáticas e manuais) de cada repo base: quando, quantos commits, quantos passos, duração e em qual comando falhou.

### `automacli profile`

Mostra o perfil em vigor: de qual arquivo veio, os valores resolvidos e a tabela de portas.

| Flag | Descrição |
|---|---|
| `-d`, `--dir` | Diretório do repositório ou da task. Padrão: diretório atual. |
| `-s`, `--slot` | Slot usado na tabela de portas. Padrão: o do `.slot`, senão 1. |

```text
$ automacli profile --slot 2
Repositório: minha-org/minha-api
Perfil:      /home/voce/.config/automacli/repos/minha-org-minha-api.yml
Projeto:     minha-api-slot2 (slot 2)
...
SERVIÇO  ORIGINAL  SLOT 2
api      4000      4200
db       5432      5432  (compartilhado)
```

### `automacli clone BRANCH -f OWNER/REPO`

Cria uma task para a branch. Se a branch não existe no remoto, é criada localmente a partir da branch principal.

| Flag | Descrição |
|---|---|
| `-f`, `--filter` | Repositório no formato `owner/repo`. Obrigatório. |
| `-d`, `--directory` | Diretório onde criar a task. Padrão: `clonePath` da configuração. |
| `-s`, `--slot` | Força um número de slot. `0` mantém portas e projeto originais. |
| `--no-slot` | Não isola: sem override, sem reescrita, sem `.slot`. |
| `--strategy` | `auto`, `snapshot`, `reflink`, `git-reference` ou `git`. Padrão: `cloneStrategy` do config. Veja [como as tasks são criadas](requisitos.md#como-as-tasks-sao-criadas). |
| `--no-reflink` | Atalho para `--strategy git`. |
| `--base-update` | O que fazer com um repo base atrasado, sem perguntar: `regen` (pull e regenerar artefatos), `pull` (só o `git pull`) ou `skip` (usar como está). |
| `--create-base` / `--no-create-base` | Criar ou não o repo base quando ele não existe, sem perguntar. Sem base, o clone é por `git`. |
| `--replace` | Se a task já existe, apaga e recria sem perguntar. |
| `-y`, `--yes` | Sem prompts, com as respostas padrão: `--base-update regen` e `--create-base`. Não apaga task existente; para isso, `--replace`. |
| `--json` | Sem prompts; o progresso vai para o stderr e o stdout recebe só um JSON com o resultado. |

#### Sem terminal: delegar o clone a uma IA ou a um script

O clone tem três perguntas: repo base atrasado, repo base inexistente e task já existente. Cada uma tem uma flag que a
responde. Sem terminal (stdin não é um TTY), ou com `--yes` ou `--json`, nada é perguntado: um repo base atrasado é
usado como está (ou atualizado, com `--yes`), um repo base inexistente é criado, e uma task existente dá erro em vez de
ser apagada, a menos que `--replace` esteja presente.

```sh
automacli clone minha-feature -f minha-org/minha-api --json --base-update regen
```

```json
{
  "ok": true,
  "branch": "minha-feature",
  "owner": "minha-org",
  "repo": "minha-api",
  "directory": "/home/voce/tasks/task-minha-feature-minha-org-minha-api",
  "strategy": "snapshot",
  "slot": 3,
  "composeProjectName": "minha-api-slot3",
  "ports": {"api": [4300], "web": [3300]},
  "sharedServices": ["db", "redis"],
  "baseRepo": {"path": "/home/voce/tasks/task-base-minha-org-minha-api", "behindBefore": 12, "action": "regen", "update": "updated"}
}
```

`baseRepo.update` é o status da atualização (`updated`, `up-to-date`, `failed`, `skipped-dirty`, `locked`) ou `null`
quando não houve. Em erro, o stdout recebe `{"ok": false, "error": "..."}` e o código de saída é 1.

Ao final, `cd <diretório da task>` vai para a área de transferência (`wl-copy`, `xclip`, `xsel`, `pbcopy` ou `clip.exe`, o que existir); sem nenhum deles, o comando é só impresso.

O passo a passo está em [Como funciona](como-funciona.md).

### `automacli override`

Regenera o override da task atual com o estado de agora e, com ElasticMQ compartilhado, garante as filas do slot ([`queues`](perfil.md#queues)). Com `shared.host: base` (padrão), os serviços compartilhados ficam fora da stack e, se nenhum estiver no ar, o comando sobe a stack compartilhada a partir do repo base; com `shared.host: first-task`, eles só são desabilitados quando outra task já os roda. Rode antes de subir a stack.

| Flag | Descrição |
|---|---|
| `-d`, `--dir` | Diretório da task. Padrão: diretório atual. |
| `-q`, `--quiet` | Sem saída em caso de sucesso, para uso em Makefiles. |

Em um diretório sem `.slot` não faz nada.

### `automacli shared up|status|down`

Serviços compartilhados (banco, cache, DNS) rodando a partir do repo base. `status` mostra onde cada um está; `up` sobe o que falta e move para o repo base o que estiver rodando dentro de uma task; `down` para todos (pede confirmação).

| Flag | Descrição |
|---|---|
| `-r`, `--repo` | `owner/repo`. Padrão: o do diretório atual. |
| `-y`, `--yes` | Não pede confirmação no `down`. |

O `branches` recusa remover o repo base enquanto ele hospedar os compartilhados.

### `automacli branches`

Lista as tasks com tamanho e data, em uma interface interativa que permite renomear e remover.

| Flag | Descrição |
|---|---|
| `--no-tui` | Imprime uma tabela em vez da interface interativa. |
| `-s`, `--sort` | `date` (padrão), `size` ou `name`. |
| `-d`, `--remove` | Remove a task com esse nome de diretório, ou um trecho que case com um único diretório. |
| `-r`, `--rename` com `-t`, `--to` | Renomeia uma task (mesma regra de nome ou trecho). Para a stack antes. |
| `-y`, `--yes` | Não pede confirmação no `--remove`. É o que o painel e scripts usam. |

Remover uma task para a stack dela antes (`docker compose -p <projeto> down`), apaga as filas do slot num ElasticMQ compartilhado ([`queues`](perfil.md#queues)), retoma a posse de arquivos criados por containers e apaga o diretório. Em btrfs com `user_subvol_rm_allowed`, um snapshot some na hora; senão, a task é movida para `<clonePath>/.lixeira/` (um rename, instantâneo) e apagada em segundo plano, então o slot fica livre na hora e o disco volta em alguns minutos. Restos de uma remoção interrompida por reinício são apagados na próxima.

### `automacli oom`

Lista os serviços e processos das tasks que foram mortos por falta de memória, lendo o journal do sistema. Sem `journalctl`, usa o `OOMKilled` que o Docker registra, sem motivo nem slice.

| Flag | Descrição |
|---|---|
| `--since` | Período: `30min`, `2h`, `7d`, `today` ou uma data. Padrão: `24h`. |
| `-s`, `--slot` | Só as mortes de um slot. |
| `-j`, `--json` | Saída em JSON. |

```text
$ automacli oom --since 3h --slot 4
QUANDO                O QUÊ                       CAUSA   MOTIVO
05/10/2026, 17:56:57  marketplace-front (slot 4)  oomd    memory pressure for /automacli.slice being 83.39% > 60.00% for > 10s
05/10/2026, 17:57:26  necso-front (slot 4)        oomd    memory pressure for /automacli.slice/automacli-slot4.slice being 62.69% > 60.00% for > 10s

2 morte(s). Mais afetados:
    1×  marketplace-front (slot 4)
    1×  necso-front (slot 4)
```

A coluna **CAUSA** diz quem matou: `oomd` é o systemd-oomd reagindo à pressão de memória no slice; `kernel` é o OOM killer do kernel: o teto `MemoryMax` de um slice ou a memória da máquina esgotada. O **MOTIVO** mostra qual slice estourou: o do slot ou o global `automacli.slice` (todas as stacks juntas). Detalhes em [Isolamento de recursos](isolamento.md#saber-o-que-foi-morto).

## Monitor

### `automacli monitor`

Sobe o servidor de monitoramento. Veja [Monitor](monitor.md).

| Flag | Descrição |
|---|---|
| `-p`, `--port` | Porta HTTP. Padrão: `7777`. |
| `--http-only` | Só o servidor HTTP, sem a dica sobre o MCP. É o que o serviço systemd usa. |
| `--watch` / `--no-watch` | Recarrega `config.json`, `repos/*.yml` e `templates/` quando mudam. Ligado por padrão. |
| `--notify` / `--no-notify` | Liga ou desliga as notificações de desktop, vencendo o `notifications` do `config.json`. |
| `--mcp-only` | Só o servidor MCP, por stdio. |
| `--no-notify` | Desliga a notificação de desktop quando um serviço é morto por falta de memória. |

### `automacli mcp setup|config|status [CLIENTE]`

Liga o MCP do monitor a um assistente de IA. Clientes: `claude-code`, `codex`, `gemini`, `cursor`, `windsurf`,
`vscode`, `claude-desktop` ou `all` (todos os encontrados na máquina). Veja [MCP para agentes de IA](mcp.md).

| Ação | O que faz |
|---|---|
| `setup [CLIENTE]` | Registra o servidor pela CLI do cliente quando existe (`claude mcp add` etc.), senão grava o arquivo de configuração dele, preservando outros servidores. Sem cliente, pergunta entre os instalados. |
| `config [CLIENTE]` | Imprime o trecho para colar: JSON de `mcpServers` (padrão), TOML para `codex`, JSON de `servers` para `vscode`. |
| `status` | Para cada cliente: ligado, instalado sem o MCP, ou não encontrado. |

| Flag | Descrição |
|---|---|
| `--dry-run` | Mostra o comando que rodaria ou o arquivo que gravaria, sem fazer nada. |
| `--file` | Grava no arquivo de configuração mesmo quando o cliente tem CLI. |

### `automacli monitor setup`

Instala o monitor como serviço systemd do usuário, iniciado no login.

| Flag | Descrição |
|---|---|
| `-p`, `--port` | Porta HTTP. Padrão: `7777`. |
| `--status` | Mostra o estado do serviço. |
| `--remove` | Remove o serviço. |

## Utilitários

### `automacli update`

Atualiza o próprio automacli para a última release, respeitando o versionamento semântico: correções (`patch`) e
novidades compatíveis (`minor`) são aplicadas; um salto de versão maior (`major`), que pode trazer mudanças
incompatíveis, só com `--major`.

| Flag | Descrição |
|---|---|
| `-c`, `--check` | Só verifica: diz se há versão nova e de que tipo, sem instalar. |
| `--major` | Permite atualizar para uma versão maior. |
| `--json` | Resultado em JSON: `current`, `latest`, `diff`, `newer`, `installMethod`, `action`, `command`. |

O que o comando faz depende de como esta cópia foi instalada:

| Instalação | O que acontece |
|---|---|
| Executável do instalador (`~/.local/share/automacli`) | Baixa o tarball da versão nova, confere o SHA-256 pelo `SHA256SUMS` do release e troca o diretório de uma vez, como o `install.sh`. |
| Pacote da distribuição (`/opt/automacli`) | Mostra o comando do gerenciador (`apt`, `dnf`, `zypper` ou `yay`), que é quem deve atualizar. |
| Código-fonte | Mostra `git pull && pnpm install && pnpm build`. |

Além do comando, qualquer execução no terminal avisa, no máximo uma vez por dia, quando existe versão nova (a consulta
fica em cache em `~/.cache/automacli/update-check.json`). O aviso não aparece com `--json`, fora de um terminal, no
monitor, nem com `AUTOMACLI_NO_UPDATE_CHECK=1`.

### `automacli outdated`

Procura pacotes npm desatualizados no diretório atual.

| Flag | Descrição |
|---|---|
| `-d`, `--directory` | Diretório a examinar. |
| `-f`, `--filter` | Regex de nomes de pacote (ex.: `^@minha-org`). Padrão: `packageFilter` da configuração. |
| `--no-dev` | Ignora `devDependencies`. |
| `-j`, `--json` | Saída em JSON. |

### `automacli cleanup`

Apaga diretórios descartáveis (`node_modules`, `dist`, `coverage`...) a partir do diretório atual.

| Flag | Descrição |
|---|---|
| `-p`, `--patterns` | Nomes de diretório a apagar, separados por espaço. |
| `-s`, `--save <padrões>` | Só grava os padrões em `~/.cleanup_patterns`, usados nas próximas execuções, e sai sem apagar nada. Não combina com `-p` nem com `--verdaccio*`. |
| `-i`, `--interactive` / `--no-interactive` | Pergunta antes de apagar. Ligado por padrão; `--no-interactive` exige `-p`. |
| `--verdaccioAll` | Apaga todo o armazenamento do Verdaccio (`verdaccioStoragePath`). |
| `--verdaccioScope` | Apaga só os pacotes de um escopo, ex.: `@minha-org`. Faz `sudo chown` do armazenamento antes de apagar. |
| `-v`, `--verbose` | Mostra no spinner o diretório sendo removido. |

### `automacli setup [BRANCH]`

Faz `stash`, troca para a branch principal, `pull` e `stash apply` no repositório atual; com `BRANCH`, repete para essa branch. Com `all`, faz o mesmo em todos os repositórios de `setup.repos` da [configuração global](instalacao.md#configuracao-global).

A branch principal é a de `origin/HEAD` (senão `main`), salvo quando o caminho contém uma chave de `setup.branchOverrides`.
