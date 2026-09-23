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
          && startsWith(github.event.comment.body, '/review')
        )
      }}
    uses: parsel-dev/ai-pr-review/.github/workflows/ai-pr-review.yml@1.0.1
    with:
      omit_prefixes: generated/, vendor/
    secrets:
      OPENROUTER_API_KEY: ${{ secrets.OPENROUTER_API_KEY }}
```

Ese `if` es de cada repositorio. El ejemplo revisa un pull request listo y, si alguien que no es un bot comenta `/review`, lo vuelve a revisar. Puedes quitar el comentario, cambiar el comando o limitar quién lo dispara. Este repositorio no lee el texto del comentario: si el workflow lo llama, revisa.

`omit_prefixes` es opcional. El script lo lee y deja fuera del diff cualquier path que empiece así; el modelo no ve esos archivos ni el valor del input. Varios prefijos van separados por comas (`generated/, vendor/`). Para un directorio, cierra con `/`.

Los permisos del YAML hacen falta para leer el pull request y publicar la revisión. El workflow compartido tampoco revisa borradores ni pull requests cuyo head venga de otro repositorio.

## Secret

En el repositorio consumidor, o en la organización, crea el secret `OPENROUTER_API_KEY` con una clave de OpenRouter.

## Modelo

El modelo por defecto es `x-ai/grok-4.7`. Para usar otro:

```yaml
jobs:
  review:
    uses: parsel-dev/ai-pr-review/.github/workflows/ai-pr-review.yml@1.0.1
    with:
      model: otro/modelo
    secrets:
      OPENROUTER_API_KEY: ${{ secrets.OPENROUTER_API_KEY }}
```

## Contexto del repositorio

Opcional. En la rama base, `.github/ai-pr-review.md` es texto para el modelo: reglas de ese proyecto. Si el archivo no existe, la revisión usa solo el prompt genérico. Se lee de la rama base, no del head, así que un pull request no cambia esas reglas hasta que se mezcla.

Ejemplo de `.github/ai-pr-review.md`:

```markdown
Los archivos en `generated/` salen del build.
Un valor de configuración fijo de este repositorio es intencional.
```
