---
url: https://gustavo210.github.io/automacli-releases/instalacao.md
---

# Instalação

## Instalar

### Executável pronto (recomendado)

Não precisa de Node: o pacote traz o próprio.

```sh
curl -fsSL https://github.com/Gustavo210/automacli-releases/releases/latest/download/install.sh | bash
```

Instala em `~/.local/share/automacli` e cria `~/.local/bin/automacli`. Versões específicas, repositório privado e instalação manual estão em [Releases e CI/CD](releases.md).

### Pela sua distribuição

Cada release também alimenta um repositório **apt** e um **rpm**, assinados, servidos ao lado desta documentação. Você
adiciona a fonte uma vez, instala pelo gerenciador, e o `apt upgrade`, o `dnf upgrade` ou o `zypper update` de sempre
trazem as versões novas. O comando fica em `/usr/bin/automacli`, com `git` como dependência e Docker como recomendação.
Arquiteturas: `amd64`/`x86_64` e `arm64`/`aarch64`.

::: code-group

```sh [Debian / Ubuntu]
sudo curl -fsSLo /usr/share/keyrings/automacli.gpg https://gustavo210.github.io/automacli-releases/apt/automacli.gpg
sudo curl -fsSLo /etc/apt/sources.list.d/automacli.sources https://gustavo210.github.io/automacli-releases/apt/automacli.sources
sudo apt update && sudo apt install automacli
```

```sh [Fedora / RHEL]
sudo curl -fsSLo /etc/yum.repos.d/automacli.repo https://gustavo210.github.io/automacli-releases/rpm/automacli.repo
sudo dnf install automacli
```

```sh [openSUSE]
sudo zypper addrepo -f https://gustavo210.github.io/automacli-releases/rpm/automacli.repo
sudo zypper --gpg-auto-import-keys install automacli
```

```sh [Arch / Manjaro]
# Pacote automacli-bin no AUR, com o seu helper preferido
yay -S automacli-bin        # ou: paru -S automacli-bin
```

```sh [Outra distribuição]
# Qualquer Linux x64 ou arm64: o instalador do executável, sem root
curl -fsSL https://github.com/Gustavo210/automacli-releases/releases/latest/download/install.sh | bash
```

:::

