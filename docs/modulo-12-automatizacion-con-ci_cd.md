# Módulo 12 - AUTOMATIZACIÓN CON CI\_CD

## Módulo 12.1: Uso de Terraform en Pipelines de CI/CD (El "Por Qué")

### 1. El Anti-Patrón: El apply Local

Hasta ahora, hemos ejecutado `terraform apply` desde nuestro portátil. En un entorno profesional, este es el anti-patrón número uno.

¿Por qué es tan peligroso el apply local?

* Gobernanza Cero: ¿Quién revisó ese cambio? ¿Quién lo aprobó? No hay rastro de auditoría, excepto un evento genérico en CloudTrail.
* Gestión de Secretos: El desarrollador necesita credenciales de administrador (`AdministratorAccess`) en su máquina local. Esto es un riesgo de seguridad catastrófico.
* Consistencia (Deriva): ¿Y si el desarrollador hizo un cambio manual en la consola (ClickOps) y olvidó importarlo? Su plan local será diferente al de sus compañeros, llevando a una "deriva".
* Bloqueo de Estado: Si dos desarrolladores ejecutan apply al mismo tiempo, corromperán el estado (el problema del Módulo 11).

### 2. La Solución: El Pipeline de CI/CD como Única Vía

La "buena práctica" empresarial es inequívoca: **Prohibir los apply locales**.

La **única** vía autorizada para desplegar cambios en la infraestructura debe ser a través de un pipeline de CI/CD (Integración Continua / Entrega Continua).

Este pipeline se convierte en el "guardián" de su infraestructura.

#### Beneficios del Flujo de Trabajo GitOps

Al forzar todos los cambios a través de un pipeline basado en Git (un flujo **GitOps**), ganamos:

* Revisión de Pares (Peer Review): Todo cambio de infraestructura debe pasar por un Pull Request (PR). Sus compañeros de equipo pueden revisar el código HCL y el `terraform plan` antes de que se apruebe.
* Seguridad "Shift-Left": El pipeline puede ejecutar automáticamente herramientas de análisis de seguridad (como `tfsec` o `checkov`) en cada PR, buscando SGs abiertos o buckets S3 públicos antes de que se desplieguen.
* Auditoría Inmutable: Su historial de Git (`git log`) se convierte en un registro de auditoría perfecto e inmutable de quién, qué y por qué cambió en su infraestructura.
* Consistencia: El pipeline se ejecuta en un entorno limpio (un "runner" de CI/CD), asegurando que cada plan es consistente y fiable.
* Gestión de Secretos Segura: Solo el pipeline (el "rol" del runner) tiene las credenciales de AWS, no los desarrolladores.

### 3. El Blueprint Canónico del Pipeline

El flujo de trabajo estándar de GitOps para Terraform, que implementaremos, sigue este patrón:

{% stepper %}
{% step %}
### En un Pull Request (PR)

* El desarrollador abre un PR.
* El pipeline se dispara **automáticamente**.
* Ejecuta: `init`, `validate`, `fmt --check`, `tfsec` (seguridad) y `terraform plan`.
* El pipeline publica el resultado del plan como un **comentario en el PR**.
* El equipo revisa el plan.
{% endstep %}

{% step %}
### En una Fusión (Merge) a main

* Un revisor aprueba el PR y lo fusiona.
* El pipeline se dispara **automáticamente** en la rama `main` (para el entorno de dev o test).
* Ejecuta `terraform apply` con el plan guardado.
{% endstep %}

{% step %}
### En un Despliegue a Producción

* El despliegue a `prod` **NO** debe ser automático.
* Requiere una **aprobación manual**, como un clic en un "Entorno" de GitHub, una etiqueta de Git (`git tag`) o un `workflow_dispatch`.
* Una vez aprobado, el pipeline ejecuta `terraform apply` en producción.
{% endstep %}
{% endstepper %}

***

### 4. Material de Apoyo

#### Documentos Clave

* (Interno) **Manual Empresarial (Fuente 1):** Las secciones "Gobernanza a través de la Automatización" y "Automatización y Gobernanza" son la base de este módulo.

#### Nota del Formador

* Analogía clave:
  * `apply` Local: Es como si cada cirujano trajera sus propias herramientas (potencialmente sucias) de casa para operar.
  * Pipeline de CI/CD: Es el hospital que exige que todas las operaciones se realicen en un quirófano estéril (el runner), con herramientas esterilizadas (el código base) y la supervisión de un jefe de cirugía (la aprobación del PR).

***

## Módulo 12.2 y 12.3: GitHub Actions (Presentación e Integración)

