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

1. **Checkout code**: Realiza un checkout del código fuente del repositorio.

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

2. **Extractor del nombre de un fichero JSON**: Extrae el nombre de un archivo JSON especificado.

```yaml
- name: Extractor del nombre de un fichero JSON
  id: name_extractor
  uses: ./.github/actions/extractor
  with:
    file_path: ${{ vars.PACKAGE_JSON_PATH }}
    field_name: name
```

3. **Llamar a la acción que crea el nombre de la imagen**: Genera el nombre de la imagen utilizando los valores extraídos y los nombres de rama.

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

4. **Setear el valor del nombre de la imagen como output**: Establece el nombre de la imagen como un output del job.

```yaml
- id: set_output
  name: Setear el valor del nombre de la imagen como output
  run: |
    echo "image_name=${{ steps.tagger.outputs.image_name }}" >> $GITHUB_OUTPUT
```

5. **Hacer login en DockerHub**: Se autentica en DockerHub utilizando las credenciales almacenadas en los secretos del repositorio.

```yaml
- name: Hacer login en DockerHub
  uses: docker/login-action@v3
  with:
    username: ${{ vars.DOCKERHUB_USER }}
    password: ${{ secrets.DOCKERHUB_PASS }}
```

6. **Construir imagen**: Construye la imagen Docker de la aplicación Angular.

```yaml
- id: image_creation
  name: Construir imagen
  run: |
    docker build -t ${{ steps
```

7. **Subir imagen a Docker Hub**: Sube la imagen creada a Docker Hub.

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


# GitHub Actions Workflow: **Modificación del versionado**

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


# GitHub Actions Workflow: **Despliegue de la Aplicación**

Este workflow se encarga del despliegue de una aplicación Docker en un entorno especificado. Se activa mediante una llamada de otro workflow y permite gestionar la implementación de la imagen Docker.

## Disparadores

El workflow se activa mediante el evento `workflow_call`, lo que permite su invocación desde otros workflows. Se definen las entradas necesarias:

```yaml
on:
  workflow_call:
    inputs:
      image_name:
        required: true
        type: string
      env_name:
        required: true
        type: string
```

### Jobs

1. **deploy**  
   Este job se encarga de desplegar la imagen de la aplicación en un entorno de Ubuntu.

```yaml
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.env_name }}
```

### Pasos del Job Deploy

1. **Mostrar entorno**: Imprime el nombre del entorno en el que se está realizando el despliegue. Esto ayuda a verificar que se está en el entorno correcto antes de proceder.

```yaml
- name: show env
  run: |
    echo "Este es el entorno ${{ inputs.env_name }}"
´´´

2. **Pull image**: Descarga la imagen Docker especificada utilizando el nombre de la imagen proporcionado como entrada.

```yaml
- name: Pull image
  run: |
    docker pull ${{ inputs.image_name }}
```

3. **Run container**: Inicia un contenedor en segundo plano a partir de la imagen descargada y asigna el puerto 8080 del contenedor al puerto 8080 de la máquina anfitriona.

```yaml
- name: Run container
  run: |
    docker run -d -p 8080:8080 --name github-weekly ${{ inputs.image_name }}
``` 

4. **Mostrar contenedores**: Muestra la lista de contenedores en ejecución para confirmar que el contenedor ha sido iniciado correctamente.

```yaml
- name: Show containers
  run: |
    docker ps
```

5. **Verificación del funcionamiento de la app**: Realiza una solicitud `curl` a la aplicación en el puerto 8080 para verificar que está funcionando correctamente.

```yaml
- name: Verificación del funcionamiento de la app
  run: |
    curl http://localhost:8080