Os índices dos repositórios e os RPMs são assinados pela chave `automacli packages`, impressão digital
`1C27 7A0D 3569 3660 D604 431F B108 4A86 A1CD 6EA3`, publicada em
[`apt/automacli.asc`](https://gustavo210.github.io/automacli-releases/apt/automacli.asc) e
[`rpm/RPM-GPG-KEY-automacli`](https://gustavo210.github.io/automacli-releases/rpm/RPM-GPG-KEY-automacli).
Os repositórios guardam as três últimas versões; as anteriores continuam na página de
[Releases](https://github.com/Gustavo210/automacli-releases/releases), onde cada pacote também pode ser baixado
avulso (`automacli_amd64.deb`, `automacli.x86_64.rpm`, `PKGBUILD`), com o `SHA256SUMS`.

Para remover: `sudo apt remove automacli`, `sudo dnf remove automacli`, `sudo zypper remove automacli` ou
`sudo pacman -R automacli-bin`.

### Instalação manual

Sem o instalador: baixe o tarball da sua arquitetura (`x64` ou `arm64`) no [release](https://github.com/Gustavo210/automacli-releases/releases/latest), extraia e aponte um link para o executável. O tarball traz o Node embutido.

```sh
curl -fsSLO https://github.com/Gustavo210/automacli-releases/releases/latest/download/automacli-linux-x64.tar.gz
mkdir -p ~/.local/share/automacli ~/.local/bin
tar -xzf automacli-linux-x64.tar.gz -C ~/.local/share/automacli --strip-components=1
ln -sf ~/.local/share/automacli/bin/automacli ~/.local/bin/automacli
```

Confirme com `automacli --version`.

## Atualizar

```sh
automacli update            # correções e novidades compatíveis
automacli update --check    # só diz se há versão nova
automacli update --major    # aceita um salto de versão maior
```

Instalado pelo executável, o `update` baixa a versão nova, confere o checksum e troca o diretório de uma vez.
Instalado por pacote, ele mostra o comando do seu gerenciador, que é quem atualiza (`apt`, `dnf`, `zypper`, `yay`).
Qualquer comando no terminal avisa, uma vez por dia, quando há versão nova. Detalhes em
[Comandos › update](comandos.md#automacli-update).

## Configuração global

Na primeira execução o automacli cria `~/.config/automacli/config.json` (ou `$XDG_CONFIG_HOME/automacli/config.json`). Esse arquivo guarda o que vale para a **sua máquina**, independentemente do repositório.

```json
{
  "githubToken": "ghp_xxx",
  "clonePath": "/home/voce/tasks"
}
```

| Campo | Padrão | Para que serve |
|---|---|---|
| `cloneStrategy` | `auto` | Como as tasks nascem do repo base: `auto`, `snapshot`, `reflink`, `git-reference` ou `git`. Veja [Requisitos](requisitos.md#como-as-tasks-sao-criadas). |
| `notifications` | `true` | Aviso de desktop do monitor quando um serviço morre por falta de memória. |
| `githubToken` | — | Token usado para consultar o repositório e os pull requests. Necessário para repositórios privados e para não bater no limite de requisições sem autenticação. |
| `clonePath` | `~/tasks` | Diretório onde as tasks são criadas. |
| `packageFilter` | — | Regex padrão do `automacli outdated` (ex.: `^@minha-org`). |
| `scanDirectories` | — | Diretórios esperados pelo `automacli outdated`. |
| `setup.repos` | `[]` | Repositórios locais percorridos por `automacli setup all`. |
| `setup.branchOverrides` | `{}` | Trecho do caminho → branch principal, quando ela não é a de `origin/HEAD`. |
| `verdaccioStoragePath` | `~/.local/share/verdaccio/storage` | Armazenamento limpo por `automacli cleanup --verdaccio*`. |

O token precisa de leitura no repositório (escopo `repo` em tokens clássicos). Ele fica em texto puro nesse arquivo: mantenha-o fora de repositórios e backups compartilhados.

## Estrutura do diretório de configuração

```text
~/.config/automacli/
├── config.json                    # configuração global (acima)
├── repos/
│   └── <owner>-<repo>.yml         # perfil de cada repositório
└── templates/
    └── <owner>-<repo>/            # arquivos injetados em cada task desse repo
        ├── .env.template
        └── scripts/algo.sh.template
```

* **Perfis** descrevem como isolar um repositório. Veja [Perfil do repositório](perfil.md).
* **Templates** são arquivos copiados para dentro de cada task logo após o clone, no mesmo caminho relativo, com o sufixo `.template` removido. Servem para o que não é versionado: `.env` com seus segredos, scripts pessoais. Arquivos `.sh` ficam executáveis.

### Placeholders nos templates

Um template pode conter `{{placeholders}}`, trocados ao copiar para a task:

| Placeholder | Valor | Exemplo no slot 4 |
|---|---|---|
| `{{slot}}` | Número do slot | `4` |
| `{{slotSuffix}}` | `-slot<N>`; vazio no slot 0 (repo base) | `-slot4` |
| `{{queueSuffix}}` | Sufixo das filas SQS ([`queues.suffix`](perfil.md#queues)); vazio no slot 0 ou sem ElasticMQ compartilhado | `-slot4` |
| `{{composeProject}}` | Projeto compose da task | `minha-api-slot4` |
| `{{branch}}`, `{{owner}}`, `{{repo}}` | Branch e repositório | `feat/x`, `minha-org`, `minha-api` |

É assim que um ElasticMQ compartilhado recebe filas por task: `SQS_SUFFIX={{queueSuffix}}` no `.env.template` e, num
arquivo que liste URLs de fila, `…/gerar-pagamento{{queueSuffix}}.fifo`. Arquivos binários são copiados sem
alteração, e um `{{nome}}` desconhecido fica como está.

## Primeiro uso

```sh
automacli doctor             # o que esta máquina oferece e o que o automacli vai usar
cd ~/projetos/minha-api      # um checkout qualquer do repositório
automacli init               # configura a máquina (1ª vez) e grava o perfil do repositório
automacli profile            # mostra o perfil em vigor e as portas por slot
```

Enquanto não houver token gravado (normalmente na primeira execução), o `init` pergunta o diretório das tasks, o token do GitHub, a estratégia de clone e, se a máquina tem `notify-send`, se quer notificações, e grava tudo no `config.json`; `automacli init --global` repete só essa parte. Em seguida ele lê os arquivos compose da raiz, sugere quais serviços são compartilhados (bancos, caches, filas) e grava `~/.config/automacli/repos/<owner>-<repo>.yml`. Com `--local` ele grava `.automacli.yml` na raiz do repositório, para o time versionar um perfil comum.

Depois disso:

```sh
automacli clone minha-feature -f minha-org/minha-api
```
