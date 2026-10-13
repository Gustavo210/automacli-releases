# automacli

Várias branches do mesmo repositório rodando lado a lado, cada uma com sua stack Docker isolada.

Este repositório hospeda a **documentação** e os **releases** do automacli. O código-fonte fica em
[`Gustavo210/automacli`](https://github.com/Gustavo210/automacli).

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
| Debian, Ubuntu e derivados | `sudo apt install ./automacli_<versão>_amd64.deb` |
| Fedora, RHEL, openSUSE | `sudo dnf install ./automacli-<versão>-1.x86_64.rpm` |
| Arch, Manjaro | `PKGBUILD` do pacote `automacli-bin`, com `makepkg -si` |
| Desinstalar | `rm -rf ~/.local/share/automacli ~/.local/bin/automacli` |

Os arquivos de cada release, os checksums (`SHA256SUMS`) e as notas estão na página de
[Releases](https://github.com/Gustavo210/automacli-releases/releases). O `install.sh` confere o SHA-256 antes de
instalar.
