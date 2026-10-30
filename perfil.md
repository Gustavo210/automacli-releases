---
url: https://gustavo210.github.io/automacli-releases/perfil.md
---

# Perfil do repositório

O perfil é um arquivo YAML que descreve **um** repositório para o automacli. Todos os campos são opcionais: um perfil vazio já funciona para um projeto com um único `compose.yaml` e nenhum serviço compartilhado.

| Onde fica | Quando usar |
|---|---|
| `~/.config/automacli/repos/<owner>-<repo>.yml` | Ajustes seus, que não vão para o repositório. Criado por `automacli init`. |
| `.automacli.yml` (ou `.automacli.yaml`) na raiz do repositório | Perfil comum ao time, versionado. Criado por `automacli init --local`. |

Se os dois existirem, vale o do usuário. Confira sempre com `automacli profile`, que mostra o arquivo em uso, os valores resolvidos e a tabela de portas.

## Exemplo completo

```yaml
version: 1
defaultBranch: main

compose:
  projectName: minha-api
  files: [docker-compose.yml]
  wireInto: [Makefile]

slots:
  offset: 100

ports:
  discover: true
  extra: [5173]

shared:
  services: [db, redis]
  volumes: [pgdata]

rewrite:
  - mode: env
    files: ['.env', 'apps/*/.env']
  - mode: cli-flags
    files: ['apps/*/package.json']

urls:
  services:
    api: {port: 4000}
    web: {port: 3000}
```

## Referência

### `version`

Versão do formato. Hoje só existe `1`.

### `defaultBranch`

Branch principal do repositório. Usada para criar branches novas e para medir o quanto o repo base está desatualizado. **Padrão:** a branch de `origin/HEAD`; se não houver, `main`.

### `compose`

| Campo | Padrão | Descrição |
|---|---|---|
| `projectName` | nome do repositório | Nome do projeto compose do slot 0. Os demais slots usam `<projectName>-slot<N>`. Deve ser o nome com que a sua stack principal roda hoje, se você quer reaproveitar os volumes dela. |
| `files` | primeiro existente entre `compose.yaml`, `compose.yml`, `docker-compose.yaml`, `docker-compose.yml` | Arquivos compose da stack de desenvolvimento, na ordem em que são passados com `-f`. |
| `overrideFile` | `docker-compose.override.yml` | Nome do arquivo gerado. |
| `projectNameEnv` | `true` | Grava `COMPOSE_PROJECT_NAME` no `.env` da task. Desligue se os seus comandos já passam `-p`. |
| `wireInto` | `[]` | Arquivos (Makefile, scripts) com comandos `docker compose`. Neles o automacli troca `projectName` pelo projeto do slot e acrescenta `-f <overrideFile>` depois do último arquivo de `files`. |
| `testFile` | — | Nome dos arquivos compose de teste espalhados pelo repo (ex.: `docker-compose.test.yml`). Usado só pelo [isolamento](isolamento.md). |
| `testBaseFiles` | `[]` | Arquivos compose sempre passados junto com cada `testFile`. |

`wireInto` é necessário quando os comandos usam `-f` explícito, porque nesse caso o Compose não carrega o override sozinho. Exemplo, antes e depois no slot 2:

```make
# antes
up:
	docker compose -p minha-api -f docker-compose.yml up -d

# depois
up:
	docker compose -p minha-api-slot2 -f docker-compose.yml -f docker-compose.override.yml up -d
```

### `slots`

| Campo | Padrão | Descrição |
|---|---|---|
| `offset` | `100` | Distância entre as portas de slots consecutivos. Com `100`, cada slot ocupa a "centena" seguinte. |

Escolha um offset maior que a maior distância entre duas portas que não devem colidir. Se a stack publica `8000` e `8100`, com offset `100` o slot 1 colidiria com a porta `8100` do slot 0. O automacli pula sozinho os slots que colidem (e avisa se você forçar `--slot`), mas os números de slot crescem; um offset de `1000` evita o salto.

### `ports`

