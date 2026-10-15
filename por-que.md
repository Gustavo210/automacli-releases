---
url: https://gustavo210.github.io/automacli-releases/por-que.md
---

# Por que o automacli?

Quem precisa de várias branches no ar ao mesmo tempo costuma chegar em uma destas três respostas: um checkout só com
scripts à mão, `git worktree`, ou uma ferramenta que cuide do ambiente inteiro. As duas primeiras resolvem a parte do
**código**. O problema de verdade está no que fica em volta do código: dependências instaladas, portas publicadas,
bancos com dados, arquivos que o git não versiona e o custo de limpar tudo depois.

É esse "em volta" que o automacli automatiza.

## O que uma branch precisa para rodar

Uma branch de um projeto com Docker não é só a árvore de arquivos. Para ela subir, é preciso:

| Peça | Exemplo | De onde vem |
|---|---|---|
| **Código** | a branch `feat/pagamentos` | git |
| **Dependências** | `node_modules`, `vendor`, `.venv` | `pnpm install`, `composer install`, `pip install` |
| **Portas livres** | `3000`, `4000`, `8080` | ninguém mais pode estar usando |
| **Serviços com estado** | Postgres, Redis, OpenSearch, SQS | um compose, volumes, seeds |
| **Arquivos não versionados** | `.env`, scripts pessoais, instruções do agente | você, de novo, em cada cópia |
| **Limpeza** | parar containers, apagar volumes, remover o diretório | você, quando lembrar |

Cada abordagem abaixo cobre uma fatia diferente dessa tabela.

## Um checkout e scripts manuais

É o ponto de partida de quase todo time: um clone, `git checkout` para trocar de branch e, com o tempo, um
`clone-task.sh` que copia o repositório e troca as portas com `sed`.

O que funciona:

* Não exige nada além de git e do próprio compose.
* Um script simples resolve o primeiro caso de uso em uma tarde.

Onde quebra:

