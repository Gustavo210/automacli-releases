---
url: https://gustavo210.github.io/automacli-releases/app.md
---
# App para o telefone (Android)

Um app leve para acompanhar o monitor do automacli pelo celular, **só quando o telefone está no mesmo Wi‑Fi do PC**: widget na tela inicial com os containers no ar, notificação quando um serviço falha e, no Android 16+ (One UI 8+), os containers em execução no Now Bar como Live Update. Nada passa pela nuvem; fora da rede do PC o app não usa rede nem bateria.

## Instalar

1. Baixe o APK: [app-latest em automacli-releases](https://github.com/Gustavo210/automacli-releases/releases/tag/app-latest) (link fixo, atualizado a cada merge na `main` que toque em `android/`). Também fica como artefato de cada execução do workflow *App Android*.
2. Instale pelo navegador do telefone (permita instalar de fontes desconhecidas para o navegador).
3. No PC, com o monitor rodando como serviço (`automacli monitor setup`), pareie:

```sh
automacli monitor pair s24
```

4. Aponte a câmera do telefone para o QR code. O app abre, guarda a URL e o token (só de leitura) e já conecta. Sem QR code, preencha a URL (`http://<ip-do-pc>:7777`) e o token à mão na tela de configurações.

## O que ele faz

| Recurso | Como funciona |
|---|---|
| **Tela principal** | O mesmo formato do painel do PC: disco real, aparente e economia do reflink no topo, busca, e um card por task com slot (`UP S35` quando no ar), disco real e aparente, arquivos não commitados, estado do PR, chips de review e checks, gráfico de CPU e memória ao vivo (a cada 5 s com a tela aberta) e os botões **PR**, **Links**, **Recursos** e **Remover** (só quando o ciclo de vida permite; o motivo aparece embaixo). |
| **Widget** | Três tamanhos: pequeno (containers, stacks, disco real e aparente), médio (mais duas tasks) e grande (até sete tasks com a situação: mergeada, pronta para merge, checks, review; atalho `#PR` que abre o GitHub e botão *recursos* que abre o app na task). Cores do tema do aparelho. Fora da rede mostra o último estado com a idade. |
| **Modo apresentação** | Botão *Apresentar*: tela preta pura para o AMOLED, brilho reduzido, sempre ligada, retrato ou paisagem. O conteúdo é o gráfico de CPU e memória de cada stack no ar ocupando a tela (lado a lado em paisagem, em faixas em retrato), com os valores atuais em números grandes, lido a cada 5 s; as tasks paradas viram só uma contagem. Contra burn‑in, o conteúdo desloca alguns pixels a cada minuto e o relógio muda de canto; sem animações, só recompõe quando chega evento. Toque para sair. |
| **Notificação fixa** | Exigência do Android para a conexão ficar aberta, e útil: "48 containers em 2 stacks" ou "Fora da rede do PC". |
| **Now Bar / Live Update** | No Android 16+ a notificação fixa pede para ser promovida: aparece no Now Bar do One UI com uma barra verde (no ar) e vermelha (com problema). Precisa permitir "notificações ao vivo" para o app nos ajustes do Android. Pode ser desligado no app. |
| **Alertas** | Container em restart ou parado com erro, serviço compartilhado parado, disco btrfs baixo, serviço morto por falta de memória, falha na atualização do repo base. A notificação some sozinha quando o monitor avisa que resolveu. |
| **Catch‑up** | Ao voltar para a rede, o app pede ao monitor o que perdeu desde o último evento; alertas de quando você estava fora chegam nessa hora. |
| **Recursos** | Memória e CPU por stack (soma do `docker stats`) na lista, e por serviço ao tocar na task; disco real em btrfs por task e no total, com o que o reflink poupa. |
| **Links** | Ao tocar na task, todas as URLs dela abrem no navegador do telefone. Nomes como `*.devtest` só resolvem se o DNS do Wi‑Fi do telefone apontar para o PC (o `dnsmasq` compartilhado atende na porta 53). |
| **Ações** | Parar ou iniciar a stack de uma task e subir ou parar os serviços compartilhados, com confirmação. Exige aparelho pareado com escopo `control` (padrão de `automacli monitor pair`); com `--read-only` os botões respondem que o aparelho só lê. |
| **Atualização** | Em Configurações, *Verificar atualização* consulta a release `app-latest` no GitHub (única coisa que usa internet), baixa o APK e abre o instalador. Só atualiza por cima se o APK do CI estiver assinado com a mesma chave (`setup-signing.sh`). |

## Como ele se conecta

O app observa o Wi‑Fi. Em qualquer rede nova, tenta o monitor por 3 segundos; se responde, abre o fluxo de eventos ([`/api/events`](monitor.md#eventos)) e fica nele; se não responde, é outra rede, e ele tenta de novo a cada minuto sem alarde. O tráfego é amarrado à interface Wi‑Fi, então funciona mesmo quando o Wi‑Fi não tem internet e o Android preferiria o 4G. O token é só de leitura ([Acesso do telefone](monitor.md#acesso-do-telefone)).

## Aparelhos pareados

`automacli monitor pair --list` mostra cada aparelho com escopo, último acesso, IP e quantas conexões de eventos estão abertas agora. `--disable <nome>` corta o acesso na hora (as conexões caem e o token é recusado) sem apagar o pareamento; `--enable` reativa; `--revoke` apaga. Detalhes em [Monitor → Acesso do telefone](monitor.md#acesso-do-telefone).

## Samsung (One UI)

* Em **Ajustes > Apps > automacli > Bateria**, escolha *Sem restrições* e tire o app da lista de *apps que dormem*; senão o One UI encerra a conexão depois de alguns dias.
* O app usa dois caminhos para o Now Bar ao mesmo tempo: o Live Update do Android 16 e o que os apps parceiros do Samsung usam (extras `android.ongoingActivityNoti.*` com o `meta-data` `com.samsung.android.support.ongoing_activity`, como o Samsung Health). Em Config há ainda o toggle *Estilo nativo Samsung*, que troca o Live Update pelo estilo oculto `OngoingActivityStyle` do Gravador de Voz; nenhum dos dois é API pública, então o diagnóstico mostra o que o aparelho aceitou.
* Para o Now Bar, dois ajustes: **Ajustes > Notificações > Configurações avançadas > Notificações ao vivo** e permita o automacli; e, como o Now Bar só mostra apps parceiros por padrão, ligue **Opções do desenvolvedor > Notificações ao vivo para todos os apps**. O app tem um diagnóstico em Config (permissão, elegibilidade e se a notificação foi promovida).
* Depois de reiniciar o telefone, o app volta sozinho se estava ligado.

## Compilar

O projeto fica em `android/` (Kotlin, Jetpack Compose, Glance, OkHttp; sem Firebase). Fora do pacote npm. Com o SDK do Android instalado:

```sh
cd android
./gradlew :app:assembleDebug      # app/build/outputs/apk/debug/app-debug.apk
./gradlew :app:assembleRelease    # minificado (~4 MB); sem chave, assinado com a de debug
```

O workflow `.github/workflows/android.yml` compila em todo push ou PR que toque `android/` e, na `main`, publica a release rolante `app-latest`. Para o APK do CI atualizar por cima do instalado, a chave precisa ser sempre a mesma: rode uma vez `android/scripts/setup-signing.sh`, que cria `~/.config/automacli/app-release.jks` e grava os secrets `ANDROID_KEYSTORE_BASE64`, `ANDROID_KEYSTORE_PASSWORD`, `ANDROID_KEY_ALIAS` e `ANDROID_KEY_PASSWORD` no repositório. Guarde o `.jks` e o `.env` ao lado: perdê-los obriga a trocar de chave e reinstalar.