```

### Notas Adicionales

- **Entradas**: Este workflow requiere las entradas `image_name` (nombre de la imagen Docker) y `env_name` (nombre del entorno), lo que permite personalizar el proceso de despliegue según las necesidades específicas.

- **Contenedor en Segundo Plano**: Al ejecutar el contenedor en segundo plano, se garantiza que el flujo de trabajo continúe ejecutándose sin esperar a que el contenedor termine.

- **Verificación**: La verificación mediante `curl` es un paso importante para asegurar que la aplicación se haya desplegado correctamente y esté respondiendo a las solicitudes.


## Custom actions

# GitHub Action: **Extractor Action**

Esta acción compuesta se encarga de extraer el valor de un campo específico de un archivo JSON. Es útil para recuperar datos configurables desde archivos de configuración en formato JSON.

## Descripción

- **Descripción**: Esta acción busca un campo en un archivo JSON y devuelve su valor.
- **Tipo**: Acción compuesta

## Entradas

La acción recibe las siguientes entradas:

```yaml
inputs:
  field_name:
    description: 'Campo a buscar'
    required: true
    default: 'default'
  file_path:
    description: 'Ubicación del fichero json'
    required: true
    default: 'default'
```

### Entradas

- **field_name**: (Requerido) El nombre del campo que se desea buscar en el archivo JSON.
- **file_path**: (Requerido) La ruta al archivo JSON desde el que se desea extraer el valor.

### Salidas

La acción proporciona una salida que contiene el valor del campo extraído:

```yaml
outputs:
  field_value:
    description: 'Valor del campo a buscar en el fichero'
    value: ${{ steps.extractor.outputs.field_value }}
```

Ejecución
La acción se ejecuta utilizando un script en bash que realiza los siguientes pasos:

```yaml
runs:
  using: "composite"
  steps:
    - id: extractor
      name: version_extractor
      run: |
        field_value=$(grep -o '"${{ inputs.field_name }}": *"[^"]*"' ${{ inputs.file_path }} | sed 's/"${{ inputs.field_name }}": "//;s/"//g')
        echo "field_value=${field_value}" >> $GITHUB_OUTPUT
        echo "Este es el nombre de la app $field_value"
      shell: bash
```

### Pasos del Script

1. **Buscar el Campo**: Utiliza `grep` para buscar el campo especificado en el archivo JSON.

2. **Extraer el Valor**: Utiliza `sed` para limpiar la salida y obtener solo el valor del campo.

3. **Establecer la Salida**: Guarda el valor extraído en la variable de salida `field_value`, que se puede utilizar en otros pasos de los workflows.

4. **Imprimir el Valor**: Muestra el valor extraído en la consola para referencia.

### Notas Adicionales

- **Uso de grep y sed**: Esta acción depende de herramientas de línea de comandos como `grep` y `sed`, que son comunes en entornos basados en Unix.

- **Valores por Defecto**: Aunque se proporcionan valores por defecto para las entradas, es recomendable especificar siempre los valores al utilizar la acción.

- **Errores en el Archivo**: Si el archivo JSON no existe o el campo no se encuentra, el comportamiento de la acción dependerá de la configuración del entorno y la gestión de errores en el script.


# GitHub Action: **Tagger Action**

Esta acción compuesta se encarga de etiquetar una imagen de Docker con el formato adecuado, dependiendo de la rama desde la que se ejecute. Es útil para establecer un esquema de nomenclatura consistente para las imágenes de la aplicación.

## Descripción

- **Descripción**: Esta acción genera el nombre de la imagen de Docker basada en el nombre de usuario, el nombre de la aplicación, la versión de la aplicación y la rama de ejecución.
- **Tipo**: Acción compuesta

## Entradas

La acción recibe las siguientes entradas:

```yaml
inputs:
  user_name:
    description: 'Nombre del usuario'
    required: true
    default: 'default'
  app_version:
    description: 'Versión de la aplicación'
    required: true
    default: 'default'
  app_name:
    description: 'Nombre de la aplicación'
    required: true
    default: 'default'
  branch_name:
    description: 'Nombre de la rama de ejecución'
    required: true
    default: 'main'