* **Portas na mão.** O script precisa saber todas as portas publicadas pela stack e escolher um deslocamento que não
  colida com nenhuma outra cópia. Com `extends`, `include` e variáveis no compose, a lista muda sem avisar. O automacli
  lê as portas com `docker compose config` e reserva um [slot](como-funciona.md#_2-slot) livre, pulando os que colidiriam com
  outra task ([port is already allocated](solucao-de-problemas.md#port-is-already-allocated-ao-subir-a-stack)).
* **Edita o que é do repositório.** Trocar portas com `sed` nos arquivos compose e no `.env` suja o `git status` e
  vira conflito no próximo rebase. O automacli gera um [override](como-funciona.md#_4-override) ao lado e o esconde
  em `.git/info/exclude`.
* **Uma instalação por cópia.** Cada clone novo repete `pnpm install`, `composer install` e o download de imagens.
  O automacli copia a task a partir do repo base por [snapshot ou reflink](requisitos.md): segundos, com
  `node_modules` e `vendor` já no lugar e sem gastar disco até os arquivos divergirem.
* **Um banco por cópia, ou um banco brigando com o outro.** Ou cada cópia sobe o seu Postgres (e cada um precisa de
  porta, volume e seed), ou todas apontam para o mesmo e a primeira a dar `docker compose down` derruba o das outras. Os
  [serviços compartilhados](como-funciona.md#_5-servicos-compartilhados) rodam uma vez, a partir do repo base, e,
  no modo padrão, nenhuma task consegue derrubá-los.
* **O script é de uma pessoa.** Ele conhece um repositório, uma máquina e um jeito de trabalhar. Quando alguém novo
  entra, recebe o script por mensagem e descobre as pegadinhas sozinho. No automacli o que varia entre projetos fica em
  um [perfil YAML](perfil.md) versionável; o resto é descoberto.
* **Ninguém sabe o que está no ar.** Sem um lugar que liste as cópias, os slots e o consumo de cada uma, a máquina
  acumula stacks esquecidas. O [monitor](monitor.md) mostra cada task com o estado do PR, CPU, memória e disco real, e
  `automacli branches` remove uma task derrubando a stack dela antes.

## git worktree

`git worktree add ../feat-pagamentos feat/pagamentos` cria um segundo diretório de trabalho ligado ao mesmo `.git`.
É a resposta certa para a pergunta "como ter duas branches abertas ao mesmo tempo", e o automacli não tenta
substituí-la nesse ponto.

O que funciona:

* Faz parte do git, sem instalar nada.
* A segunda árvore nasce em segundos e compartilha objetos, refs e stashes com a primeira.
* Perfeito para olhar um PR, fazer um cherry-pick ou comparar dois estados do código.

Onde para:

* **Só o código vem junto.** `node_modules`, `vendor` e `.venv` não são versionados, então cada worktree começa vazia
  e precisa de um install. Com pnpm e o store quente ele é barato em disco (hard links) e leva o tempo de linkar os
  arquivos; com Composer, pip, npm ou yarn clássico é uma cópia inteira dos pacotes. A task do automacli nasce de um
  snapshot do repo base: as dependências já estão lá, em blocos compartilhados.
* **Portas continuam fixas.** O compose de cada worktree publica as mesmas portas; a segunda stack falha com
  `port is already allocated`. Resolver isso é voltar aos scripts manuais do item anterior.
* **Um banco por worktree, ou um banco brigando com o outro.** Cada worktree é um projeto compose com seus volumes,
  então ou cada uma sobe o seu Postgres, ou todas apontam para o mesmo e qualquer `docker compose down` derruba o
  das outras. O automacli roda os serviços com estado uma vez, a partir do repo base, fora da stack de cada task.
* **Arquivos não versionados ficam para trás.** O `.env`, o `CLAUDE.local.md` e o script que você usa todo dia não
  acompanham a worktree. Os [templates](instalacao.md#estrutura-do-diretorio-de-configuracao) do automacli entram em
  toda task no mesmo caminho, automaticamente.

O que o automacli faz a mais, sem que isso seja falha do worktree, que nunca se propôs a cuidar de ambiente: slot e
override gerados, [isolamento](isolamento.md) por slice systemd com teto de memória, e o [monitor](monitor.md)
registrando o que está no ar e quem foi morto por falta de memória. Nos dois casos vale uma branch por vez: o git
recusa a mesma branch em duas worktrees, e o automacli cria um diretório por branch.

Se o seu projeto não tem Docker e as dependências instalam em segundos, `git worktree` basta. O automacli entra
quando a stack em volta do código pesa mais do que o código.

## Lado a lado

| | Checkout + scripts | `git worktree` | automacli |
|---|---|---|---|
| Diretório por branch | o script copia | sim | sim, task por branch |
| Tempo até a branch rodar | clone + install | install | segundos (snapshot ou reflink) |
| Dependências instaladas | de novo em cada cópia | de novo em cada worktree | vêm do repo base |
| Disco por branch | uma cópia inteira | só o código, mais o install | só o que a branch muda |
| Portas | calculadas no script | colidem | slot por task, override gerado |
| Arquivos do repositório | editados com `sed` | intactos | intactos, override fora do git |
| Banco, cache e busca | um por cópia, ou briga | um por worktree | uma instância, compartilhada e protegida |
| `.env` e arquivos pessoais | copiar à mão | copiar à mão | templates em toda task |
| Limite de memória | não | não | slice systemd por slot |
| Visibilidade do que está no ar | `docker ps` | `git worktree list` | monitor HTTP e MCP |
| Limpeza | à mão | `git worktree remove` | `branches` para a stack e apaga a task |
| Configuração por projeto | um script por repositório | nenhuma | um perfil YAML, descoberto quando falta |

## Quando não vale a pena

* O repositório não tem stack Docker e as dependências instalam em segundos: `git worktree` resolve.
* Você trabalha sempre em uma branch por vez e troca pouco: um checkout basta.
* O disco das tasks não suporta reflink nem btrfs: o automacli continua funcionando, caindo para `git clone --reference` (e, sem repo base, `git clone`), mas a
  vantagem de tempo e disco diminui. Veja o que a sua máquina tem com `automacli doctor` em
  [Requisitos e ambiente](requisitos.md).

## Para seguir

* [Como funciona](como-funciona.md): slots, portas, override e serviços compartilhados em detalhe.
* [Instalação](instalacao.md): o executável pronto e os pacotes por distribuição.
* [Exemplos](exemplos.md): perfis prontos para copiar.
