---
url: https://gustavo210.github.io/automacli-releases/exemplos.md
---

# Exemplos de perfil

## API com banco, sem configuração

Repositório com um `compose.yaml` na raiz:

```yaml
services:
  api:
    build: .
    ports: ['4000:4000']
  db:
    image: postgres:16
    ports: ['5432:5432']
    volumes: [pgdata:/var/lib/postgresql/data]
volumes:
  pgdata:
```

Sem perfil nenhum, o automacli descobre as portas e dá a cada task a sua `api` **e o seu `db`**, em portas diferentes (`4100`/`5532`, `4200`/`5632`). Funciona, mas cada task tem um banco vazio.

Para todas as tasks usarem o mesmo banco:

```yaml
# .automacli.yml
compose:
  projectName: minha-api
shared:
  services: [db]
  volumes: [pgdata]
rewrite:
  - mode: env
    files: ['.env']
```

É, na essência, o que `automacli init` gera para esse repositório: ele ainda acrescenta `version`, `compose.files` e `isolation`, e só inclui `rewrite` se o checkout tiver um `.env`.

## Front-end com dev server no host

O compose só tem a API; o Vite roda fora do Docker, na 5173, com a porta escrita no `package.json`.

```yaml
compose:
  projectName: loja
ports:
  extra: [5173]              # não está no compose, mas precisa mudar por slot
rewrite:
  - mode: cli-flags
    files: ['package.json']  # "dev": "vite --port 5173"
  - mode: env
    files: ['.env']          # VITE_API_URL=http://localhost:4000
```

## Monorepo com Makefile e vários arquivos compose

```yaml
version: 1

compose:
  projectName: plataforma
  files:
    - docker-compose.base.yml
    - docker-compose.development.yml
  projectNameEnv: false          # o Makefile já passa -p
  wireInto: [Makefile]
  testFile: docker-compose.test.yml
  testBaseFiles: [docker-compose.base.yml]

ports:
  extra: [7000, 7001, 7002]      # dev servers dos front-ends

shared:
  services: [database, redis, opensearch, minio]
  volumes: [database_data, opensearch_data, minio_data]

services:
  gateway:
    user: '${UID}:${GID}'

rewrite:
  - mode: env
    files: ['.env', 'apps/*/.env']
  - mode: cli-flags
    files: ['apps/*/package.json']
  - mode: urls
    files: ['apps/gateway/generated/supergraph.graphql']

baseRepo:
  artifacts: [apps/gateway/generated/supergraph.graphql]
  watch: [composer.lock, pnpm-lock.yaml, supergraph.graphql]
  regenerate: ['make codegen']
  cacheDirs: [.artifact-cache]

isolation:
  enabled: true

commands:
  stop: make stop

urls:
  hostEnvVar: HOST_IP
  defaultHost: localhost
  services:
    gateway: {port: 8004}
    admin: {port: 7000}
    loja: {port: 7001, template: 'http://loja.devtest:{port}'}
```

## Lista de portas mantida à mão

Quando o compose usa recursos que a descoberta não cobre, ou você quer controle total:

```yaml
ports:
  discover: false
  services:
    proxy: ['8080:8080', '8443:8443']
    api: ['8008:80']
    dns: ['53:53/udp', '53:53/tcp']
shared:
  services: [dns]
services:
  dns:
    portHost: '${HOST_IP}'
```

## Regenerar o override ao subir a stack

Para que os serviços compartilhados sejam avaliados sempre no momento da subida, chame `automacli override` antes do `up`:

```make
up:
	@automacli override --quiet
	docker compose up -d
```

## Um ElasticMQ para todas as tasks (filas por slot)

Cada task com o próprio ElasticMQ custa memória à toa, e duas stacks com filas de mesmo nome só não se atrapalham
porque são instâncias separadas. Dá para ter um ElasticMQ só, compartilhado como o banco, desde que cada task use
as filas com um sufixo no nome. Três partes participam: o perfil, os templates e as aplicações.

**1. Perfil:** ponha o serviço em `shared.services` e descreva as filas em [`queues`](perfil.md#queues). Os padrões
servem para o ElasticMQ comum (API na 9324, interface na 9325, conta `000000000000`):

```yaml
ports:
  services:
    elasticmq: ['9325:9325', '9324:9324']
shared:
  services: [database, redis, elasticmq]
queues:
  service: elasticmq
  conf: apps/elasticmq/queues.conf   # as filas do slot 0, no formato do ElasticMQ
  suffix: '-slot{slot}'              # gerar-pagamento.fifo → gerar-pagamento-slot4.fifo
```

A partir daí o `clone` e o `automacli override` criam as filas do `conf` com o sufixo do slot pela API do ElasticMQ
(idempotente), `branches --remove` e `remove_task` as apagam, e o monitor mostra as de cada task em `/api/queues`
e `get_queues`. Suba o serviço uma vez com `automacli shared up`.

**2. Templates:** as aplicações precisam saber o sufixo. O placeholder `{{queueSuffix}}` vira `-slot4` na task,
vazio no repo base, e também vazio enquanto o ElasticMQ não estiver em `shared.services` (assim o mesmo template
serve nos dois modos). No `.env.template`:

```ini
ELASTICMQ_ENDPOINT=http://${HOST_IP}:9324/000000000000/
SQS_SUFFIX={{queueSuffix}}
```

Num arquivo que liste URLs completas de fila, o sufixo entra antes de `.fifo`:

```php
'GERAR_PAGAMENTO' => 'http://192.168.0.188:9324/000000000000/gerar-pagamento{{queueSuffix}}.fifo',
```

**3. Aplicações:** no Laravel, a conexão `sqs` aceita `suffix`, que ele põe antes de `.fifo`, exatamente como o
automacli nomeia as filas:

```php
'sqs' => [
    'driver' => 'sqs',
    'prefix' => env('AWS_PREFIX'),
    'queue' => env('SQS_QUEUE', 'default'),
    'suffix' => env('SQS_SUFFIX'),
],
```

e a variável precisa chegar ao container. Sem mexer no compose do repositório, o override faz isso por serviço,
com o valor já resolvido para o slot:

```yaml
services:
  api:
    environment: {SQS_SUFFIX: '{queueSuffix}'}
  api-queue-worker:
    environment: {SQS_SUFFIX: '{queueSuffix}'}
```

(ou, se o repositório quiser conhecer a variável, `SQS_SUFFIX: ${SQS_SUFFIX-}` no compose de desenvolvimento, com o
padrão vazio para produção).

Tasks criadas antes da mudança continuam com o ElasticMQ próprio até o override delas ser regenerado. Para migrar
uma: no `.env`, volte a porta do ElasticMQ para a original e defina `SQS_SUFFIX=-slot<N>`; depois
`automacli override --dir <task>` e reinicie a stack.