| Campo | Padrão | Descrição |
|---|---|---|
| `discover` | `true` | Lê as portas publicadas dos arquivos em `compose.files`. |
| `services` | `{}` | Portas declaradas à mão, por serviço, na sintaxe curta do compose: `host:container` ou `host:container/proto`. **Substituem** as descobertas para o mesmo serviço. |
| `extra` | `[]` | Portas que não estão no compose mas aparecem nos arquivos reescritos, como dev servers rodando no host (`vite` na 5173). |

```yaml
ports:
  discover: false          # ignora o compose e usa só a lista abaixo
  services:
    web: ['3000:3000']
    dns: ['53:53/udp', '53:53/tcp']
  extra: [5173, 9229]
```

A descoberta usa `docker compose config`, que resolve variáveis, `extends` e `include` e expande faixas (`8000-8010:8000-8010` vira uma porta por valor). Sem Docker, os YAML são lidos diretamente, e aí faixas são ignoradas. Portas sem host fixo (`"3000"` sozinho, que o Docker mapeia para uma porta aleatória) são ignoradas nos dois casos.

### `shared`

| Campo | Padrão | Descrição |
|---|---|---|
| `autostart` | `true` | O [monitor](monitor.md#servicos-compartilhados-sempre-no-ar) mantém a stack compartilhada no ar: sobe no boot da máquina, levanta o que cair e recria as filas por slot quando o ElasticMQ reinicia. `automacli shared down` segura até o próximo boot ou `shared up`. Só vale com `host: base`. |
| `host` | `base` | Onde os compartilhados rodam: `base` (a partir do repo base, independentes das tasks) ou `first-task` (na primeira task que subir; comportamento antigo). |
| `services` | `[]` | Serviços com uma instância para todos os slots. Mantêm a porta original. Com `host: base`, ficam fora da stack de cada task; com `first-task`, só quando já rodam em outro slot. |
| `volumes` | `[]` | Volumes nomeados que todos os slots reutilizam. No override viram `external`, com o nome `<projectName>_<volume>`. |

Compartilhe o que é caro de duplicar e seguro de dividir: banco de dados, cache, busca, armazenamento de objetos. **Não** compartilhe o que carrega código da branch (APIs, workers, front-ends).

Todas as tasks enxergam os mesmos dados. Uma migração aplicada em uma branch afeta as outras.

### `services`

Ajustes por serviço escritos no override.

| Campo | Descrição |
|---|---|
| `user` | Diretiva `user:` do serviço. Com `${UID}:${GID}`, o automacli acrescenta `UID` e `GID` a cada arquivo de uma regra `rewrite` de modo `env` que ainda não os tenha (inclua o `.env` numa regra `env`). |
| `tmpfs` | Lista de montagens tmpfs. |
| `portHost` | Prefixo de host nos binds de porta. `${HOST_IP}` gera `'${HOST_IP}:53:53/udp'`. |
| `environment` | Variáveis de ambiente acrescentadas ao serviço no override. Nos valores, `{slot}` vira o número do slot, `{composeProject}` o projeto compose da task e `{queueSuffix}` o sufixo das filas de [`queues`](#queues) (vazio no slot 0 ou sem ElasticMQ compartilhado). Serve para passar algo às aplicações sem tocar no compose do repositório. |

```yaml
services:
  api:
    user: '${UID}:${GID}'       # arquivos criados no volume ficam com o seu usuário
    tmpfs: ['/tmp/cache:mode=1777']
    environment:
      SQS_SUFFIX: '{queueSuffix}'   # -slot4 na task, vazio no repo base
  dns:
    portHost: '${HOST_IP}'      # só escuta no IP da máquina
```

Um serviço listado aqui precisa existir nos arquivos compose; o Compose rejeita um override para um serviço sem `image` ou `build`.

### `rewrite`

Lista de regras que atualizam portas em arquivos da task depois do clone. Cada regra tem `files` e `mode`.

Em `files`, `*` casa qualquer trecho dentro de um segmento do caminho: `apps/*/.env`, `*.graphql`. Não existe `**`. Diretórios `node_modules`, `vendor` e `.git` são pulados.

| `mode` | O que troca | Exemplo (slot 1) |
|---|---|---|
| `env` | `:PORTA` em URLs e `VAR_PORT=PORTA` | `API_URL=http://localhost:4000` → `:4100` |
| `cli-flags` | `-p PORTA`, `--port PORTA`, `--listen PORTA` | `"dev": "vite --port 5173"` → `5273` |
| `urls` | `:PORTA` seguido de `/`, `"` ou `)` | `url: "http://localhost:4000/graphql"` → `:4100` |

Só entram as portas que mudam de slot para slot: as do compose que não são de serviços compartilhados, mais as de `ports.extra`. Linhas de comentário em arquivos `env` não são tocadas.

No modo `env`, uma atribuição simples só é trocada quando o nome da variável é maiúsculo e termina em `PORT` com algum prefixo (`WEB_PORT=3000`, `API_PORT=3000`). `PORT=3000` sozinho, nomes minúsculos e `TIMEOUT=3000` ficam como estão.

### `isolation`

| Campo | Padrão | Descrição |
|---|---|---|
| `enabled` | `false` | Coloca os containers de cada slot em um slice systemd `automacli-slot<N>.slice`. |
| `pidsLimit` | `2048` | Limite de processos por container. |

Veja [Isolamento de recursos](isolamento.md).

### `baseRepo`

Controla a manutenção do repo base.

| Campo | Padrão | Descrição |
|---|---|---|
| `artifacts` | `[]` | Arquivos gerados que precisam existir (ex.: schema compilado). Se faltarem, o `clone` oferece regenerar. |
| `watch` | `[]` | Trechos de caminho. Se a branch principal mudou algum arquivo que os contém, o `clone` lista o arquivo e sugere regenerar. |
| `regenerate` | `[]` | Comandos de shell executados na raiz do repo base para regenerar os artefatos. |
| `preserve` | `[node_modules, vendor]` | Caminhos mantidos no `git clean` ao trocar de branch. |
| `cacheDirs` | `[]` | Diretórios de cache apagados ao atualizar o repo base (e preservados na troca de branch). |
| `warmup` | `[]` | Comandos de pré-aquecimento executados na raiz do repo base por último (cache de bundler, compilação incremental). Melhor esforço: uma falha fica no histórico, mas não invalida a atualização. |
| `autoUpdate.enabled` | `false` | O monitor mantém o repo base atualizado sozinho. |
| `autoUpdate.every` | `1h` | Intervalo entre verificações: `30m` (ou `30min`), `2h`, `1d`. |
| `autoUpdate.regenerate` | `true` | Roda `regenerate` depois de atualizar. |
| `autoUpdate.test` | `[]` | Comandos de teste executados na raiz do repo base depois de atualizar e regenerar. |
| `autoUpdate.warmup` | `true` | Roda `warmup` depois de atualizar, regenerar e testar. |
| `autoUpdate.notify` | `true` | Aviso de desktop quando o pull, o codegen ou um teste falha. |

```yaml
baseRepo:
  artifacts: [packages/schema/dist/schema.graphql]
  watch: [pnpm-lock.yaml, schema.graphql]
  regenerate: ['pnpm install', 'pnpm codegen']
  warmup: ['pnpm --filter web exec expo export --dev --no-minify -p web --output-dir /tmp/warm']
  autoUpdate:
    enabled: true
    every: 2h
    test: ['pnpm test']
```

Com `autoUpdate` ligado, o [monitor](monitor.md) verifica a cada `every` se a branch principal avançou. Se avançou, faz `git pull --ff-only`, apaga `cacheDirs`, roda `regenerate` e depois `test`, nessa ordem, parando no primeiro que falhar. Por último roda `warmup`, sem parar em falha.

`warmup` existe porque caches de bundler (Metro, Vite, Turbopack...) são caros de construir a frio e, com `preserve` incluindo `node_modules`, uma task criada por snapshot ou reflink herda o que o repo base já tiver aquecido em `node_modules/.cache`. Assim o custo é pago uma vez, no repo base e em segundo plano, em vez de na primeira vez que cada task sobe. `automacli base warmup` roda só esses comandos, para aquecer na hora ou medir o ganho. Arquivos sobrescritos pelos [templates](instalacao.md#estrutura-do-diretorio-de-configuracao) não contam como alteração local: são restaurados antes do pull e reinjetados depois. Um repo base com outras alterações locais é pulado, e duas atualizações pelo monitor ou por `automacli base update` não se sobrepõem (lock em `.automacli-update.lock`). O resultado fica em `automacli base status` e em `GET /api/base`.

### `commands`

| Campo | Descrição |
|---|---|
| `start` | – | Comando de shell que sobe a stack da task (ação "iniciar" do app e da API). Omitido: `docker compose -p <projeto> -f <arquivos do compose> -f <override> up -d`. |
| `stop` | Comando de shell para parar a stack de uma task **sem** slot (clonada com `--no-slot`). Padrão: `docker compose down --remove-orphans`. |

Tasks com slot são paradas por `docker compose -p <projeto> down --remove-orphans`.

### `gitExclude`

Padrões adicionais para `.git/info/exclude` de cada task com slot (regravados por `automacli override`). Use para arquivos que os seus templates injetam e que não devem aparecer no `git status`.

### `urls`

URLs exibidas pelo [monitor](monitor.md).

| Campo | Padrão | Descrição |
|---|---|---|
| `services` | — | Mapa `nome: {port, template}`. `port` é a porta do slot 0: o monitor soma `slot × slots.offset`, salvo para portas de serviços em `shared.services`. `template` aceita `{host}` e `{port}`; padrão `http://{host}:{port}`. |
| `defaultHost` | `localhost` | Host usado nas URLs. |
| `hostEnvVar` | — | Variável do `.env` da task que contém o host (ex.: `HOST_IP`), para quem acessa por IP de rede. |

Sem `services`, o monitor gera uma URL por porta publicada de serviço não compartilhado (portas UDP ficam de fora).

## `queues`

Filas SQS da stack, servidas por um ElasticMQ. Serve para duas coisas: o monitor mostrar as filas de cada task
(`/api/queues`, `get_queues`), e, quando o ElasticMQ é **compartilhado**, cada task ganhar as próprias filas.

| Campo | Padrão | Descrição |
|---|---|---|
| `service` | `elasticmq` | Serviço compose que expõe a API SQS. Se a stack não o publica, não há filas. |
| `port` | `9324` | Porta original (slot 0) da API no host. |
| `uiPort` | `9325` | Porta original da interface web do ElasticMQ, que o monitor devolve como `ui` em `/api/queues`; `null` se não houver. |
| `accountPath` | `/000000000000` | Caminho da conta nas URLs das filas. |
| `conf` | `apps/elasticmq/queues.conf` | Arquivo de filas do ElasticMQ (HOCON): cada bloco de `queues { }` é uma fila; `fifo = true` marca FIFO e ganha `.fifo` no nome. |
| `suffix` | `-slot{slot}` | Sufixo dos nomes nos slots diferentes de 0; `{slot}` vira o número. |

Com `service` em `shared.services`, há um ElasticMQ só, no repo base, e as filas de uma task são as do `conf` com
o sufixo no nome: `marketplace-api-slot4`, `gerar-pagamento-slot4.fifo` (o sufixo entra antes de `.fifo`, como o
Laravel faz com `SQS_SUFFIX`). O `clone` e o `automacli override` criam essas filas pela API; `branches --remove` e
`remove_task` as apagam. Os apps precisam usar o mesmo sufixo: ponha `SQS_SUFFIX={{queueSuffix}}` no `.env` dos
[templates](instalacao.md#placeholders-nos-templates), que vira `SQS_SUFFIX=-slot4` na task, `SQS_SUFFIX=` no repo
base e também vazio enquanto o ElasticMQ não for compartilhado. Sem o serviço em `shared.services`, cada task tem o próprio ElasticMQ com as filas do `conf`, e nada muda.

## Validação

Um perfil inválido interrompe o comando com o campo apontado:

```text
~/.config/automacli/repos/minha-org-minha-api.yml: perfil inválido:
  - slots.offset: Invalid input: expected number, received string
```

Chaves desconhecidas são ignoradas sem aviso, inclusive erros de digitação; confira com `automacli profile` se o valor entrou.
