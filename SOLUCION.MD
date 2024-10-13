# GitHub Actions Workflow: **Construcción y Despliegue de App**

Este workflow automatiza el proceso de construcción y despliegue de la aplicación en los entornos **production** o **UAT**, dependiendo de la rama en la que se realice la acción (`main` o `develop`).

## Disparadores

El workflow se ejecuta cuando hay un push en las siguientes ramas:
- `main`
- `develop`

```yaml
on:
  push:
    branches:
      - main
      - develop
``` 

Este formato usa los bloques de código en Markdown para presentar el código YAML. Si necesitas más ayuda o personalización, avísame.

```yaml
permissions:
  contents: write
```

### Concurrencia

Se configura la ejecución concurrente para evitar que el mismo flujo de trabajo se ejecute en paralelo para la misma referencia de rama. Si se realiza un nuevo push antes de que termine la ejecución actual, esta será cancelada.

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

### Jobs

1. **get_branch**  
   Este job obtiene el nombre de la rama en la que se está ejecutando la acción y lo almacena en la variable `BRANCH_NAME`.

```yaml
jobs:
  get_branch:
    outputs:
      BRANCH_NAME: ${{ steps.get_branch.outputs.BRANCH_NAME }}
    runs-on: ubuntu-latest
    steps:
      - id: get_branch
        name: Obtener el nombre de la rama donde se está realizando la acción
        run: |
          echo "BRANCH_NAME=$(basename $GITHUB_REF)" >> $GITHUB_OUTPUT
```


2. **set_environment**  
   Este job establece el entorno (`production` o `UAT`) en función de la rama donde se realiza la acción. Si la rama es `main`, el entorno será `production`. Si es `develop`, el entorno será `UAT`.

```yaml
  set_environment:
    outputs:
      ENV_NAME: ${{ steps.set_env.outputs.ENV_NAME }}
    runs-on: ubuntu-latest
    needs: get_branch
    steps:
      - id: set_env
        name: Setear el valor del environment en función de la rama donde se realiza la acción
        run: |
          if [ "${{ needs.get_branch.outputs.BRANCH_NAME }}" = "main" ]; then
            echo "ENV_NAME=production" >> $GITHUB_OUTPUT
          elif [ "${{ needs.get_branch.outputs.BRANCH_NAME }}" = "develop" ]; then
            echo "ENV_NAME=UAT" >> $GITHUB_OUTPUT
          fi
```

3. **call_workflow_CI**  
Este job llama al workflow de integración continua (CI) definido en `.github/workflows/ci_workflow.yaml`. Se le pasan los valores de `env_name` y `branch_name` obtenidos en los jobs previos.

```yaml
  call_workflow_CI: 
    needs: [set_environment, get_branch]
    uses: ./.github/workflows/ci_workflow.yaml
    secrets: inherit
    with:
      env_name: ${{ needs.set_environment.outputs.ENV_NAME }}
      branch_name: ${{ needs.get_branch.outputs.BRANCH_NAME }}
```

4. **call_upload_version**  
   Este job actualiza la versión de la app en un archivo JSON, pero solo si la acción se realiza en la rama `develop`. Utiliza el workflow ubicado en `.github/workflows/update_app_version_on_json.yaml`.

```yaml
  call_upload_version: 
    needs: [set_environment, get_branch, call_workflow_CI]
    if: ${{ needs.get_branch.outputs.BRANCH_NAME == 'develop' }}
    uses: ./.github/workflows/update_app_version_on_json.yaml
    secrets: inherit
    with:
      env_name: ${{ needs.set_environment.outputs.ENV_NAME }}
      branch_name: ${{ needs.get_branch.outputs.BRANCH_NAME }}

```

5. **call_workflow_CD**  
   Este job llama al workflow de despliegue (CD) definido en `.github/workflows/cd_workflow.yaml`. Se utilizan la imagen generada en el job de CI y los valores de entorno establecidos previamente para el despliegue en el entorno correspondiente.

```yaml
  call_workflow_CD: 
    needs: [set_environment, call_workflow_CI]
    uses: ./.github/workflows/cd_workflow.yaml
    secrets: inherit
    with:
        image_name: ${{ needs.call_workflow_CI.outputs.image_name }}
        env_name: ${{ needs.set_environment.outputs.ENV_NAME }}
```
[!WARNING]
Notas Adicionales

- **Secrets**: Los secretos se heredan en cada job que utiliza otros workflows mediante el parámetro `secrets: inherit`.

- **Condicionales**: El job `call_upload_version` solo se ejecuta si la rama en la que se realizó la acción es `develop`.

```yaml
if: ${{ needs.get_branch.outputs.BRANCH_NAME == 'develop' }}
```
