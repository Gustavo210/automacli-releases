---
url: https://gustavo210.github.io/automacli-releases/solucao-de-problemas.md
---

# Solução de problemas

Comece sempre por `automacli profile` dentro da task: ele mostra qual arquivo de perfil está valendo e a porta de cada serviço.

## "port is already allocated" ao subir a stack

| Causa provável | Como conferir | Correção |
|---|---|---|
| O override não está sendo carregado | `docker compose config` não mostra as portas do slot | Com `-f` explícito, o Compose ignora o override: liste o Makefile/script em `compose.wireInto` ou acrescente `-f docker-compose.override.yml`. |
| A stack compartilhada está parada | `automacli shared status` mostra `parado` | `automacli shared up`. Com o monitor rodando ela volta sozinha em até um minuto ([autostart](monitor.md#servicos-compartilhados-sempre-no-ar)). |
| O serviço deveria ser compartilhado e não está no perfil | `automacli profile` não o marca como compartilhado | Adicione-o a `shared.services`. |
| Duas portas colidem depois do deslocamento | Tabela de `automacli profile` com portas repetidas | Aumente `slots.offset`. |
| `port is already allocated` ao subir a stack temporária de um script no repo base | Outro slot publica uma porta igual à do slot 0 (ex.: `adm-api` do slot 8 na `8808`) | Pare a stack daquele slot enquanto o script roda. Tasks criadas com a versão atual já não recebem slots que colidem. |

## "all predefined address pools have been fully subnetted"

O Docker reserva uma sub-rede para cada projeto compose e, por padrão, só tem 31. Cada slot usa uma, e comandos rodados sem `-p` fora da raiz da task (ou com `compose.projectNameEnv: false`) criam mais uma com o nome do diretório, que fica para trás depois do `run --rm`.

Para liberar na hora, remova as redes que nenhum container usa:

```sh
docker network prune
```

Para não voltar a acontecer, aumente os pools em `/etc/docker/daemon.json` e reinicie o Docker:

```json
{
  "default-address-pools": [{"base": "172.16.0.0/12", "size": 24}]
}
```

Isso dá 4096 redes de 254 endereços e tira o Docker da faixa `192.168.x`, que costuma colidir com a rede local. O `scripts/install-system-limits.sh` já grava essa chave quando o `daemon.json` ainda não define `default-address-pools`; se já definir, ajuste à mão. Redes existentes mantêm a sub-rede antiga até serem recriadas.

## `automacli profile` não mostra nenhuma porta

* `compose.files` aponta para arquivos que não existem nesse diretório.
* As portas não têm host fixo (`"3000"` sozinho) ou são faixas; declare-as em `ports.services`.
* A porta de host vem de uma variável sem valor (`${API_PORT}:3000`): sem `.env`, nem o `docker compose config` nem a leitura direta do YAML conseguem resolvê-la. Crie o `.env` ou declare essas portas em `ports.services`.

## A aplicação ainda chama a porta antiga

O override só muda o Docker. A porta está escrita em algum arquivo que não entrou em `rewrite`:

```sh
grep -rn ':4000' --include='.env*' --include='package.json' .
```

Acrescente o arquivo a uma regra de [`rewrite`](perfil.md#rewrite). Se a porta é de um processo fora do compose, inclua-a em `ports.extra`. A reescrita acontece no `clone`; em uma task já criada, ajuste o arquivo à mão.

## A task subiu com o banco vazio

* O serviço de banco não está em `shared.services`, então cada task tem o seu.
* O volume não está em `shared.volumes`.
* `compose.projectName` difere do nome com que a sua stack principal roda. Os volumes compartilhados se chamam `<projectName>_<volume>`: confira com `docker volume ls`.

## Todas as tasks perderam o banco ou o DNS de uma vez

Os compartilhados estavam rodando dentro de uma task que foi parada (perfis com `shared.host: first-task`, ou stacks criadas antes da stack compartilhada). Confira com `automacli shared status` e suba a partir do repo base com `automacli shared up`, que move o que ainda estiver em alguma task.

## `QueueDoesNotExist` depois de reiniciar a máquina (ou o ElasticMQ)

As filas por slot (`marketplace-api-slot4`, `gerar-pagamento-slot4.fifo`) são criadas pela API e vivem na memória do ElasticMQ compartilhado: quando o container reinicia, só as do `queues.conf` voltam. `automacli shared status` mostra quantas faltam; `automacli shared up` (ou `--all`) recria todas. Com o monitor rodando e `shared.autostart` ligado, isso acontece sozinho em até um minuto.

## O monitor mostra a task sem containers

O projeto compose em execução não é o registrado no `.slot`. Isso acontece quando a stack foi iniciada com outro `-p` ou sem `COMPOSE_PROJECT_NAME`. Compare `cat .slot` com `docker compose ls`.

## Cada task parece ocupar o disco inteiro de novo

`du` e gerenciadores de arquivos mostram o tamanho aparente, que conta de novo os blocos compartilhados pelo reflink. Para ver o que cada task ocupa de fato:

```sh
btrfs filesystem du -s ~/tasks/task-*/
```

A coluna `Exclusive` é o que você recupera apagando a task. O [monitor](monitor.md#disco-aparente-e-real) mostra os dois números. O que faz uma task crescer é o que ela gera depois do clone: dependências reinstaladas e builds deixam de ser compartilhados.

## O clone não usou reflink

`cp --reflink` só funciona em Btrfs, XFS (com reflink habilitado) e APFS, com origem e destino no mesmo sistema de arquivos; em Btrfs o automacli prefere um snapshot do repo base. Sem reflink ele cai para `git clone --reference`, que reaproveita os objetos do repo base mas não traz as dependências instaladas. `automacli doctor` mostra a estratégia escolhida e por que as outras foram descartadas.

## Um serviço parou de responder sem erro aparente

Sintomas típicos: `502` no load balancer, container que reinicia sozinho, teste que termina com `Killed` ou código 137. Verifique se ele foi morto por falta de memória:

```sh
automacli oom --since 1h
```

Se aparecer na lista, veja [como ler o motivo](isolamento.md#saber-o-que-foi-morto).

## Containers presos em "Restarting"

Workers que dependem de um serviço compartilhado reiniciam em laço quando ele cai. Pare-os e suba a stack que hospeda os serviços compartilhados primeiro:

```sh
docker ps -aq --filter status=restarting | xargs -r -n 1 docker stop
```

## Perfil inválido

A mensagem aponta o campo. Erros comuns: indentação do YAML, `mode` de `rewrite` diferente de `env`, `cli-flags` ou `urls`, e portas fora do formato `host:container`.

## Owner ou repositório identificados errado

O automacli lê `owner` e `repo` do `.slot` da task; sem `.slot`, do remoto `origin`; sem remoto, deduz do nome `task-<branch>-<owner>-<repo>`, o que falha quando owner ou repo têm hífen. Confira com `cat .slot` e `git remote get-url origin`.