### 1. ¿Qué es GitHub Actions?

`GitHub Actions` es una plataforma de CI/CD integrada directamente en GitHub. Le permite automatizar flujos de trabajo en respuesta a eventos de GitHub (como `push` o `pull_request`).

* Es uno de los sistemas de CI/CD más populares para Terraform por su estrecha integración con el código fuente.
* Otras opciones populares incluyen GitLab CI, Jenkins y Terraform Cloud.

### 2. Anatomía de un Archivo de Configuración de GitHub Actions

Los flujos de trabajo (Workflows) se definen en archivos YAML que se almacenan en el directorio `.github/workflows/` de su repositorio.

Un flujo de trabajo se compone de:

1. `name`: El nombre del flujo de trabajo.
2. `on (Eventos)`: El disparador (trigger). Ejemplos:
   * `on: pull_request`: Cada vez que se abre o actualiza un PR.
   * `on: push: branches: [ main ]`: Cada vez que se fusiona a `main`.
   * `on: workflow_dispatch`: Un botón de "Ejecutar" manual en la interfaz de GitHub.
3. `jobs` (Trabajos): Uno o más trabajos que se ejecutan (ej. un job para plan y otro para apply).
4. `steps` (Pasos): Los comandos individuales dentro de un trabajo.

### 3. Las "Actions" Clave para Terraform

Dentro de un `step`, podemos ejecutar comandos (`run: terraform plan`) o usar "Actions" reutilizables de la comunidad. Las 3 "Actions" fundamentales para Terraform son:

* `actions/checkout@v4`
  * Clona su repositorio de Git en el runner para que el pipeline pueda acceder a su código HCL.
* `hashicorp/setup-terraform@v3`
  * Instala el binario de Terraform (en la versión que usted especifique) en el runner.
* `aws-actions/configure-aws-credentials@v4`
  * Autentica el runner con AWS.

### 4. Buena Práctica: Autenticación con OIDC (Sin Claves)

Anti-Patrón: Crear un usuario IAM, generar claves de acceso (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`) y guardarlas en GitHub Secrets. Si el repositorio es comprometido, sus claves se filtran.

Buena Práctica (OIDC):

Usamos OIDC (OpenID Connect) para crear una relación de confianza sin claves.

Flujo general:

1. En AWS:
   * Crear un "Proveedor de Identidad" OIDC en IAM que confía en `token.actions.githubusercontent.com`.
   * Crear un Rol de IAM (ej. `rol-github-actions`) que confía en ese proveedor (condicionado al repositorio).
   * Asignar a ese rol los permisos necesarios para Terraform.
2. En GitHub:
   * No guardamos secretos de credenciales estáticas.
   * Damos al workflow `id-token: write`.
   * Usamos la Action `aws-actions/configure-aws-credentials` y le indicamos que asuma ese rol.

Flujo:

* GitHub pide a AWS: "Soy repo:mi-repo, dame credenciales".
* AWS responde con credenciales temporales para el rol, que caducan en corto tiempo.

### 5. Lab: Autenticación OIDC (El Lado de Terraform)

Objetivo: Crear el Rol de IAM que GitHub Actions asumirá.

Código de Terraform ejemplo (bootstrap-iam):

```hcl
# main.tf (en un proyecto de 'bootstrap-iam')

# 1. El Proveedor OIDC de GitHub (se crea una vez por cuenta)
resource "aws_iam_openid_connect_provider" "github" {
  url             = "https://token.actions.githubusercontent.com"
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = ["1b511abead59c6ce207077c0bf0e0043b1382612"] # (Este es el thumbprint actual)
}

# 2. La Política de Confianza (El "Portero")
data "aws_iam_policy_document" "github_trust" {
  statement {
    effect = "Allow"
    actions = ["sts:AssumeRoleWithWebIdentity"]
    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.github.arn]
    }
    # ¡CONDICIÓN DE SEGURIDAD!
    # Solo permite asumir el rol si viene de nuestro repositorio específico
    condition {
      test     = "StringLike"
      variable = "token.actions.githubusercontent.com:sub"
      values   = ["repo:mi-organizacion/mi-repo-terraform:*"]
    }
  }
}

# 3. El Rol de IAM que usará el pipeline
resource "aws_iam_role" "github_actions_role" {
  name               = "rol-github-actions-terraform"
  assume_role_policy = data.aws_iam_policy_document.github_trust.json
}

# 4. Los Permisos (La "Pulsera VIP")
# (Aquí adjuntaría su política de administrador o una política de privilegios mínimos)
resource "aws_iam_role_policy_attachment" "github_permisos" {
  role       = aws_iam_role.github_actions_role.name
  policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess" # (Para el lab)
}

