# Revisor de pull requests

Workflow reutilizable que lee un pull request, lo revisa con [OpenRouter](https://openrouter.ai) y publica el comentario en el pull request. El código vive en [parsel-dev/ai-pr-review](https://github.com/parsel-dev/ai-pr-review). Cada repositorio solo agrega un YAML que lo llama.

## Cómo usarlo

Crea `.github/workflows/ai-pr-review.yml` en el repositorio que quieres revisar:

```yaml
name: AI PR review

on:
  pull_request_target:
    types: [opened, synchronize, ready_for_review, reopened]
  issue_comment:
    types: [created]

permissions:
  contents: read
  pull-requests: write

jobs:
  review:
    if: >-
      ${{
        (
          github.event_name == 'pull_request_target'
          && github.event.pull_request.draft == false
          && github.event.pull_request.head.repo.full_name == github.repository
        )
        ||
        (
          github.event_name == 'issue_comment'
          && github.event.issue.pull_request
          && github.event.comment.user.type != 'Bot'
          && (
            github.event.comment.author_association == 'OWNER'
            || github.event.comment.author_association == 'MEMBER'
            || github.event.comment.author_association == 'COLLABORATOR'
          )
          && startsWith(github.event.comment.body, '/review')
          && !startsWith(github.event.comment.body, '/reviewer')
          && !startsWith(github.event.comment.body, '/reviews')
        )
      }}
    uses: parsel-dev/ai-pr-review/.github/workflows/ai-pr-review.yml@v1
    secrets:
      OPENROUTER_API_KEY: ${{ secrets.OPENROUTER_API_KEY }}
```

Ese `if` es de cada repositorio. El ejemplo revisa un pull request listo y, si un owner, member o collaborator comenta `/review`, lo vuelve a revisar. Puedes quitar el comentario, cambiar el comando o limitar quién lo dispara. Este repositorio no lee el texto del comentario: si el workflow lo llama, revisa.

Los permisos del YAML hacen falta para leer el pull request y publicar la revisión. El workflow compartido tampoco revisa borradores ni pull requests cuyo head venga de otro repositorio.

## Secret

En el repositorio consumidor, o en la organización, crea el secret `OPENROUTER_API_KEY` con una clave de OpenRouter.

## Modelo

El modelo por defecto es `x-ai/grok-4.7`. Para usar otro:

```yaml
jobs:
  review:
    uses: parsel-dev/ai-pr-review/.github/workflows/ai-pr-review.yml@v1
    with:
      model: otro/modelo
    secrets:
      OPENROUTER_API_KEY: ${{ secrets.OPENROUTER_API_KEY }}
```

## Contexto del repositorio

Opcional. En la rama base, `.github/ai-pr-review.md` agrega reglas de ese proyecto al prompt. Si el archivo no existe, la revisión usa solo el prompt genérico.

```markdown
---
omit_prefixes: src/staticfiles/, dist/
---
Cada stage desplegado es una tienda. settings.SHOP es normal.
```

`omit_prefixes` son prefijos separados por comas. Un path que empiece así no se manda al modelo. Para un directorio, incluye la barra final (`src/staticfiles/`).

El archivo se lee de la rama base, no del head. Un pull request no puede cambiar las reglas hasta que se mezcla.

## Actualizar todos los repositorios

La referencia `@v1` es una etiqueta móvil. Moverla en este repositorio actualiza a quien la use:

```bash
git tag -f v1
git push -f origin v1
```

`@main` toma cada push a `main` en cuanto ocurre. Un SHA fijo no cambia hasta que cada repositorio actualice su YAML.

Este repositorio tiene que ser público para que los demás puedan usar `uses:`.
