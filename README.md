# automacli

Várias branches do mesmo repositório rodando lado a lado, cada uma com sua stack Docker isolada.

Este repositório é a área de distribuição do automacli: a **documentação** e os **releases** prontos para
instalar. O código-fonte é mantido em um repositório privado.

- Documentação: https://gustavo210.github.io/automacli-releases/
- Releases: https://github.com/Gustavo210/automacli-releases/releases
- Para LLMs: [`llms.txt`](https://gustavo210.github.io/automacli-releases/llms.txt) (índice) e [`llms-full.txt`](https://gustavo210.github.io/automacli-releases/llms-full.txt) (documentação inteira em Markdown)

## Instalar

```sh
curl -fsSL https://github.com/Gustavo210/automacli-releases/releases/latest/download/install.sh | bash
```

Instala em `~/.local/share/automacli` e cria `~/.local/bin/automacli`. Não precisa de Node: o executável traz o
próprio. Rodar de novo atualiza para a última versão.

| Quero... | Como |
|---|---|
| Uma versão específica | `curl -fsSL .../install.sh \| bash -s v0.1.1` |
| Debian, Ubuntu e derivados | `sudo curl -fsSLo /usr/share/keyrings/automacli.gpg https://gustavo210.github.io/automacli-releases/apt/automacli.gpg && sudo curl -fsSLo /etc/apt/sources.list.d/automacli.sources https://gustavo210.github.io/automacli-releases/apt/automacli.sources && sudo apt update && sudo apt install automacli` |
| Fedora, RHEL | `sudo curl -fsSLo /etc/yum.repos.d/automacli.repo https://gustavo210.github.io/automacli-releases/rpm/automacli.repo && sudo dnf install automacli` |
| openSUSE | `sudo zypper addrepo -f https://gustavo210.github.io/automacli-releases/rpm/automacli.repo && sudo zypper --gpg-auto-import-keys install automacli` |
| Arch, Manjaro | `yay -S automacli-bin` (AUR) |
| Desinstalar | `rm -rf ~/.local/share/automacli ~/.local/bin/automacli` |

Os repositórios apt e rpm são assinados pela chave `automacli packages` (`1C27 7A0D 3569 3660 D604 431F B108 4A86 A1CD 6EA3`) e atendem `amd64`/`x86_64` e `arm64`/`aarch64`; as atualizações vêm pelo gerenciador de pacotes. Os arquivos de cada release, os checksums (`SHA256SUMS`) e as notas estão na página de
[Releases](https://github.com/Gustavo210/automacli-releases/releases). O `install.sh` confere o SHA-256 antes de
instalar.
