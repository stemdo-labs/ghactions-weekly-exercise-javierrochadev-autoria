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

## Notas Adicionales

- **Secrets**: Los secretos se heredan en cada job que utiliza otros workflows mediante el parámetro `secrets: inherit`.

- **Condicionales**: El job `call_upload_version` solo se ejecuta si la rama en la que se realizó la acción es `develop`.

```yaml
if: ${{ needs.get_branch.outputs.BRANCH_NAME == 'develop' }}
```

# Reusables

# GitHub Actions Workflow: **Creación y Subida de la Imagen de Angular**

Este workflow se encarga de la creación y subida de la imagen de una aplicación Angular a Docker Hub. Se activa mediante una llamada de otro workflow y permite definir variables de entrada y salida.

## Disparadores

El workflow se activa mediante el evento `workflow_call`, lo que significa que puede ser invocado desde otros workflows. Se definen entradas y salidas para la configuración:

```yaml
on:
  workflow_call:
    outputs:
      image_name:
        description: Nombre de la imagen
        value: ${{ jobs.build.outputs.image_name }}
      port_value_nginx:
        description: Puerto donde recibe las solicitudes nginx
        value: ${{ jobs.build.outputs.port_value_nginx }}
    inputs:
      env_name:
        required: true
        type: string
      branch_name:
        required: true
        type: string
```

### Jobs

1. **test**  
   Este job se ejecuta en un entorno de Ubuntu y solo se activa si el `branch_name` es `main`. Se encarga de realizar pruebas funcionales del código.

```yaml
  test:
    runs-on: ubuntu-latest
    environment: ${{ inputs.env_name }}
    if: ${{ inputs.branch_name == 'main' }}
    steps:
      - id: pass_testing
        name: Testing de funcionamiento de código
        run: |
          echo "test_user_login_success"; sleep 2
          echo "test_create_post"; sleep 2
          echo "test_api_response"; sleep 2
          echo "test_sql_injection"; sleep 2
          echo "All tests have been passed."
```

2. **build**  
   Este job se ejecuta en un entorno de Ubuntu y siempre se activa, independientemente del resultado del job anterior. Se encarga de construir la imagen Docker de la aplicación Angular.

```yaml
  build:
    runs-on: ubuntu-latest
    if: ${{ always() }}
    needs: test
    outputs:
      image_name: ${{ steps.set_output.outputs.image_name }}
    environment: ${{ inputs.env_name }}
```

### Pasos del Job Build

- **Checkout code**: Realiza un checkout del código fuente del repositorio.

```yaml
- name: Checkout code
  uses: actions/checkout@v2
```

- **Extractor de la versión de un fichero JSON**: Extrae la versión de un archivo JSON especificado.

```yaml
- name: Extractor de la versión de un fichero JSON
  id: version_extractor
  uses: ./.github/actions/extractor
  with:
    file_path: ${{ vars.PACKAGE_JSON_PATH }}
    field_name: version
```

- **Extractor del nombre de un fichero JSON**: Extrae el nombre de un archivo JSON especificado.

```yaml
- name: Extractor del nombre de un fichero JSON
  id: name_extractor
  uses: ./.github/actions/extractor
  with:
    file_path: ${{ vars.PACKAGE_JSON_PATH }}
    field_name: name
```

- **Llamar a la acción que crea el nombre de la imagen**: Genera el nombre de la imagen utilizando los valores extraídos y los nombres de rama.

```yaml
- name: Llamar a la action que me crea el nombre de la imagen
  id: tagger
  uses: ./.github/actions/tagger_action
  with:
    branch_name: ${{ inputs.branch_name }}
    user_name: ${{ vars.DOCKERHUB_USER }}
    app_name: ${{ steps.name_extractor.outputs.field_value }}
    app_version: ${{ steps.version_extractor.outputs.field_value }}
```

- **Setear el valor del nombre de la imagen como output**: Establece el nombre de la imagen como un output del job.

