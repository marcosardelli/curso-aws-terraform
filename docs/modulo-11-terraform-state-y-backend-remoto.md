# Módulo 11 - TERRAFORM STATE Y BACKEND REMOTO

## Módulo 11.1: La Importancia del Estado en Terraform (.tfstate)

### 1. ¿Qué es el Estado? (Recordatorio)

Como vimos en el Módulo 3, el archivo de estado (por defecto, `terraform.tfstate`) es el cerebro o el mapa de Terraform. Es un archivo JSON que Terraform usa para:

* Mapear Recursos: Conecta los nombres locales de su código (ej. `resource "aws_instance" "web"`) con los IDs del mundo real (ej. `i-12345abcdef`).
* Almacenar Metadatos: Guarda una copia de todos los atributos y sus valores de los recursos que gestiona.
* Determinar Dependencias: Registra el "grafo" de dependencias (ej. esta instancia depende de esta subred).

### 2. ¿Por qué es tan Crítico?

El estado es la "fuente de verdad" de Terraform. Cuando usted ejecuta `terraform plan`, Terraform realiza una comparación de 3 vías:

1. Código HCL: el estado deseado (lo que usted escribió).
2. Archivo `.tfstate`: el estado conocido (lo que Terraform cree que existe).
3. API de AWS: el estado real (lo que existe de verdad en la nube).

Sin el archivo de estado, Terraform es "ciego". Si borra su archivo `.tfstate` local:

* Terraform pierde el mapa.
* No tiene idea de qué recursos gestiona.
* Si ejecuta `terraform apply` otra vez, intentará crear todos los recursos de nuevo (y fallará con conflictos de nombres).
* Si ejecuta `terraform destroy`, Terraform dirá "No hay nada que destruir" (porque su estado está vacío).

El estado es el activo más sensible de su proyecto.

### 3. Material de Apoyo

#### Documentos Clave

