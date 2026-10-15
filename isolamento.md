---
url: https://gustavo210.github.io/automacli-releases/isolamento.md
---

# Isolamento de recursos

Opcional. Disponível em Linux com systemd e Docker usando o driver de cgroup `systemd`.

Com várias stacks no ar, um teste descontrolado em uma task pode consumir toda a memória e travar a máquina. Com o isolamento ligado, tudo o que uma task roda entra em um slice systemd próprio, `automacli-slot<N>.slice`, com limite de memória e peso de CPU reduzido (`CPUWeight=50`: a task perde a disputa de CPU para o desktop, sem teto fixo). Quem estoura o limite de memória é morto dentro do slice, e o resto da máquina segue respondendo.

## Ligar

No perfil do repositório:

```yaml
isolation:
  enabled: true
  pidsLimit: 2048
```

E, uma vez por máquina, instale os slices e os limites a partir do repositório do automacli:

```sh
sudo bash scripts/install-system-limits.sh   # slices, systemd-oomd, sysctl e daemon.json do Docker
bash scripts/install-user-limits.sh          # slices da sessão do usuário
sudo systemctl restart docker                # quando for conveniente
```

::: warning Atenção
`install-system-limits.sh` altera `/etc/docker/daemon.json` (define `cgroup-parent`, `live-restore` e, se ainda não houver, `default-address-pools`; guarda um backup em `daemon.json.bak-automacli`) e o primeiro restart do Docker derruba os containers em execução. Leia o script antes de rodar.
:::

Sem os slices instalados, o `cgroup_parent` ainda agrupa os containers por slot, mas **sem limite de memória nem de CPU**; só o `pids_limit` por container, aplicado pelo Docker, continua valendo.

## Como cada coisa entra no slice

| O que roda | Como entra | Onde ficam os limites |
|---|---|---|
| Containers da stack | `cgroup_parent` no override | `/etc/systemd/system/automacli-.slice.d/` |
| Containers de teste | `cgroup_parent` em `docker-compose.limits.yml`, ao lado de cada `compose.testFile` | idem |
| Outros containers | `cgroup-parent: automacli.slice` no `daemon.json` | `/etc/systemd/system/automacli.slice` (teto global) |
| Processos do host | `systemd-run --user --scope --slice=automacli-slot<N>.slice` | `~/.config/systemd/user/automacli-.slice.d/` |

O automacli decide **onde** cada processo roda. **Quanto** ele pode usar fica na pasta `system/` do repositório, nunca no código: tetos globais nos arquivos `.slice`, limites por slot nos `.conf`.

### Arquivos de teste

O Compose rejeita um override que mencione um serviço ausente dos arquivos passados com `-f`. Por isso os testes não podem reaproveitar o override da stack: para cada arquivo com o nome de `compose.testFile`, o automacli grava ao lado um `docker-compose.limits.yml` com os serviços daquele teste e os de `compose.testBaseFiles`. Inclua-o no comando de teste:

```sh
docker compose -f docker-compose.base.yml \
  -f apps/api/docker-compose.test.yml \
  -f apps/api/docker-compose.limits.yml run --rm api
```

### Processos do host

Para `pnpm`, `jest` e afins rodando fora do Docker, use um wrapper nos seus [templates](instalacao.md#estrutura-do-diretorio-de-configuracao):

```sh
#!/bin/bash
# scripts/isolate.sh — uso: scripts/isolate.sh pnpm test
SLOT=$(grep -oE '"slot":[[:space:]]*[0-9]+' .slot | grep -oE '[0-9]+$' || echo 0)
exec systemd-run --user --quiet --scope --slice="automacli-slot${SLOT}.slice" -- "$@"
```

## Saber o que foi morto

Um serviço morto por falta de memória não avisa: o container some ou reinicia, e o sintoma aparece em outro lugar (um `502` no proxy, um teste que termina com `Killed`). Há três formas de ver o que aconteceu:

| Como | Quando usar |
|---|---|
| `automacli oom` | Depois do fato: lista quem morreu, quando, em qual slot e por quê. |
| Notificação do [monitor](monitor.md#notificacoes) | Na hora: aviso de desktop em até 30 segundos. |
| `GET /api/oom` e o campo `oomKills` de cada task | Para painéis e agentes de IA. |

Como ler o motivo:

* **`memory pressure for /automacli.slice/automacli-slot4.slice`**: o slot 4 sozinho passou do limite. O problema está naquela task.
* **`memory pressure for /automacli.slice`**: a soma de todas as stacks passou do teto global. Quem morre é o maior consumidor, que pode não ser o culpado. Pare stacks que não está usando.
* **`limite de memória de ... atingido`** (causa `kernel`): o slice bateu em `MemoryMax`.

Um container é identificado pelo nome do serviço enquanto existir no Docker. Depois de removido, só se o monitor (ou um `automacli oom` anterior) já tinha visto o container; caso contrário aparece como `container <id> (removido)` nas mortes do oomd, ou só com o nome do processo nas mortes do kernel.

## Limites padrão

| Escopo | `MemoryHigh` | `MemoryMax` | Pressão para o oomd matar |
|---|---|---|---|
| Containers de um slot | 13 GB | 14 GB | 80% por 20 s |
| Todas as stacks juntas | 20 GB | 22 GB | 70% por 20 s |
| Processos do host de um slot | 6 GB | 8 GB | 60% por 20 s |
| Todos os processos do host juntos | 10 GB | 12 GB | 60% por 20 s |

Os valores partem de uma máquina com 30 GB. Uma stack completa com os front-ends compilando passa de 7 GB; o `MemoryHigh` fica perto do `MemoryMax` de propósito, porque é ao atingi-lo que o kernel começa a frear o grupo, e é essa frenagem que o `systemd-oomd` lê como pressão. Com o High baixo demais, o oomd mata um container durante um pico de compilação mesmo havendo memória sobrando na máquina.

`MemoryHigh` desacelera o grupo antes de matar; `MemoryMax` é o teto. O `systemd-oomd` mata o cgroup (o container, ou o scope do processo do host) que mais pressiona a memória dentro do slice, e o teste falha com `Killed` ou código 137 em vez de travar a máquina.

Para mudar, edite os `.slice` e `.conf` em `system/` e rode os scripts de instalação de novo. Antes de subir um limite, tente reduzir o paralelismo do teste (`--maxWorkers`): o peso de CPU não reduz o número de workers do Jest, que conta as CPUs do host.

::: info Nota
Nunca coloque o slice da sua sessão gráfica sob o `systemd-oomd`. Em muitos ambientes o compositor, os terminais e o navegador vivem no mesmo escopo, e um kill ali encerra a sessão inteira.
:::