```


- **user_name**: (Requerido) El nombre del usuario de Docker Hub.
  
- **app_version**: (Requerido) La versión de la aplicación que se está etiquetando.
  
- **app_name**: (Requerido) El nombre de la aplicación.
  
- **branch_name**: (Requerido) El nombre de la rama de ejecución que se está utilizando.

### Salidas

La acción proporciona una salida que contiene el nombre completo de la imagen:

```yaml
outputs:
  image_name:
    description: 'Nombre de la imagen completa'
    value: ${{ steps.image_name_creator.outputs.image_name }}
```

### Salidas

- **image_name**: Contiene el nombre de la imagen de Docker que se ha creado en base a las entradas.


### Ejecución

La acción se ejecuta utilizando un script en bash que realiza los siguientes pasos:

```yaml
runs:
  using: "composite"
  steps:
    - id: image_name_creator
      name: Extraer información sobre la versión del archivo JSON
      run: |
        if [[ ${{ inputs.branch_name }} = 'main' ]]; then
          image_name="${{ inputs.user_name }}/${{ inputs.app_name }}:${{ inputs.app_version }}"
        elif [[ ${{ inputs.branch_name }} = 'develop' ]]; then
          image_name="${{ inputs.user_name }}/pre_${{ inputs.app_name }}:${{ inputs.app_version }}"
        fi
        
        echo "Esta es la imagen desde el action: ${image_name}"
        
        echo "image_name=${image_name}" >> $GITHUB_OUTPUT
      shell: bash
```

### Pasos del Script

1. **Comprobación de la Rama**: Verifica si la rama de ejecución es `main` o `develop`.

2. **Generación del Nombre de la Imagen**:
   - Si la rama es `main`, el nombre de la imagen tendrá el formato: `user_name/app_name:app_version`.
   - Si la rama es `develop`, el nombre tendrá el formato: `user_name/pre_app_name:app_version`.

3. **Imprimir el Nombre de la Imagen**: Muestra el nombre generado en la consola para referencia.

4. **Establecer la Salida**: Guarda el nombre generado en la variable de salida `image_name` para su uso en otros pasos de los workflows.


### Notas Adicionales

- **Formato de Nombres**: La diferenciación entre ramas permite tener imágenes etiquetadas correctamente, lo que es útil para implementar entornos de prueba (prefijados con `pre_`).

- **Valores por Defecto**: Se proporcionan valores por defecto para las entradas, pero es recomendable especificar siempre los valores al utilizar la acción.

- **Errores Potenciales**: Si se proporciona un nombre de rama no válido, el nombre de la imagen no se generará correctamente. Asegúrate de validar las entradas antes de llamar a esta acción.


# Explicación de los workflowa

## Workflows

### UAT

Este workflow esta diseñado con ciertas variantes ya de tratarse del entorno de UAT, en este entorno es en donde tienen lugar los cambios en el desarrollo de la apliación, con lo cual es un entorno que podriamos denominar como  pre-producción, no se van a ejecutar test de código y no necesita de aprobadores de entorno para ninguno de sus jobs.
En este entorno se va a producir las modificaciones de la versión de la app cuando la aplicación haya sido construida y subida a DockerHub sin errores.
Esta aplicación va a ser subida a DockerHub con este formato <user_name>/pre_<app_name>:<tag_name or version_name>.

### Production

Este workflow esta diseñado con ciertas variantes ya de tratarse del entorno de Producción, en este entorno es en donde tienen lugar los test y los despliegues de la aplicación que ya esta funcionando en el entorno de UAT (pre-producción), para asi tener un entorno de producción real siempre y cuando el entorno de pre-producción este funionando desplegado de manera óptima.
En este entorno se va a producir las modificaciones de la versión de manera que va a coger la versión que se tenga en los archivos JSON de la verión de UAT la cual este funcionando de manera óptima desplegada en la red.
Este entrono va a necesitar de la aprobación de desarrolladores seniors en cada uno de los jobs que se vayan a ejecutar.
Esta aplicación va a ser subida a DockerHub con este formato <user_name>/<app_name>:<tag_name or version_name>.




