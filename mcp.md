---
url: https://gustavo210.github.io/automacli-releases/mcp.md
---

# MCP para agentes de IA

O monitor do automacli fala [MCP](https://modelcontextprotocol.io) (Model Context Protocol). Ligado a um assistente
como Claude Code, Cursor ou Codex, ele dá ao modelo o contexto do que está rodando na máquina: quais branches estão
clonadas, em que slot, se a stack está no ar, se o PR foi mesclado, o que cada task mudou em relação ao repo base,
quanto disco ela ocupa de verdade e se os serviços compartilhados estão de pé. Com isso o agente consegue responder
"posso apagar essa task?", liberar slots, e diagnosticar e corrigir problemas sem que você precise traduzir o estado
da máquina para ele.

## Ligar em um comando

Um comando por cliente. Ele usa a CLI do cliente quando existe (`claude mcp add`, `codex mcp add`, `gemini mcp add`,
`code --add-mcp`) e, nos outros, grava o arquivo de configuração sem mexer no que já está lá:

::: code-group

```sh [Claude Code]
automacli mcp setup claude-code
```

```sh [Codex CLI]
automacli mcp setup codex
```

```sh [Gemini CLI]
automacli mcp setup gemini
```

```sh [Cursor]
automacli mcp setup cursor
```

```sh [Windsurf]
automacli mcp setup windsurf
```

```sh [VS Code]
automacli mcp setup vscode
```

```sh [Claude Desktop]
automacli mcp setup claude-desktop
```

```sh [Todos]
automacli mcp setup all        # todos os clientes encontrados na máquina
```

:::

Depois, reinicie o cliente (ou a sessão) e pergunte: "o que o automacli está gerenciando?".

```sh
automacli mcp setup            # sem argumento: escolhe entre os clientes instalados
automacli mcp status           # onde já está ligado
automacli mcp setup cursor --dry-run   # mostra o que faria, sem gravar
```

### À mão

O servidor é `automacli monitor --mcp-only`, por stdio. `automacli mcp config` imprime o trecho pronto para colar
(`automacli mcp config codex` no formato TOML do Codex, `automacli mcp config vscode` no formato do VS Code):

```json
{
  "mcpServers": {
    "automacli": {"command": "automacli", "args": ["monitor", "--mcp-only"]}
  }
}
```

O `setup` nunca depende do `PATH`, porque clientes gráficos como Cursor e Claude Desktop, e o Claude Code aberto por
eles, não herdam o `PATH` do seu shell. Instalado pelo release, ele grava o caminho absoluto do executável
(`~/.local/share/automacli/bin/automacli` ou `/opt/automacli/bin/automacli`), que traz o próprio Node. Rodando do
código-fonte (`npm link`), grava o Node em uso e o `bin/run.js` absolutos.

O servidor usa o mesmo `config.json` e os mesmos perfis do resto do automacli. O token do GitHub é opcional, mas sem
ele o ciclo de vida não enxerga PRs e só reconhece uma task como terminada quando todos os commits dela já estão na
branch principal.

::: tip Instruções embutidas
O servidor envia ao cliente, no handshake, um texto de instruções com os conceitos (task, slot, repo base, ciclo de
vida, disco real × aparente), a ordem recomendada de uso e a regra principal: **toda tarefa nova começa com
`create_task`**. Quando você pede "começa a tarefa X" ou "implementa Y na branch Z", o agente cria a task dessa branch
antes de tocar em código, e nunca desenvolve no repo base nem na task de outra branch. A maioria dos clientes mostra
as instruções ao modelo automaticamente; você não precisa explicar o automacli no seu prompt.
:::

## O que o agente passa a saber

| Pergunta | De onde vem a resposta |
|---|---|
| Quais branches estão clonadas e em que slot? | `list_tasks`: uma entrada por diretório `task-*`, com `slot`, `pr`, `stack` |
| Quais estão rodando agora? | `list_tasks` com `filter: running` (containers via `docker compose ps`) |
| Quais já foram mescladas ou fechadas? | `lifecycle.state` de cada task: `merged`, `closed`, `abandoned` |
| Posso apagar esta task para liberar o slot? | `lifecycle.canDelete`, com `blockers` (o que impede) e `warnings` |
| O que ela mudou em relação ao repo base? | `get_task_changes`: commits só dela, arquivos tocados, não commitado, não enviado |
| Quanto disco ela ocupa de verdade? | `storage.exclusiveBytes` (o que apagar devolve) × `apparentBytes` (o que `du` mostra) |
| Os serviços compartilhados estão de pé? | `get_shared_services`: cada serviço e o projeto compose onde roda |
| O repo base está atualizado? | `get_base_repos`: commits atrás da principal, alterações locais, última atualização |
| O que está errado e como resolver? | `diagnose`: achados com severidade e comando (ou ferramenta MCP) de correção |
| Cria uma task para esta branch? | `create_task`: o `automacli clone` sem prompts, devolvendo diretório, slot e portas |

## Ciclo de vida de uma task

Cada task recebe um `lifecycle` calculado cruzando o PR no GitHub com o git local:

| `state` | Quando | Pode apagar? |
|---|---|---|
| `base` | É o repo base (`task-base-<owner>-<repo>`), origem das outras tasks e host dos compartilhados. | Nunca. |
| `merged` | PR mesclado, ou, sem PR, todos os commits da task já estão na branch principal. | Sim, se não houver bloqueio. |
| `closed` | PR fechado sem merge. | Sim, com aviso para confirmar que o trabalho não volta. |
| `abandoned` | Nunca teve PR nem branch remota, e o último commit tem mais de 30 dias. | Não: os commits só existem nessa máquina. |
| `in-review` | PR aberto, em rascunho ou aprovado aguardando merge. | Não. |
| `in-progress` | Sem PR; branch enviada ou com commits locais. | Não. |
| `unknown` | Sem dados do git (diretório sem repositório, por exemplo). | Não. |

`canDelete` só é `true` quando o estado é `merged`, `closed` ou `abandoned` **e** não há bloqueio. São bloqueios:
arquivos não commitados, commits não enviados para a branch remota e commits que não estão nem na principal nem no
remoto. Avisos (`warnings`) não impedem: stack no ar (será parada ao apagar), PR fechado sem merge.

`wouldFree` diz o que apagar libera: o número do slot e os bytes exclusivos da task, quando o disco já foi medido.

::: warning O PR vence o git
Um PR mesclado por squash deixa a branch com commits que não são ancestrais da principal, então o git sozinho diria
"não mesclada". Por isso o estado vem do PR quando há token, e o git é o critério só sem PR. Sem token, uma task
mesclada por squash fica em `in-progress` até você apagá-la à mão: é o lado conservador.
:::

## Ferramentas

### Leitura

| Ferramenta | Parâmetros | O que devolve |
|---|---|---|
| `get_overview` | — | Repositórios, repo base, slots em uso, serviços compartilhados, tasks por estado, quais podem ser apagadas, disco real × aparente. Comece por aqui. |
| `list_tasks` | `filter`: `all`, `running`, `finished`, `deletable`, `in-progress`, `in-review`, `base`; `repo`: `owner/repo` | Uma entrada por task: slot, `lifecycle` com resumo em uma linha, PR com checks, containers (inclusive os com problema), mudanças em relação à principal, disco, URLs. |
| `get_task` | `task`: diretório, branch ou trecho único | O mesmo de `list_tasks` para uma task, mais as mortes por memória das últimas 24 h. |
| `get_task_changes` | `task`; `includeFiles` (padrão `true`) | `commits` (até 50), `changedFiles` (até 200), `uncommitted`, `unpushedCommits`, `aheadOfBase`, `behindBase`, `mergedIntoBase`, mais o `lifecycle`. |
| `get_shared_services` | — | Por repositório: cada serviço compartilhado e o projeto compose onde roda; `healthy: false` quando parado ou rodando dentro de uma task. |
| `get_storage` | — | Relatório do btrfs: totais aparente, exclusivo, real, economia do reflink, e por task o `exclusiveBytes`, ordenado do maior. Fora de btrfs, só o aparente. |
| `get_base_repos` | — | Repo base de cada repositório: branch principal, commits atrás do remoto, alterações locais, `autoUpdate` do perfil, resultado da última atualização. |
| `diagnose` | `task` (opcional) | Achados ordenados por severidade, cada um com `code`, `message` e `fix` (comando ou ferramenta). |
| `get_queues` | `task` (opcional) | Filas SQS de cada task, com nome e estado (`waiting`, `processing`, `delayed`, `idle`), os três contadores e a URL da interface web do ElasticMQ (`ui`). Com ElasticMQ compartilhado, cada slot vê só as suas, pelo sufixo. |
| `get_oom_kills` | `since` (padrão `24h`) | Serviços e processos mortos por falta de memória, como `automacli oom`. |

### Ação

| Ferramenta | Parâmetros | O que faz |
|---|---|---|
| `create_task` | `branch`, `repo`: `owner/repo`; opcionais `baseUpdate` (`skip`, `pull`, `regen`), `slot`, `strategy`, `replace` | `automacli clone` sem prompts: cria a task por snapshot do repo base, aloca o slot, gera o override e injeta os templates. Devolve o JSON do clone (diretório, slot, portas, estratégia, repo base) e as últimas linhas do progresso. |
| `update_base_repo` | `repo`; `force` | `automacli base update` em segundo plano: pull, regenerar artefatos, testes e pré-aquecimento do perfil. Devolve na hora o arquivo de log; o resultado aparece em `get_base_repos.lastUpdate`. |
| `start_shared_services` | `repo`: `owner/repo` | `automacli shared up`: sobe os compartilhados no repo base, migrando os que rodam dentro de uma task. |
| `stop_task_stack` | `task` | `docker compose down` da task, mantendo o diretório. Recusa no repo base que hospeda os compartilhados. |
| `remove_task` | `task`; `force` (padrão `false`) | `automacli branches --remove`: para a stack, retoma a posse dos arquivos e apaga o diretório (snapshot na hora, ou lixeira com remoção em segundo plano). Só aceita `lifecycle.canDelete: true`; `force` ignora os bloqueios. Nunca apaga o repo base. |
| `refresh` | — | Descarta os caches e inicia nova medição de disco. |

`get_branches_status` e `get_branch_detail`, da primeira versão, continuam existindo e devolvem os campos brutos.

Cada ação devolve `{ok, message, steps}`. Quando recusa, `message` explica o motivo e o que fazer antes.

## Diagnóstico

`diagnose` procura, nesta ordem de severidade:

| `code` | Severidade | O que significa | `fix` |
|---|---|---|---|
| `shared-stopped` | erro | Serviço compartilhado parado; as tasks não conectam. | `automacli shared up` / `start_shared_services` |
| `shared-no-base` | erro | O repositório usa compartilhados e não há repo base para hospedá-los. | `automacli clone … -f owner/repo` (oferece criar o base) |
| `containers-unhealthy` | erro | Container da task em `restarting` ou parado com erro. | `docker compose -p <projeto> logs --tail 50 <serviço>` |
| `shared-in-task` | aviso | Compartilhado rodando dentro de uma task; parar essa task derruba o das outras. | `automacli shared up` (migra) |
| `base-behind` | aviso | Repo base atrás da principal; tasks novas nascem desatualizadas. | `automacli base update` |
| `base-dirty` | aviso | Repo base com alterações locais; a atualização automática o pula. | `git status` no repo base |
| `slot-port-collision` | aviso | Dois slots publicam portas iguais; as stacks não sobem juntas. | reclonar uma das tasks |
| `oom-kills` | aviso | Mortes por falta de memória no slot nas últimas 24 h. | `automacli oom --slot N` |
| `task-finished-blocked` | aviso | Task terminada, mas com trabalho não commitado ou não enviado. | `git status` na task |
| `task-finished` | info | Task terminada e sem bloqueio: pode ser apagada. | `automacli branches --remove` / `remove_task` |
| `finished-stack-running` | info | Task terminada com a stack no ar gastando recurso. | `docker compose down` / `stop_task_stack` |
| `task-far-behind` | info | Task mais de 50 commits atrás da principal. | `git rebase origin/<principal>` |
| `no-base-repo` | info | Repositório sem repo base: clones completos, sem dependências prontas. | `automacli clone … -f owner/repo` |
| `storage-freeable` | info | Quanto disco real as tasks terminadas liberariam. | — |

## Um diálogo típico

> **Pessoa:** tem task que eu já posso apagar?
>
> **Agente** chama `get_overview`, vê `tasks.deletable: ["task-feat-login-owner-api"]`, chama `get_task` para
> confirmar `lifecycle`: `merged (pode apagar): PR #312 mesclado; branch remota já apagada`, sem bloqueios, e
> responde: "A `feat-login` foi mesclada no PR #312 e não tem nada local. Apagar libera o slot 3 e 1,2 GB. Apago?"
>
> **Pessoa:** apaga.
>
> **Agente** chama `remove_task` com `task: "feat-login"` e relata os passos: stack parada, posse retomada,
> subvolume apagado.

Para uma task que não terminou, `remove_task` recusa e explica. O agente só deve usar `force: true` depois de você
confirmar que o trabalho pode ser perdido; as instruções embutidas dizem isso a ele.

## Criar tasks pela IA

`create_task` é o `automacli clone` chamado pelo MCP: a IA informa a branch e o `owner/repo`, e recebe o diretório,
o slot, as portas por serviço e a estratégia usada. Por baixo, o servidor roda o próprio automacli como subprocesso
com `--json` e as flags sem prompt, então o comportamento é idêntico ao do terminal: perfil, templates, lixeira,
tudo igual.

> **Pessoa:** cria uma task para a branch `feat/pagamentos` do `minha-org/minha-api`.
>
> **Agente** chama `get_base_repos`, vê o repo base 40 commits atrás, chama `update_base_repo` e avisa: "o repo base
> está atualizando em segundo plano; crio a task agora a partir dele como está, ou espero?" Depois chama
> `create_task` com `branch: "feat/pagamentos"` e responde com o diretório e as portas.

Duas regras que as instruções embutidas já dão ao agente:

* Um repo base atrasado é usado como está (`baseUpdate: skip`, o padrão). `regen` dentro do `create_task` funciona,
  mas pode levar minutos e estourar o timeout do cliente MCP; por isso existe `update_base_repo`, que roda solto.
* Uma task que já existe dá erro, nunca é apagada; `replace: true` só depois de a pessoa confirmar.

Sem MCP, o mesmo é possível pelo terminal, com as flags do clone:
[Comandos › clone](comandos.md#sem-terminal-delegar-o-clone-a-uma-ia-ou-a-um-script).

## HTTP

Tudo que o MCP responde também existe na [API HTTP](monitor.md#api-http) do monitor: `GET /api/overview`,
`GET /api/branches/:dirName/changes`, `GET /api/shared`, `GET /api/diagnose?task=…`, e `lifecycle`, `changes` e
`unhealthyContainers` em cada task de `GET /api/branches`. As ações ficam só no MCP e no terminal.

## Segurança

O servidor MCP roda por stdio, no seu usuário, só para o cliente que o iniciou. As ações fazem o mesmo que os
comandos do terminal e respeitam os mesmos limites: nunca apagam o repo base, nunca param os compartilhados por
uma task, e `remove_task` sem `force` só apaga o que o ciclo de vida liberou. Mesmo assim, configure o cliente para
pedir confirmação nas ferramentas de ação.
