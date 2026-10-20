# pr-automation

Workflows reutilizáveis de GitHub Actions para uma esteira de PR com merge automático:

1. Um PR de branch de trabalho (`feature/*`, `feat/*`, `fix/*`, `hotfix/*`, `chore/*`, `ci/*`, `docs/*`, `refactor/*`, `release/*` e afins) para a branch de integração (ex. `develop`) recebe merge automático por squash assim que os checks obrigatórios passam. Branches de sincronização (`chore/sync-*`, `sync/*`, `backmerge/*`) entram com merge commit, para manter a ancestralidade.
2. Cada push na branch de integração abre, ou reaproveita, o PR integração → promoção (ex. `develop` → `homolog`) e liga o merge automático com merge commit quando os checks dele passam.

Os merges usam o token de um GitHub App (`actions/create-github-app-token`), não o `GITHUB_TOKEN`. Um merge feito com o `GITHUB_TOKEN` não dispara outros workflows, e a cadeia merge → PR de promoção → checks → merge → deploy pararia no primeiro passo.

## Como usar

1. Instale o GitHub App na conta ou organização. Ele precisa de leitura e escrita em contents, pull requests, actions e workflows, e de leitura em checks, statuses e metadata.
2. Disponibilize a variável `AUTOMATION_APP_ID` e o segredo `AUTOMATION_APP_PRIVATE_KEY`, de organização ou de repositório.
3. Ligue "Allow auto-merge" no repositório e proteja as branches de integração e de promoção com os checks de CI obrigatórios. Sem checks obrigatórios, o GitHub recusa o merge automático.
4. Copie [`examples/caller-develop-homolog.yml`](examples/caller-develop-homolog.yml) para `.github/workflows/pr-flow.yml` e ajuste as entradas.

### Entradas

| Entrada | Padrão | Descrição |
| --- | --- | --- |
| `app_id` | — | ID do GitHub App |
| `integration_branch` | — | Branch que recebe as branches de trabalho |
| `promotion_branch` | vazio | Branch que recebe a integração; vazio desliga a promoção |
| `work_branch_pattern` | `^(feature\|feat\|fix\|bugfix\|hotfix\|chore\|ci\|docs\|refactor\|perf\|test\|build\|style\|revert\|deps\|release)/` | Regex das branches com merge automático |
| `merge_method` | `squash` | Método do merge das branches de trabalho |
| `merge_commit_pattern` | `^(chore/sync-\|sync/\|back-?merge/)` | Regex das branches que entram com merge commit |

Para repositórios só com `main`, use `integration_branch: main` e deixe `promotion_branch` vazio.

Para fixar versões, use a tag `v1` (`@v1`).

### PR que já nasce com os checks verdes

Todo PR de promoção ou de release aponta para um SHA que já passou pelo CI do push na branch de origem. Nesse caso o GitHub recusa ligar o merge automático (`Pull request is in clean status`) ou tenta o merge na hora e o desliga em seguida (`Repository rule violations found`), quando o CI do `pull_request` entra pendente. Os workflows conferem o estado depois de ligar: se o PR está `CLEAN`, fazem o merge direto; se está `BLOCKED`, ligam de novo, até o merge automático ficar ligado e estável ou o PR ser mergeado. O App não é ator de bypass, então o merge direto também exige os checks obrigatórios.

## Release para produção

`.github/workflows/release.yml` sobe a branch validada (padrão `homolog`) para a de produção (padrão `main`) sob demanda: copie `examples/caller-release.yml` para o repositório e rode **Actions → release → Run workflow**. O workflow abre (ou reaproveita) o PR `homolog → main` e liga o merge automático com merge commit. O merge espera os checks obrigatórios de `main`, e o deploy de produção segue do push em `main`.

- Recusa quando `main` não exige checks, para nunca ir a produção sem CI.
- Avisa quando `main` tem commits fora de `homolog`: eles entram no merge, mas devem voltar para `homolog`.
