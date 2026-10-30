---
url: https://gustavo210.github.io/automacli-releases/monitor.md
---

# Monitor

O monitor é um servidor local que reúne o estado de todas as tasks: git, pull request, containers, uso de CPU e memória. Ele expõe os dados por HTTP (para painéis e aplicativos) e por MCP (para agentes de IA).

```sh
automacli monitor                 # http://localhost:7777
automacli monitor setup           # instala como serviço e inicia no login
```

::: warning Atenção
A API **não tem autenticação** e aceita requisições de qualquer origem (CORS aberto). Deixe-a acessível só em redes em que você confia, ou atrás de uma VPN como Tailscale ou WireGuard.
:::

## O que é coletado

| Dado | Fonte | Atualização |
|---|---|---|
| Arquivos não commitados, branch publicada e commits atrás do remoto | `git status`, `ls-remote` e `fetch` na task (o nome da branch vem do diretório) | 60 s |
| Containers em execução | `docker compose -p <projeto> ps` | 15 s |
| Pull request, revisões e checks | API do GitHub (precisa de `githubToken`) | 5 min |
| Tamanho aparente | `du` | 5 min |
| Uso real de disco (só btrfs) | `btrfs filesystem du` | 30 min, em segundo plano |
| CPU e memória por serviço | `docker stats`, agrupado pelos labels do Compose | 5 s, últimos 5 min |
| URLs dos serviços | [`urls`](perfil.md#urls) do perfil | a cada consulta |
| Mortes por falta de memória | journal do sistema (systemd-oomd e kernel), só nos cgroups do automacli; sem `journalctl`, o `OOMKilled` registrado pelo Docker | 30 s |
| Atualização dos repos base | `git fetch`, `pull`, codegen e testes do perfil | `baseRepo.autoUpdate.every` de cada repositório |

As estatísticas cobrem qualquer projeto compose em execução na máquina, não só as tasks.

## Disco: aparente e real

Com reflink, as tasks compartilham blocos com o repo base. Ferramentas como `du` não sabem disso e contam cada arquivo inteiro, então dez tasks de 9 GB parecem ocupar 90 GB quando ocupam uma fração. O monitor informa os dois números:

| Campo | O que é |
|---|---|
| `apparentBytes` | Soma do tamanho dos arquivos, como `du` mostra. É quanto ocuparia **sem** o btrfs. |
| `exclusiveBytes` | Blocos que só aquela task usa. É o que volta para o disco ao apagá-la. |
| `total.realBytes` | Ocupação de verdade do diretório das tasks: o exclusivo de todas, mais o compartilhado contado uma vez. |
| `total.savedBytes` | Economia do reflink: aparente − real. |
| `total.sharedBytes` | Blocos compartilhados entre tasks, contados uma vez. |

Cada task traz `storage: {apparentBytes, exclusiveBytes}`, e a resposta de `/api/branches` traz `storage: {measuredAt, measuring, supported, total}` com o resumo. `/api/storage` devolve o relatório inteiro: `{durationMs, measuredAt, measuring, supported, tasks: {<diretório>: {apparentBytes, exclusiveBytes}}, total}`. O campo `sizeBytes` continua sendo o tamanho aparente.

A medição lê os extents de todos os arquivos e leva alguns minutos de CPU, por isso roda em segundo plano com prioridade mínima, no máximo a cada 30 minutos e só quando alguém consulta a API. Até a primeira terminar, `storage` vem `null`. Fora de btrfs não há dado de compartilhamento: `supported` vem `false` e vale só o `sizeBytes`.

## API HTTP

| Método | Rota | Resposta |
|---|---|---|
| `GET` | `/api/health` | `{status, uptimeMs, configLoadedAt}` |
| `GET` | `/api/branches` | Todas as tasks. |
| `GET` | `/api/branches/:dirName` | Uma task, pelo nome do diretório. `404` se não existir. |
| `GET` | `/api/branches/:dirName/changes` | Commits, arquivos e não commitado da task em relação à principal, mais o `lifecycle`. |
| `GET` | `/api/overview` | Resumo: repositórios, repo base, slots, compartilhados, tasks por ciclo de vida, disco. |
| `GET` | `/api/summary` | Resumo compacto para o telefone: containers no ar e com problema, stacks, tasks (slot, containers, CPU e memória da última amostra, disco real e aparente, URLs, PR com review e checks, arquivos não commitados, `canDelete`), compartilhados, OOM em 24 h, disco real total e espaço. Veja [Acesso do telefone](#acesso-do-telefone). |
| `GET` | `/api/devices` | Aparelhos pareados: escopo, desativado, último acesso, IP, pedidos e conexões abertas. |
| `GET` | `/api/events?since=` | Resumo e alertas em Server-Sent Events. `since` (ou o cabeçalho `Last-Event-ID`) reenvia o que o cliente perdeu; veja [Eventos](#eventos). |
| `GET` | `/api/shared` | Serviços compartilhados de cada repositório e onde rodam. |
| `GET` | `/api/diagnose?task=` | Achados do diagnóstico, com severidade e correção; `task` filtra. |
| `GET` | `/api/queues?task=` | Filas SQS por task: nome, `visible` (aguardando consumidor), `inFlight` (em processamento), `delayed` e `state`, mais `endpoint` da API e `ui` (interface web do ElasticMQ); com ElasticMQ compartilhado, separadas por slot pelo sufixo. `task` filtra. |
| `GET` | `/api/stats` | Histórico de CPU/memória de cada projeto compose. |
| `GET` | `/api/stats/:project` | Histórico de um projeto. |
| `GET` | `/api/stats/:project/services` | Histórico por serviço de um projeto. |
| `GET` | `/api/stats/:project/stream` | O mesmo, em Server-Sent Events a cada 5 s. |
| `GET` | `/api/oom?since=24h` | Serviços e processos mortos por falta de memória no período. |
| `GET` | `/api/base` | Histórico das atualizações dos repos base (status, commits, passos e saída). |
| `GET` | `/api/storage` | Uso real de disco: total e por task. `null` até a primeira medição terminar. |
| `POST` | `/api/storage/refresh` | Dispara uma nova medição de disco (ignorado se já houver uma em andamento). |
| `POST` | `/api/cache/invalidate` | Descarta os caches de git, containers, PR, OOM e `du` (não a medição de disco real) e força nova coleta. |
| `GET` | `/dashboard/:project` | Página HTML com gráficos ao vivo do projeto. |

As rotas `GET /api/*` de dados respondem `{data, timestamp}`. `/api/health` responde `{status, uptimeMs, configLoadedAt}`, os `POST` respondem `{message}` e um caminho inexistente responde `{error}` com 404. O stream SSE envia o array de serviços direto, sem envelope.

### Formato de uma task

```json
{
  "dirName": "task-minha-feature-minha-org-minha-api",
  "directory": "/home/voce/tasks/task-minha-feature-minha-org-minha-api",
  "branch": "minha-feature",
  "owner": "minha-org",
  "repo": "minha-api",
  "uncommittedFiles": 2,
  "pushedToRemote": true,
  "behindRemoteCount": 0,
  "sizeBytes": 734003200,
  "storage": {"apparentBytes": 734003200, "exclusiveBytes": 52428800},
  "slot": {"slot": 2, "composeProjectName": "minha-api-slot2", "createdAt": "2026-10-05T12:00:00.000Z"},
  "stackRunning": true,
  "runningContainers": ["api", "web"],
  "unhealthyContainers": [{"service": "worker", "state": "exited:137"}],
  "urls": {"api": "http://localhost:4200", "web": "http://localhost:3200"},
  "changes": {"baseBranch": "main", "aheadOfBase": 3, "behindBase": 0, "mergedIntoBase": false, "remoteBranchExists": true, "unpushedCommits": 0, "changedFilesCount": 7, "lastCommitAt": "2026-10-05T11:50:00.000Z"},
  "lifecycle": {"state": "in-review", "canDelete": false, "reasons": ["PR #128 aprovado, aguardando merge"], "blockers": ["2 arquivo(s) não commitado(s)"], "warnings": ["stack no ar: será parada ao apagar"], "wouldFree": {"slot": 2, "exclusiveBytes": 52428800}},
  "pr": {
    "number": 128,
    "title": "Minha feature",
    "url": "https://github.com/minha-org/minha-api/pull/128",
    "state": "approved",
    "reviewState": "approved",
    "reviewCount": 1,
    "mergeable": true,
    "mergeableState": "clean",
    "checks": [{"name": "ci", "state": "success"}]
  },
  "oomKills": [],
  "lastScannedAt": "2026-10-05T12:34:56.000Z"
}
```

`oomKills` traz as mortes por falta de memória no slot da task nas últimas 24 horas, no formato abaixo. `changes` resume a diferença em relação à branch principal e `lifecycle` diz em que ponto a task está e se pode ser apagada; os dois estão explicados em [MCP para agentes de IA](mcp.md#ciclo-de-vida-de-uma-task). `unhealthyContainers` lista serviços em `restarting` ou parados com erro.

### Formato de uma morte por falta de memória

```json
{
  "at": "2026-10-05T20:57:26.998Z",
  "kind": "oomd",
  "slot": 4,
  "service": "web",
  "composeProject": "minha-api-slot4",
  "container": "minha-api-slot4-web-1",
  "containerId": "71e074b70d13...",
  "process": null,
  "unit": "docker-71e074b70d13....scope",
  "cgroup": "/automacli.slice/automacli-slot4.slice/docker-71e074b70d13....scope",
  "reason": "memory pressure for /automacli.slice/automacli-slot4.slice being 62.69% > 60.00% for > 10s"
}
```

`kind` é `oomd` (pressão de memória no slice) ou `kernel` (OOM do kernel: teto `MemoryMax` do slice ou memória do sistema esgotada; `process` traz o nome do processo quando lido do journal). `service`, `composeProject` e `container` são `null` para processos do host e para containers removidos antes de o monitor os ter visto.

`pr.state` é `draft`, `merged`, `closed`, `approved` (aberto e aprovado) ou `open`; `reviewState` é `approved`, `changes_requested`, `pending` ou `none`; `checks[].state` é `success`, `failure`, `error` ou `pending`.

`slot` e `urls` são `null` em tasks clonadas com `--no-slot` (nelas `stackRunning` é `false` e `runningContainers` é `[]`); `urls` também é `null` quando o perfil da task é inválido. `pr` é `null` sem token ou sem pull request.

### Formato das estatísticas

`/api/stats` devolve `[{composeProjectName, history}]`; `/api/stats/:project` devolve só o `history` do projeto; `/api/stats/:project/services` e o `/stream` devolvem um item por serviço, do mais pesado para o mais leve:

```json
[
  {
    "service": "api",
    "history": [{"cpuPercent": 3.2, "memoryMb": 412, "timestamp": "2026-10-05T12:34:50.000Z"}]
  }
]
```

`history` guarda até 60 amostras, uma a cada 5 segundos.

## Acesso do telefone

O monitor escuta em todas as interfaces, mas só a própria máquina usa a API sem autenticação (painel, CLI, `curl` local), seja por `localhost`, seja por um IP dela mesma, como o painel Quickshell faz. De fora, cada pedido precisa do token de um aparelho pareado. Sem nenhum aparelho pareado, pedidos de fora recebem `401`.

```sh
automacli monitor pair s24              # token com controle (parar e iniciar stacks) e QR code
automacli monitor pair tablet --read-only
automacli monitor pair --list           # último acesso, IP e conexões abertas de cada aparelho
automacli monitor pair --disable s24    # o token para de valer e as conexões caem; --enable reativa
automacli monitor pair --revoke s24
```

Cada aparelho tem um **escopo**: `control` (padrão ao parear) pode chamar as ações abaixo; `read` só lê (`POST` responde `403`). Aparelhos pareados antes de existir escopo contam como `read`: pareie de novo para ganhar controle. `--disable` mantém o aparelho no `config.json` mas recusa o token e fecha as conexões de eventos dele na hora; `--revoke` apaga. `GET /api/devices` lista os aparelhos com último acesso, IP, quantidade de pedidos e conexões abertas (o token nunca sai da máquina; só os últimos 6 caracteres).

| Método | Rota | Ação |
|---|---|---|
| `POST` | `/api/tasks/:dirName/stack/stop` | Para a stack da task (recusa se ela hospeda os compartilhados). |
| `POST` | `/api/tasks/:dirName/stack/start` | Sobe a stack: `commands.start` do perfil ou `docker compose -p <projeto> -f <arquivos> -f <override> up -d` dentro da task. |
| `POST` | `/api/tasks/:dirName/remove` | Apaga a task como `automacli branches --remove`: só quando o ciclo de vida diz que pode (`canDelete`); sem `force`. |
| `POST` | `/api/shared/:owner/:repo/up` | Sobe (ou migra para o repo base) os compartilhados. |
| `POST` | `/api/shared/:owner/:repo/down` | Para os compartilhados; o monitor só os sobe de novo no próximo boot ou com `shared up`. |

As ações respondem `{data: {ok, message, steps}}` com `200` quando fizeram e `409` quando se recusaram (motivo em `message`); task inexistente é `404`. Da própria máquina não precisam de token.

O QR code é um link `automacli://pair?url=…&token=…&name=…` que o app do automacli abre ao ser lido pela câmera. O token vai em `Authorization: Bearer <token>` ou, para um `EventSource` de navegador, em `?token=`. Os aparelhos ficam em `monitor.devices` do `config.json`; parear e revogar valem assim que o monitor relê o arquivo, sem reiniciar. Não há TLS: o acesso é pensado para a rede local (o app só conecta quando está no mesmo Wi‑Fi do PC).

## Eventos

`GET /api/events` mantém a conexão aberta e manda um evento por linha `data:`, com `id` igual ao número de sequência e `event` igual ao tipo. Ao conectar, o cliente recebe o último resumo; com `since=<id>` (ou `Last-Event-ID`) recebe também o que perdeu enquanto esteve fora. O monitor guarda os últimos 500 eventos; se o cliente perdeu mais do que isso, chega um `event: reset` e ele deve recarregar o estado por `/api/summary`. Um comentário `: ping` a cada 20 s mantém a conexão viva.

| Tipo | Nível | Quando |
|---|---|---|
| `summary` | info | O resumo mudou (container subiu ou caiu, stack parou, compartilhado voltou...) ou passaram 2 min desde o último, para CPU e memória seguirem atuais. `data` traz o resumo inteiro. |
| `container-unhealthy` / `container-recovered` | error / info | Um container entrou em `restarting` ou parou com erro, e quando voltou. |
| `shared-stopped` / `shared-recovered` | error / info | Um serviço compartilhado parou, e quando voltou. |
| `space-low` / `space-recovered` | warn ou error / info | Espaço não alocado do btrfs baixo ou crítico. |
| `oom` | error | Serviço morto por falta de memória (o mesmo que gera a notificação de desktop). |
| `base-update-failed` | warn | A atualização automática do repo base falhou. |

Cada evento tem `key`, que identifica a condição (ex.: `unhealthy:<task>:<serviço>`): o alerta e a recuperação compartilham a mesma `key`, para o cliente apagar a notificação quando o problema some. O observador roda a cada 30 s; o que já estava quebrado quando o monitor iniciou não gera alerta de novo, só aparece no resumo.

## Repos base atualizados sozinhos

Com `baseRepo.autoUpdate.enabled: true` no perfil, o monitor verifica no intervalo configurado se a branch principal avançou e, se avançou, atualiza o repo base, regenera os artefatos, roda os testes declarados e, por fim, o `warmup`. Um repo base com alterações locais não é tocado. Assim o próximo `clone` já parte de um base pronto, sem o prompt de atualização. Falhas no pull, no codegen ou nos testes geram notificação de desktop (`autoUpdate.notify`, ligado por padrão) e aparecem em `automacli base status`. Detalhes em [Perfil → baseRepo](perfil.md#baserepo).

## Serviços compartilhados sempre no ar

Com `shared.autostart: true` no perfil (padrão) e `shared.host: base`, o monitor mantém a stack compartilhada de cada repo base no ar:

* **No boot**: o serviço systemd inicia o monitor, que espera o Docker responder e então sobe os compartilhados que não voltaram sozinhos. Um `automacli shared down` anterior deixa de valer, como o `unless-stopped` do Docker.
* **A cada minuto**: sobe o que estiver parado (OOM, `make stop` no repo base, Docker reiniciado), a não ser que tenha sido parado de propósito com `automacli shared down`, que grava `.automacli-shared-stopped` no repo base. `automacli shared up` apaga o marcador.
* **Filas por slot**: as filas `*-slotN` vivem na memória do ElasticMQ e somem quando o container reinicia, e a task passa a receber `QueueDoesNotExist`. O monitor compara as filas esperadas de cada task com as que existem e recria as que faltam.

Para desligar num repositório, `shared.autostart: false` no perfil; vale assim que o arquivo é salvo. As ações aparecem no log com o prefixo `[shared owner/repo]`:

```sh
journalctl --user -u automacli-monitor -f | grep '\[shared'
```

## Notificações

Com o monitor em execução, cada serviço morto por falta de memória gera uma notificação de desktop (`notify-send`) em até 30 segundos, com o nome do serviço, o slot e o motivo. A mesma linha vai para o log do serviço:

```sh
journalctl --user -u automacli-monitor -f | grep '\[oom\]'
```

Só são notificadas as mortes que acontecem depois de o monitor iniciar. Para desligar, inicie com `--no-notify` ou defina `"notifications": false` no `config.json`, o que vale também para o serviço systemd, sem reiniciar.

## MCP

`automacli monitor --mcp-only` fala MCP por stdio e dá a um agente de IA o contexto do que o automacli gerencia:
tasks com ciclo de vida (pode apagar?), mudanças em relação ao repo base, disco real × aparente, serviços
compartilhados e um diagnóstico com correções. Ferramentas, estados e exemplos em [MCP para agentes de IA](mcp.md).

## Serviço systemd

`automacli monitor setup` cria `~/.config/systemd/user/automacli-monitor.service`, habilita o serviço e liga o *linger* do usuário, para ele rodar sem sessão aberta.

```sh
automacli monitor setup --status
journalctl --user -u automacli-monitor -f
systemctl --user restart automacli-monitor     # depois de atualizar o automacli
```