* [Propósito del Estado (Terraform Docs)](https://developer.hashicorp.com/terraform/language/state/purpose)

{% hint style="info" %}
Nota del Formador:

* Analogía:
  * HCL: El plano del arquitecto.
  * AWS: El edificio real.
  * `.tfstate`: El inventario detallado del capataz.
{% endhint %}

***

## Módulo 11.2: Las Limitaciones del Estado Local

### Introducción

Por defecto Terraform opera en modo "local", creando `terraform.tfstate` en el directorio donde ejecuta `apply`. Esto funciona para un desarrollador en pruebas, pero falla en escenarios reales.

{% stepper %}
{% step %}
### Colaboración (El Anti-Patrón del Equipo)

Escenario:

* Usted y Ana clonan el mismo repositorio.
* Día 1 (Usted): `terraform apply` → crea `terraform.tfstate` (Estado A) con `aws_instance.web`.
* Día 2 (Ana): sin estado local, `terraform apply` → su propio `terraform.tfstate` (Estado B) con otra `aws_instance.web`.

Desastre:

* Dos servidores web, dos archivos de estado diferentes y contradictorios.
* Si usted añade `aws_s3_bucket` (Estado A+) y Ana añade `aws_db_instance` (Estado B+), los estados divergen y la infraestructura queda en caos.

"No solución" peligrosa: subir `.tfstate` a Git → NUNCA.
{% endstep %}

{% step %}
### Seguridad (Secretos en Texto Plano)

* Valores sensibles (p. ej. contraseñas) se escriben en texto plano en `terraform.tfstate`.
* Ejemplo JSON (resumido):

```json
{
  "resources": [
    {
      "type": "aws_db_instance",
      "name": "db",
      "instances": [
        {
          "attributes": {
            "password": "MiPasswordSecreto123",
            "sensitive_attributes": ["password"]
          }
        }
      ]
    }
  ]
}
```

Si hace commit de este archivo en GitHub, ha filtrado secretos a su organización o al mundo.
{% endstep %}

{% step %}
### Fragilidad (El Portátil Perdido)

* El archivo local es un SPOF.
* Disco duro falla, carpeta borrada o portátil robado → pierde el estado.
* Consecuencia: no puede gestionar (actualizar o destruir) la infraestructura sin reimportar o borrar manualmente.
{% endstep %}
{% endstepper %}

### La Solución: Backends Remotos

Use un Backend Remoto: no guarde el estado localmente; guárdelo en una ubicación centralizada, segura y duradera.

Para AWS, el patrón estándar es:

* Amazon S3: almacenar el archivo de estado.
* Amazon DynamoDB: bloquear el archivo de estado (control de concurrencia).

{% hint style="info" %}
Nota del Formador:

* Regla de Oro: la primera acción en cualquier proyecto de Terraform (tras `init`) es configurar un `.gitignore` y un Backend Remoto.
{% endhint %}

***

## Módulo 11.3: Lab - Configuración de Backend Remoto en S3

### 1. Introducción

Configurar S3 para almacenar `terraform.tfstate` de forma centralizada.

Beneficios: centralización, durabilidad, cifrado, control de acceso y versionado.

### 2. El Problema del "Huevo y la Gallina" (Bootstrap)

Necesitamos el bucket S3 para el estado, pero para crear el bucket normalmente usaríamos Terraform. La solución: crear un proyecto de bootstrap separado que use estado local para crear el bucket (y luego descartarlo).

### 3. Objetivo del Laboratorio (Parte 1: Crear el Backend)

Crear los recursos AWS necesarios para alojar el estado.

{% stepper %}
{% step %}
### Paso: Crear un Nuevo Proyecto (terraform-backend-setup)

Estructura sugerida:

```
terraform-backend-setup/
├── main.tf
├── providers.tf
└── terraform.tfstate   # este proyecto SÍ usará estado local
```
{% endstep %}

{% step %}
### Paso: Escribir el HCL (main.tf)

Ejemplo (respetar el formato y nombres únicos de bucket):

providers.tf

```hcl
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

main.tf (ejemplos)

```hcl
resource "aws_s3_bucket" "terraform_state" {
  bucket = "mi-empresa-terraform-state-remoto-12345"
  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_s3_bucket_versioning" "state_versioning" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "state_encryption" {
  bucket = aws_s3_bucket.terraform_state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "state_pab" {
  bucket                      = aws_s3_bucket.terraform_state.id
  block_public_acls           = true
  block_public_policy         = true
  ignore_public_acls          = true
  restrict_public_buckets     = true
}
```

(En el próximo lab añadiremos DynamoDB.)
{% endstep %}

{% step %}
### Paso: Ejecutar y Verificar

1. `terraform init`
2. `terraform apply` (escriba `yes`)
3. Verifique en la consola de S3 que el bucket existe y que Cifrado, Versionado y Bloqueo de Acceso Público están activados.
{% endstep %}
{% endstepper %}

### 4. Objetivo del Laboratorio (Parte 2: Configurar el Backend)

Ahora vuelva a su proyecto principal (ej. `terraform-vpc`) y configure el backend para apuntar al bucket creado.

{% stepper %}
{% step %}
### Paso: Crear `backend.tf` en el proyecto (terraform-vpc)

Ejemplo `backend.tf`:

```hcl
terraform {
  backend "s3" {
    bucket = "mi-empresa-terraform-state-remoto-12345"
    key    = "networking/prod/terraform.tfstate"
    region = "us-east-1"
    # En el próximo lab añadiremos la tabla DynamoDB aquí
  }
}
```
{% endstep %}

{% step %}
### Paso: Ejecutar `terraform init` (Migración)

1. En su proyecto `terraform-vpc`, ejecute `terraform init`.
2. Terraform detectará el backend S3 y que existe un `terraform.tfstate` local.
3. Le preguntará si quiere copiar el estado al nuevo backend. Responda `yes`.
4. Terraform copiará el estado local a `s3://mi-empresa.../networking/prod/terraform.tfstate` y limpiará el estado local (quedará una referencia al backend).
{% endstep %}

{% step %}
### Verificación

* Estado ahora remoto en S3.
* Nota clave: `key` organiza los estados por proyecto/entorno (ej. `networking/prod/...`).
{% endstep %}
{% endstepper %}

{% hint style="info" %}
Nota del Formador:

* El proceso de bootstrap es de un solo uso. `backend.tf` es algo que todos los proyectos de aplicación tendrán.
* Use `key` por componente/entorno para aislar estados (ej. `networking/prod`, `app/dev`).
{% endhint %}

***

## Módulo 11.4: Lab - Bloqueo de Estado con DynamoDB

### 1. El Problema: Condición de Carrera (Race Condition)

Si dos personas ejecutan `apply` al mismo tiempo contra el mismo estado en S3, uno puede sobrescribir los cambios del otro. Necesitamos bloqueo.

### 2. La Solución: Bloqueo de Estado (State Locking)

Usar una tabla de DynamoDB como semáforo. Terraform intentará crear un ítem con un LockID; si otro proceso ya lo creó, el segundo fallará hasta que el bloqueo se libere.

### 3. Objetivo del Laboratorio

1. Añadir `aws_dynamodb_table` al proyecto `terraform-backend-setup`.
2. Actualizar `backend.tf` del proyecto de aplicación para usar esa tabla.

{% stepper %}
{% step %}
### Paso: Añadir la Tabla DynamoDB (en terraform-backend-setup/main.tf)

Ejemplo:

```hcl
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "mi-empresa-terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  tags = {
    Name = "Tabla de Bloqueo Terraform"
  }
}
```

Ejecute `terraform apply` en el proyecto `terraform-backend-setup` para crear la tabla.
{% endstep %}

{% step %}
### Paso: Actualizar el Backend del Proyecto (terraform-vpc/backend.tf)

Modificar `backend.tf` para incluir `dynamodb_table` y `encrypt`:

```hcl
terraform {
  backend "s3" {
    bucket         = "mi-empresa-terraform-state-remoto-12345"
    key            = "networking/prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "mi-empresa-terraform-locks"
    encrypt        = true
  }
}
```
{% endstep %}

{% step %}
### Paso: Ejecutar y Verificar (Prueba de Concurrencia)

1. `terraform init` (responda `yes` si pide reiniciar el backend).
2. Abra dos terminales en el mismo directorio (`terraform-vpc`):
   * Terminal 1: `terraform apply` (dejela esperando el `yes`).
   * Terminal 2: mientras Terminal 1 está en espera, ejecute `terraform apply`.
3. Resultado esperado:
   * Terminal 2 fallará con `Error acquiring the state lock` (ConditionalCheckFailedException).
4. Verifique en la consola AWS → DynamoDB → `mi-empresa-terraform-locks` → "Explorar elementos": verá el registro de bloqueo.
5. En Terminal 1, cancele (escriba `no`), el bloqueo se liberará.
{% endstep %}
{% endstepper %}

{% hint style="info" %}
Punto Clave: S3 + DynamoDB es el patrón estándar (S3 para almacenamiento/versionado, DynamoDB para bloqueo).
{% endhint %}

***

## Módulo 11.5: Encriptación del Estado Remoto

Este tema formaliza el concepto de cifrado del estado remoto.

### 1. El Riesgo

Si alguien obtiene acceso de lectura al bucket S3 que contiene el estado, obtendrá todos los secretos (en texto plano) del estado.

### 2. La Solución: Cifrado en Reposo (At-Rest Encryption)

Debe asegurarse de que `terraform.tfstate` en S3 esté cifrado.

#### Opción 1: Cifrado Gestionado por S3 (AES-256)

* Ejemplo ya usado en Lab 11.3 (server-side AES256).
* Pros: Simple, gratuito.
* Contras: AWS controla la clave.

En `backend.tf`:

```hcl
encrypt = true
```

#### Opción 2: Cifrado Gestionado por KMS (aws:kms)

* Crear una clave KMS y configurar S3 para usarla:

```hcl
resource "aws_kms_key" "state_kms_key" {
  description = "Clave para cifrar el estado de Terraform"
}

resource "aws_s3_bucket_server_side_encryption_configuration" "state_encryption" {
  bucket = aws_s3_bucket.terraform_state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.state_kms_key.arn
    }
  }
}
```

* Pros: Control total vía IAM y KMS.
* Contras: Coste pequeño por la clave KMS y necesidad de permisos `kms:Decrypt` para CI/CD.

{% hint style="info" %}
Punto Clave: El cifrado no es opcional. AES-256 es el mínimo; KMS es la buena práctica empresarial.
{% endhint %}

***

## Módulo 11.6: Comandos Avanzados de Manejo de Estado

### 1. Introducción

El 99% de las veces use solo `init`, `plan`, `apply`, `destroy`. Para el 1% de emergencia, Terraform ofrece comandos para manipular el estado directamente. Haga siempre backup del estado antes de usar estos comandos.

### 2. Los Comandos: `terraform state ...`

{% stepper %}
{% step %}
### terraform state list

* ¿Qué hace? Lista todos los recursos (por su nombre local) que Terraform gestiona actualmente en el estado.
* Uso: ver rápidamente qué hay en el estado.
* Ejemplo:

```bash
$ terraform state list
data.aws_ami.amazon_linux_2
aws_instance.servidor_web_publico_1
aws_key_pair.mi_clave
aws_security_group.web
```
{% endstep %}

{% step %}
### terraform state show

* ¿Qué hace? Muestra todos los atributos de un recurso tal como están en el estado (JSON).
* Uso: depuración (ej. "¿por qué Terraform piensa que el AMI es ami-123?").
* Ejemplo:

```bash
$ terraform state show aws_instance.servidor_web_publico_1
# muestra el recurso con decenas de atributos
```
{% endstep %}

{% step %}
### terraform state rm (Peligroso)

* ¿Qué hace? "Olvida" un recurso: lo elimina del archivo `.tfstate`.
* Importante: NO destruye el recurso en AWS.
* Uso: cuando un recurso fue eliminado manualmente o está roto y quiere que Terraform deje de gestionarlo.
* Ejemplo:

```bash
$ terraform state rm aws_instance.servidor_web_publico_1
```
{% endstep %}

{% step %}
### terraform state mv (Muy útil para refactorizar)

* ¿Qué hace? Mueve/renombra un recurso dentro del estado.
* Escenario: renombró su recurso en HCL y quiere que el estado refleje el nuevo nombre sin destruir/crear.
* Ejemplo:

```bash
$ terraform state mv aws_instance.web aws_instance.servidor_web
$ terraform plan  # No changes.
```
{% endstep %}

{% step %}
### terraform import (Para infraestructura existente - "Brownfield")

* ¿Qué hace? Adopta un recurso existente en AWS y lo importa al estado bajo un nombre local.
* Proceso:
  1. Escriba el HCL para el recurso.
  2. Ejecute `terraform import <nombre_local> <id_aws>`.
* Ejemplo:

```bash
$ terraform import aws_s3_bucket.importado mi-bucket-manual
```

Después de esto, el recurso existe en el estado bajo `aws_s3_bucket.importado`.
{% endstep %}
{% endstepper %}

### 3. Resolución de Conflictos (Bloqueos Atascados)

* Problema: un apply falla y el bloqueo en DynamoDB queda "stuck".
* Síntoma: `terraform plan` falla con "Error acquiring the state lock".
*   Solución:

    1. Verifique que nadie más esté ejecutando `apply`.
    2. Revise el mensaje de error para ver el `LockID`.
    3. Ejecute:

    ```bash
    $ terraform force-unlock <LockID>
    ```

    4. Alternativa: borrar manualmente el ítem en la tabla DynamoDB.

***

## Módulo 11.7: Buenas Prácticas en la Gestión del Estado

1. Usar un Backend Remoto
   * Nunca use estado local para proyectos reales. Use `backend "s3"` con `dynamodb_table`.
2. Proteger el Backend
   * El bucket de S3 del estado es el recurso más sensible.
   * Bloquear acceso público, cifrar el estado (AES-256 mínimo, KMS recomendado), habilitar versionado y aplicar políticas IAM estrictas (solo CI/CD y admins deberían tener s3:GetObject/s3:PutObject).
3. No tocar el estado manualmente
   * No edite `terraform.tfstate` a mano. Use `terraform state ...` si es necesario.
4. Aislar los estados
   * No usar un único archivo de estado monolítico para toda la empresa.
   * Divida estados por componente/entorno (ej. `networking/prod/terraform.tfstate`, `database/prod/terraform.tfstate`, `application/prod/terraform.tfstate`).
   * Use `terraform.workspace` para gestionar entornos en el mismo directorio cuando corresponda.
5. Gestionar secretos fuera del estado
   * Mejor evitar secretos en el estado.
   * Use AWS Secrets Manager o HashiCorp Vault.
   * Si debe haber secretos en el estado, cifre con KMS y marque variables como `sensitive = true`.

{% hint style="success" %}
Resumen: S3 (almacenamiento + versionado) + DynamoDB (bloqueo) + cifrado (KMS preferible) + aislamiento por proyecto/entorno = patrón recomendado para producción.
{% endhint %}