# 5. Salida
output "github_role_arn" {
  value = aws_iam_role.github_actions_role.arn
}
```

* Acción: Deberá ejecutar `terraform apply` a este código (o hacerlo manualmente) y guardar el ARN de salida.

***

## Módulo 12.4: Validación de Código (terraform validate)

### 1. Introducción

Antes de ejecutar un `plan` (que consume tiempo y hace llamadas API), queremos una comprobación rápida de "cordura" (sanity check).

### 2. terraform validate

* ¿Qué hace? `terraform validate` comprueba que el código HCL es sintácticamente válido.
* ¿Qué comprueba?
  * Errores de sintaxis HCL (ej. llaves `{` faltantes).
  * Referencias a variables correctas.
  * Tipos de datos correctos.
* ¿Qué NO comprueba?
  * No contacta a AWS.
  * No comprueba si sus valores son lógicos (ej. `ami = "ami-inexistente"` pasará `validate`, pero fallará en `plan`).
  * No comprueba la autenticación.
* Uso: Es el primer paso en cualquier pipeline de CI/CD, justo después de `init`. Si falla, el pipeline debe detenerse inmediatamente.

### 3. terraform fmt --check

* `terraform fmt`: Reformatea el código en su disco.
* `terraform fmt --check`: No reformatea; devuelve un código de error si el código necesita ser formateado.
* Uso: Comprobación de "estilo de código" (linting) en el pipeline. Fuerza a todos los desarrolladores a formatear su código antes de hacer commit.

### 4. Ejemplo de Job de "Validación" en GitHub Actions

Este job debe ejecutarse en cada PR.

YAML de ejemplo:

```yaml
# .github/workflows/terraform.yml
jobs:
  validate:
    name: 'Validar Código HCL'
    runs-on: ubuntu-latest
    steps:
      - name: 'Checkout'
        uses: actions/checkout@v4

      - name: 'Setup Terraform'
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.9.0

      - name: 'Terraform Init' # 'backend=false' porque no necesitamos el estado solo para validar la sintaxis
        run: terraform init -backend=false

      - name: 'Terraform Format Check' # Falla el pipeline si el código no está formateado
        run: terraform fmt --check --recursive

      - name: 'Terraform Validate' # Falla el pipeline si la sintaxis es inválida
        run: terraform validate
```

Nota:

* `validate` es rápido y barato. `plan` es más lento y hace llamadas API. Ejecute `validate` primero para "fail fast".

***

## Módulo 12.5: Automatización de Planes (terraform plan en PR)

### 1. Introducción

Este es el núcleo del flujo de trabajo de GitOps y la característica más importante de un pipeline de CI/CD para Terraform.

Objetivo: Cuando un desarrollador abre un Pull Request, debe ver el `terraform plan` exacto que resultará de su cambio, directamente en la interfaz del PR.

### 2. Lab: El Job plan de GitHub Actions

Objetivo: Crear el job de GitHub Actions que se dispara en un `pull_request` y ejecuta `plan`.

Requisitos:

* Haber configurado la autenticación OIDC (Lab 12.3).
* Tener el ARN del Rol de IAM (ej. `arn:aws:iam::...:role/rol-github-actions-terraform`).
* Guardar ese ARN como un GitHub Secret llamado `AWS_ROLE_TO_ASSUME`.
* Tener un backend S3/DynamoDB (Módulo 11) configurado en `backend.tf`.

Workflow YAML de ejemplo:

```yaml
name: 'Terraform CI/CD'
on:
  pull_request:
    branches:
      - main
    paths:
      - '**/*.tf'

permissions:
  id-token: write      # Necesario para que OIDC obtenga el token
  contents: read       # Necesario para 'checkout'
  pull-requests: write # Necesario para publicar el 'plan' en el PR

jobs:
  plan:
    name: 'Terraform Plan'
    runs-on: ubuntu-latest
    environment: plan
    steps:
      - name: 'Checkout'
        uses: actions/checkout@v4

      - name: 'Setup Terraform'
        uses: hashicorp/setup-terraform@v3

      - name: 'Configure AWS Credentials (OIDC)'
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
          aws-region: us-east-1

      - name: 'Terraform Init'
        run: terraform init

      - name: 'Terraform Validate'
        run: terraform validate

      - name: 'Terraform Plan'
        run: terraform plan -out=plan.tfplan
```

* Resultado: Cuando se abre un PR, este job se ejecutará. El desarrollador puede ver si su plan tuvo éxito o falló.

### 3. Comentar el Plan en el PR (La Parte Visual)

Para publicar el plan como un comentario en el PR, `run: terraform plan` no es suficiente. El método más simple es usar una Action pre-construida (como `terraform-github-actions` de HashiCorp) o una combinación de `actions/upload-artifact` y scripts que conviertan el plan en texto legible y lo publiquen como comentario.

Punto clave:

* El plan en el PR es la "Revisión de Pares".
* Guardar el plan con `-out=plan.tfplan` es vital: ese artefacto es lo que se usará en el `apply` para garantizar que lo que se aplica es exactamente lo que se revisó.

***

## Módulo 12.6 y 12.7: apply (Prueba Automática vs. Prod Aprobada)

### 1. Introducción

Ya tenemos nuestro `plan.tfplan` (artefacto) generado y revisado en el PR. Ahora, ¿cómo lo aplicamos?

Aquí es donde separamos los entornos.

{% stepper %}
{% step %}
### Escenario 1: apply Automático a test (Módulo 12.6)

Queremos que cualquier cambio fusionado a la rama `main` se despliegue automáticamente en nuestro entorno de test o dev.

Fragmento YAML añadido:

```yaml
on:
  pull_request: # ... (nuestro trigger 'plan') ...
  push:
    branches:
      - main
    paths:
      - '**/*.tf'

jobs:
  plan:
    # ... (job 'plan') ...
    - name: 'Upload Plan Artifact'
      uses: actions/upload-artifact@v4
      with:
        name: plan
        path: plan.tfplan

  apply:
    name: 'Terraform Apply (a TEST)'
    runs-on: ubuntu-latest
    needs: [plan]
    if: github.event_name == 'push'
    steps:
      - name: 'Checkout'
        uses: actions/checkout@v4
      - name: 'Setup Terraform'
        uses: hashicorp/setup-terraform@v3
      - name: 'Configure AWS Credentials (OIDC)'
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
          aws-region: us-east-1
      - name: 'Download Plan Artifact'
        uses: actions/download-artifact@v4
        with:
          name: plan
      - name: 'Terraform Init'
        run: terraform init
      - name: 'Terraform Apply'
        run: terraform apply "plan.tfplan"
```

Resultado: Un merge a `main` ahora aplica automáticamente los cambios al entorno de test.
{% endstep %}

{% step %}
### Escenario 2: apply Aprobado a prod (Módulo 12.7)

El despliegue a producción requiere aprobación humana. Usamos la característica "Environments" de GitHub y un trigger `workflow_dispatch` (manual).

Pasos:

1. En GitHub:
   * Repo -> Settings -> Environments -> crear `produccion`.
   * Activar "Required reviewers" y añadir al equipo de Ops.
2. En YAML: añadir `workflow_dispatch` y un job `apply-prod`.

Fragmento YAML:

```yaml
on:
  pull_request: ...
  push:
    branches: [ main ]
  workflow_dispatch:
    inputs:
      confirm_deploy:
        description: 'Escriba "prod" para confirmar despliegue'
        required: true

jobs:
  # plan y apply-test como antes...

  apply-prod:
    name: 'Terraform Apply (a PRODUCCION)'
    runs-on: ubuntu-latest
    if: github.event_name == 'workflow_dispatch'
    environment:
      name: produccion
    steps:
      # Checkout, Setup, OIDC...
      - name: 'Terraform Init (Prod)'
        run: terraform init
      - name: 'Terraform Plan (Prod)'
        run: terraform plan -out=plan.tfplan
      - name: 'Terraform Apply (Prod)'
        run: terraform apply "plan.tfplan"
```

Notas:

* El `if` y el `environment` son los guardianes para producción.
* Separación de estado: use workspaces o backends separados (o directorios separados) para aislar `test` y `prod`.
{% endstep %}
{% endstepper %}

***

## Módulo 12.8: Estrategias de Rollback en Terraform

### 1. El Problema: "¡Deshacer!"

Escenario: Fusiona un PR, el `apply` a producción es exitoso. Minutos después, la aplicación falla por el cambio. ¿Cómo lo deshacemos?

### 2. El Anti-Patrón: terraform undo

No existe `terraform undo` o `terraform rollback`. Terraform es declarativo y solo sabe cómo ir "hacia adelante".

### 3. La Estrategia Correcta: Un Flujo de Git

Su estrategia de rollback es, fundamentalmente, una estrategia de Git.

{% stepper %}
{% step %}
### Opción 1: "Roll Forward" (Arreglar Hacia Adelante) — La Mejor

* No retroceda. Arregle el problema con un nuevo commit (hotfix).
* Flujo: identificar bug → crear PR con la corrección → revisar → fusionar → pipeline aplica la corrección.
* Pros: rápido, mantiene historial limpio.
{% endstep %}

{% step %}
### Opción 2: "Roll Back" (Revertir) — Emergencia

* Use `git revert` para deshacer el commit problemático.

Flujo:

1. Identifica el commit problemático (Commit A).
2. Ejecuta: `git revert <ID_del_Commit_A>` (esto crea un nuevo commit B que deshace A).
3. Push de Commit B.
4. El pipeline se ejecuta: el HCL ahora describe el estado anterior; `terraform plan` mostrará los cambios para revertir la infraestructura; `terraform apply` ejecuta el revert.

* Pros: forma limpia y auditable de deshacer un cambio.
{% endstep %}
{% endstepper %}

Nota:

* Repita: No existe `terraform undo`. El rollback es un `git revert` seguido de un `terraform apply`.
* Alternativa: Blue/Green deployment; el rollback es cambiar el puntero del ALB de vuelta al stack sano.

***

## Módulo 12.9: Lab - Uso de Herramientas de Seguridad en Pipelines ("Shift-Left")

### 1. Introducción

"Shift-Left" es mover comprobaciones de seguridad lo más a la izquierda posible: analizar en el Pull Request en lugar de en producción.

### 2. Herramientas de Análisis Estático

Herramientas que leen HCL y buscan patrones inseguros:

* `tfsec`
* `checkov`

Ambas son CLI y se integran bien en GitHub Actions.

### 3. Objetivo del Laboratorio

Añadir un job de `tfsec` al pipeline de PR.

#### Paso 1: Crear un Archivo HCL "Inseguro" (para la prueba)

Añadir temporalmente a `main.tf`:

```hcl
# main.tf (AÑADIR ESTO TEMPORALMENTE)
resource "aws_s3_bucket" "bucket_inseguro" {
  bucket = "mi-bucket-inseguro-para-tfsec-123"
  # ¡INSEGURO!
  acl = "public-read"
}
```

#### Paso 2: Añadir el Job `security` al Pipeline YAML

Fragmento YAML:

```yaml
jobs:
  validate:
    # ... (job 'validate') ...

  security:
    name: 'Análisis de Seguridad (tfsec)'
    runs-on: ubuntu-latest
    needs: [validate]
    steps:
      - name: 'Checkout'
        uses: actions/checkout@v4

      - name: 'Ejecutar tfsec'
        uses: aquasecurity/tfsec-action@v1.0.0
        with:
          soft_fail: false

  plan:
    name: 'Terraform Plan'
    runs-on: ubuntu-latest
    needs: [validate, security]
    # ... (resto del job 'plan') ...
```

#### Paso 3: Ejecutar y Verificar

1. Commit y push de la rama.
2. Abra un PR.
3. Verifique en Actions:
   * `validate` pasa.
   * `security` (`tfsec`) falla, mostrando errores (ej. S3 público).
   * `plan` nunca se ejecuta mientras `security` falla.
4. Arreglar el HCL inseguro, push, y el pipeline volverá a ejecutarse correctamente.

Nota:

* Esto evita que configuraciones inseguras lleguen a producción: Shift-Left en acción.
* `checkov` y `tfsec` son intercambiables en este contexto.

***

## Módulo 12.10: Buenas Prácticas en IaC con CI/CD (Resumen)

1. ✅ Prohibir el `apply` Local: El pipeline es la única vía a producción.
2. 🔒 Usar OIDC (No Claves Estáticas): Autentique sus pipelines usando Roles de IAM OIDC.
3. 🔬 Planificar en el PR: Cada PR debe generar un `terraform plan` y publicarlo como comentario para su revisión.
4. 📦 Usar Artefactos de Plan: Guarde el plan (`plan -out=plan.tfplan`). El job `apply` debe aplicar ese artefacto exacto (`apply "plan.tfplan"`).
5. 🛡️ "Shift-Left": Ejecute `terraform fmt --check`, `terraform validate` y un escáner de seguridad (`tfsec` o `checkov`) en cada PR.
6. 🚦 Proteger la Producción: Los despliegues a producción deben requerir una aprobación humana (GitHub Environments o `workflow_dispatch`).
7. 🔄 El Rollback es `git revert`: No hay `terraform undo`. Revertir en Git y aplicar.
8. 🏗️ Usar un Backend Remoto: El pipeline debe usar un backend S3/DynamoDB para gestionar el estado y el bloqueo.

***