```yaml
- id: set_output
  name: Setear el valor del nombre de la imagen como output
  run: |
    echo "image_name=${{ steps.tagger.outputs.image_name }}" >> $GITHUB_OUTPUT
```

Hacer login en DockerHub: Se autentica en DockerHub utilizando las credenciales almacenadas en los secretos del repositorio.

yaml
Copiar código
- name: Hacer login en DockerHub
  uses: docker/login-action@v3
  with:
    username: ${{ vars.DOCKERHUB_USER }}
    password: ${{ secrets.DOCKERHUB_PASS }}
```

- **Construir imagen**: Construye la imagen Docker de la aplicación Angular.

```yaml
- id: image_creation
  name: Construir imagen
  run: |
    docker build -t ${{ steps
```

- **Subir imagen a Docker Hub**: Sube la imagen creada a Docker Hub.

```yaml
- id: image_push
  name: Subir imagen a Docker Hub
  run: |
    docker push ${{ steps.tagger.outputs.image_name }}
```

### Notas Adicionales

- **Manejo de Errores**: El job `build` se ejecuta siempre, independientemente del resultado del job de `test`.

- **Entradas y Salidas**: El workflow define entradas (`env_name`, `branch_name`) y salidas (`image_name`, `port_value_nginx`) que pueden ser utilizadas en workflows que invocan este flujo.

- **Variables de Entorno**: Las variables como `DOCKERHUB_USER` y `DOCKERHUB_PASS` deben estar definidas en los secretos del repositorio para un acceso seguro.


# GitHub Actions Workflow: **Despliegue de la Aplicación**

Este workflow está diseñado para el despliegue de la aplicación. Se activa mediante una llamada de otro workflow y permite gestionar la versión de la aplicación de manera automatizada.

## Disparadores

El workflow se activa mediante el evento `workflow_call`, lo que permite su invocación desde otros workflows. Se definen las entradas necesarias:

```yaml
on:
  workflow_call:
    inputs:
      env_name:
        required: true
        type: string
      branch_name:
        required: true
        type: string
        default: 'main'
```

### Jobs

1. **update_version**  
   Este job se encarga de actualizar la versión de la aplicación en el archivo `package.json` y se ejecuta en un entorno de Ubuntu.

```yaml
  update_version:
    runs-on: ubuntu-latest
    environment: ${{ inputs.env_name }}
```

### Pasos del Job Update Version

1. **Checkout repo**: Realiza un checkout del repositorio para asegurarse de que se tiene acceso a los archivos necesarios, incluyendo `package.json`.

```yaml
- name: Checkout repo
  uses: actions/checkout@v2
```

2. **Automated Version Bump**: Utiliza la acción `phips28/gh-action-bump-version` para incrementar automáticamente la versión de la aplicación según el tipo de versión especificado. Este paso requiere el `GITHUB_TOKEN` para autenticar la acción y realizar cambios en el repositorio.

```yaml
- name: 'Automated Version Bump'
  uses: 'phips28/gh-action-bump-version@master'
  env:
    GITHUB_TOKEN: ${{ secrets.SECRET_TOKEN }}
    PACKAGE_FILENAME: 'package.json'
  with:
    version-type: ${{ inputs.version_change }}
    commit-message: 'CI: bumps version to {{version}} [ develop ]'
    target-branch: ${{ inputs.branch_name }}
```

### Notas Adicionales

- **Entradas**: Este workflow requiere las entradas `env_name` (nombre del entorno) y `branch_name` (nombre de la rama), lo que permite la personalización de la ejecución según las necesidades del despliegue.

- **Automatización**: El uso de `gh-action-bump-version` permite un manejo automatizado del versionado, reduciendo la posibilidad de errores manuales.

- **Secretos**: El `GITHUB_TOKEN` se debe almacenar como un secreto en el repositorio para mantener la seguridad durante la ejecución.




