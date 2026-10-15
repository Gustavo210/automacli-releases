---
url: https://gustavo210.github.io/automacli-releases/requisitos.md
---

# Requisitos e ambiente

O automacli precisa de pouco para funcionar e aproveita o que a máquina oferece a mais. Nada além de git e Docker é obrigatório; cada recurso opcional melhora algo e, quando falta, o automacli usa a alternativa sozinho.

Para ver o que a sua máquina tem e o que o automacli vai fazer com isso:

```sh
automacli doctor
automacli doctor --repo minha-org/minha-api     # avalia a estratégia de clone com o repo base
```

## Obrigatório

| | Para quê |
|---|---|
| **git** | Clonar e acompanhar as branches. |
| **Docker com Compose v2.24 ou superior** | Subir as stacks. O override gerado usa a tag `!override`, que o Compose só aceita a partir da 2.24. |

O executável dos releases traz o próprio Node.js; para rodar do código-fonte é preciso Node 22 ou superior.

## Opcional

| Recurso | O que melhora | Sem ele |
|---|---|---|
| **btrfs** no diretório das tasks | Clones por snapshot (instantâneos) ou reflink; uso real de disco no monitor | Clones por `git clone --reference`; o monitor mostra só o tamanho aparente |
| **XFS ou APFS** (macOS) | Clones por reflink | Idem |
| **btrfs-progs** (`btrfs`) | Snapshot e medição de disco | Reflink continua funcionando |
| Montagem com `user_subvol_rm_allowed` | Apagar uma task por snapshot devolve o disco na hora | A task é movida para `.lixeira/` (instantâneo) e apagada em segundo plano; o disco volta em alguns minutos |
| **systemd** | Isolamento de CPU e memória por slot; monitor como serviço | Sem isolamento; monitor iniciado à mão |
| **journalctl** | `automacli oom` identifica mortes pelo systemd-oomd e pelo kernel, com o motivo | `automacli oom` usa só o que o Docker registra |
| **notify-send** | Aviso de desktop quando um serviço morre por falta de memória | Aviso só no log do monitor |
| **wl-copy, xclip, xsel, pbcopy** | O `clone` copia o `cd` da task | O `clone` imprime o `cd` |

## Como as tasks são criadas

O `clone` cria a task a partir do **repo base** (uma task mantida na branch principal) com a melhor estratégia disponível, caindo para a seguinte se uma falhar:

| Estratégia | Exige | Tempo em um repo de 9 GB e 875 mil arquivos | Traz `node_modules` instalado |
|---|---|---|---|
| **snapshot** | btrfs, btrfs-progs e repo base convertido em subvolume | < 1 s | sim |
| **reflink** | btrfs, XFS ou APFS | ~40 s | sim |
| **git clone --reference** | nada | segundos (reaproveita os objetos git do base, sem rede para o histórico) | **não**: instale depois (ou use um template para isso) |
| **git clone** | nada | depende da rede | não |

A estratégia é escolhida pelo `cloneStrategy` do `config.json` (`auto` por padrão; o `automacli init` pergunta) e pode ser escolhida por clone com `automacli clone … --strategy reflink`; se ela não estiver disponível ou falhar, o automacli cai para a próxima.

Para habilitar o snapshot, converta o repo base uma vez por repositório:

```sh
automacli base convert minha-org/minha-api
```

A conversão copia a árvore por reflink para um subvolume novo e troca os dois: não gasta disco, mantém o compartilhamento com as tasks já existentes e leva cerca de um minuto. Não rode com um clone em andamento.

### Snapshots e backups

Um snapshot do `/home` (Timeshift, snapper, `btrfs send`) **não inclui** subvolumes aninhados. Com o repo base convertido e as tasks criadas por snapshot, elas ficam fora desses backups. Para a maioria das pessoas isso é o desejado, já que tasks são descartáveis, mas vale saber.

Ferramentas que param na fronteira do sistema de arquivos (`du -x`, `find -xdev`, `rsync -x`) tratam cada subvolume como outro disco e não entram nele.

## Sistemas operacionais

| | Situação |
|---|---|
| **Linux** (qualquer distribuição) | Suportado. Executável, `.deb`, `.rpm` e `PKGBUILD` nos releases. |
| **macOS** | O código roda (reflink pelo APFS, sem systemd); ainda não há executável nos releases. |
| **Windows** | Não suportado. |
