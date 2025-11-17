# Módulo 4 - Gestión de identidad y seguridad

## Módulo 4 - Gestión de identidad y seguridad

## Módulo 4.1: Introducción a IAM como Código

### 1. ¿Por qué Automatizar IAM?

En el Módulo 2, establecimos que **IAM (Identity and Access Management)** es el corazón de la seguridad en AWS. Es la base que define "quién puede hacer qué".

Es tentador gestionar esto manualmente (ClickOps) porque a menudo se configura "solo una vez". Esto es un error crítico. La gestión de permisos es un proceso continuo, no un evento.

**Automatizar IAM con Terraform (IAM as Code) nos da:**

* **Auditabilidad:** Cada cambio de permiso (quién lo solicitó, quién lo aprobó) queda registrado en el historial de **Git**.
* **Repetibilidad:** Se puede desplegar la misma estructura de permisos en la cuenta de dev, test y prod con total confianza.
* **Gobernanza:** Se puede _revisar_ un Pull Request que otorga a un usuario acceso a s3:DeleteObject _antes_ de que se aplique.
* **Principio de Mínimo Privilegio:** Es mucho más fácil escribir y mantener políticas granulares y estrictas como código que hacer clic en la consola.

### 2. Los Recursos de IAM en Terraform

Para modelar IAM, el provider de AWS de Terraform nos da un conjunto de recursos que mapean 1 a 1 con los conceptos de IAM.

#### Recursos Principales (El "Quién"):

* **aws\_iam\_user:** Define un usuario (una persona o servicio).
* **aws\_iam\_group:** Define un grupo de usuarios.
* **aws\_iam\_role:** Define un rol (la identidad temporal, preferida para servicios).

#### Recursos de Política (El "Qué"):

* **aws\_iam\_policy:** Define una **política gestionada** (una política independiente que se puede "adjuntar" a múltiples entidades).
* **aws\_iam\_user\_policy:** Define una **política en línea (inline)** (una política vinculada directamente a 1 usuario).
* **aws\_iam\_group\_policy:** Define una política en línea para 1 grupo.
* **aws\_iam\_role\_policy:** Define una política en línea para 1 rol.

#### Recursos de Conexión (Las "Flechas"):

* **aws\_iam\_user\_policy\_attachment:** Conecta un aws\_iam\_user a una aws\_iam\_policy.
* **aws\_iam\_group\_policy\_attachment:** Conecta un aws\_iam\_group a una aws\_iam\_policy.
* **aws\_iam\_role\_policy\_attachment:** Conecta un aws\_iam\_role a una aws\_iam\_policy.
* **aws\_iam\_group\_membership:** Añade usuarios a un grupo.

### 3. El Desafío: El "Huevo y la Gallina" de los Permisos

Hay una paradoja inicial: para que Terraform _cree_ recursos de IAM, la entidad que ejecuta Terraform (nuestro usuario terraform-student) ya debe tener permisos para _gestionar_ IAM (es decir, iam:\*).

* En nuestro curso, esto está resuelto porque usamos AdministratorAccess.
* En un entorno de producción, un pipeline de CI/CD asumiría un Rol de IAM específico que tiene permisos para gestionar _otros_ roles y permisos (ej. iam:CreateRole, iam:AttachRolePolicy).

### 4. Material de Apoyo y Siguientes Pasos

* [**Documentación del Provider de AWS - IAM**](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_user)
  * Explore la barra lateral izquierda en esta página. Verá la docena de recursos aws\_iam\_... que usaremos.

## Módulo 4.2: Lab - Creación de Usuarios, Grupos y Políticas

### 1. Objetivo del Laboratorio

Crear una estructura de permisos básica y reutilizable siguiendo las buenas prácticas de IAM. En lugar de adjuntar permisos directamente a los usuarios, usaremos **Grupos**.

Flujo de Trabajo: Usuario -> Grupo -> Política

* Objetivo:
  1. Crear un aws\_iam\_user (un desarrollador).
  2. Crear un aws\_iam\_group (el equipo de "desarrolladores").
  3. Crear una aws\_iam\_policy (una política gestionada que da acceso de solo lectura a S3).
  4. **Adjuntar** la Política al Grupo.
  5. **Añadir** el Usuario al Grupo.

### 2. Estructura de Archivos

Cree una nueva carpeta de proyecto y añada los archivos estándar (providers.tf, main.tf, variables.tf, outputs.tf).

* providers.tf: Configure el provider de AWS (como en el Módulo 3.6).

### 3. Paso 1: Definir las Variables (variables.tf)

Queremos que los nombres de nuestros usuarios y grupos sean configurables.

{% code title="variables.tf" %}
```hcl
variable "nombre_de_usuario" {
  description = "Nombre del nuevo desarrollador"
  type        = string
  default     = "dev-juan-perez"
}

variable "nombre_del_grupo" {
  description = "Nombre del grupo de IAM para desarrolladores"
  type        = string
  default     = "grupo-desarrolladores"
}
```
{% endcode %}

### 4. Paso 2: Crear el Usuario y el Grupo (main.tf)

{% code title="main.tf" %}
```hcl
# 1. Crear el Usuario
resource "aws_iam_user" "usuario_dev" {
  name = var.nombre_de_usuario
  tags = { Entorno = "dev" }
}

# 2. Crear el Grupo
resource "aws_iam_group" "grupo_dev" {
  name = var.nombre_del_grupo
}

# 3. Añadir el Usuario al Grupo
resource "aws_iam_group_membership" "pertenencia_grupo" {
  name  = "pertenencia-${var.nombre_del_grupo}"
  users = [ aws_iam_user.usuario_dev.name ]
  group = aws_iam_group.grupo_dev.name
  # Terraform automáticamente sabe que debe crear
  # el usuario y el grupo ANTES de crear la pertenencia.
}
```
{% endcode %}

### 5. Paso 3: Crear y Adjuntar la Política (main.tf - continuación)

Ahora, creemos la política de "solo lectura de S3" y adjuntémosla al Grupo.

{% code title="main.tf (política y adjunto)" %}
```hcl
# 4. Definir una Política Gestionada
resource "aws_iam_policy" "politica_s3_readonly" {
  name        = "politica-s3-solo-lectura"
  description = "Permite Listar y Leer todos los buckets S3"

  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect   = "Allow",
        Action   = [ "s3:Get*", "s3:List*" ],
        Resource = "*" # Permite en TODOS los recursos (no es lo ideal, pero es un ejemplo)
      }
    ]
  })
}

# 5. Adjuntar la Política al GRUPO
resource "aws_iam_group_policy_attachment" "adjuntar_politica_grupo" {
  group      = aws_iam_group.grupo_dev.name
  policy_arn = aws_iam_policy.politica_s3_readonly.arn
}
```
{% endcode %}

### 6. Paso 4: Definir las Salidas (outputs.tf)

Expongamos el ARN (Amazon Resource Name) del nuevo usuario y de la política.

{% code title="outputs.tf" %}
```hcl
output "arn_del_usuario" {
  description = "El ARN del nuevo usuario IAM"
  value       = aws_iam_user.usuario_dev.arn
}

output "arn_de_la_politica" {
  description = "El ARN de la nueva política S3"
  value       = aws_iam_policy.politica_s3_readonly.arn
}
```
{% endcode %}

### 7. Paso: Ejecutar el Flujo de Trabajo

{% stepper %}
{% step %}
### terraform init

* Terraform descargará el provider de AWS.
{% endstep %}

{% step %}
### terraform plan

* Revise la salida. Verá que Terraform planea crear 5 recursos:
  *
    * aws\_iam\_user.usuario\_dev
  *
    * aws\_iam\_group.grupo\_dev
  *
    * aws\_iam\_group\_membership.pertenencia\_grupo
  *
    * aws\_iam\_policy.politica\_s3\_readonly
  *
    * aws\_iam\_group\_policy\_attachment.adjuntar\_politica\_grupo
{% endstep %}

{% step %}
### terraform apply

* Escriba yes para aprobar.
* En unos segundos, ¡se creará toda su estructura de IAM!
{% endstep %}

{% step %}
### Verificación

* Vaya a la consola de AWS -> IAM.
* Verifique que el usuario dev-juan-perez existe.
* Verifique que el grupo grupo-desarrolladores existe.
* Mire las Pertenencias (Memberships) del grupo y vea al usuario.
* Mire los Permisos (Permissions) del grupo y vea la politica-s3-solo-lectura adjunta.
{% endstep %}

{% step %}
### terraform destroy

* Escriba yes para limpiar todos los recursos.
{% endstep %}
{% endstepper %}

## Módulo 4.3: El Generador de Políticas: data "aws\_iam\_policy\_document"

### 1. El Problema: JSON dentro de HCL es Complicado

En el laboratorio anterior, escribimos nuestra política de IAM usando jsonencode:

{% code title="Ejemplo (jsonencode)" %}
```hcl
policy = jsonencode({
  Version = "2012-10-17",
  Statement = [
    {
      Effect = "Allow",
      Action = ["s3:Get*"],
      Resource = "*"
    }
  ]
})
```
{% endcode %}

Esto funciona, pero tiene problemas:

1. **Sintaxis Engorrosa:** Su editor de código (VS Code) piensa que está escribiendo HCL, no JSON, por lo que no le ayuda con el formato.
2. **Errores de Comillas:** Es muy fácil olvidar una coma o una comilla y el jsonencode fallará.
3. **No es Dinámico:** ¿Qué pasa si el Resource (ej. el ARN de un bucket S3) es algo que _también_ estamos creando en Terraform? Interpolar (${...}) cadenas dentro de JSON es muy complicado y propenso a errores.

### 2. La Solución: data (Fuentes de Datos)

Terraform tiene un tipo de bloque especial llamado data.

* Un bloque resource **CREA** algo nuevo.
* Un bloque data **LEE** o **COMPUTA** algo que ya existe o que se puede generar.

Terraform proporciona un "Data Source" especial llamado aws\_iam\_policy\_document. Este bloque nos permite **escribir la política de IAM usando sintaxis HCL**, y Terraform la compilará en el JSON correcto por nosotros.

### 3. La Forma Correcta: HCL Nativo

Veamos cómo reescribir la política de solo lectura de S3 usando esta técnica.

Forma Antigua (jsonencode): ya vista arriba.

Forma Nueva y Recomendada (data):

{% code title="data + aws_iam_policy_document" %}
```hcl
# 1. Definimos la política usando bloques HCL
data "aws_iam_policy_document" "politica_s3_readonly_doc" {
  statement {
    sid    = "PermitirLecturaS3"
    effect = "Allow"
    actions = [
      "s3:Get*",
      "s3:List*"
    ]
    resources = [
      "*" # Sigue siendo un mal ejemplo, ¡pero ahora es dinámico!
    ]
  }
}

# 2. Creamos el recurso de política usando el data source
resource "aws_iam_policy" "politica_s3_readonly_nueva" {
  name   = "politica-s3-solo-lectura-hcl"
  policy = data.aws_iam_policy_document.politica_s3_readonly_doc.json
}
```
{% endcode %}

### 4. El Poder de la Dinámica

Supongamos que queremos que esta política solo se aplique a un bucket S3 específico que también estamos creando en este main.tf.

{% code title="Ejemplo dinámico con bucket" %}
```hcl
# Creamos un bucket
resource "aws_s3_bucket" "bucket_de_logs" {
  bucket = "mis-logs-de-app-12345"
}

# Creamos la política de forma dinámica
data "aws_iam_policy_document" "politica_de_logs_doc" {
  statement {
    effect   = "Allow"
    actions  = [ "s3:GetObject" ]
    resources = [
      aws_s3_bucket.bucket_de_logs.arn,
      "${aws_s3_bucket.bucket_de_logs.arn}/*"
    ]
  }
}

resource "aws_iam_policy" "politica_de_logs" {
  name   = "politica-lectura-logs-especificos"
  policy = data.aws_iam_policy_document.politica_de_logs_doc.json
}
```
{% endcode %}

Beneficios:

* Código HCL nativo.
* Autocompletado y validación.
* Referencias dinámicas a otros recursos de Terraform.

## Módulo 4.4: Lab Fundamental - Roles de IAM para EC2

### 1. Objetivo del Laboratorio

Crear un Rol de IAM que permita a una instancia EC2 (un servicio) acceder a S3 (otro servicio) de forma segura, **sin usar claves de acceso (Access Keys)**.

Para crear un Rol, debemos definir **DOS** políticas:

1. **Política de Confianza (Trust Policy):** ¿**QUIÉN** puede asumir este rol? (Responde: "El servicio EC2").
2. **Política de Identidad (Identity Policy):** ¿**QUÉ** puede hacer este rol una vez asumido? (Responde: "Leer de S3").

### 2. Analogía: El Portero y la Lista VIP

* **Política de Confianza:** Es el **portero** de una discoteca. Su _único_ trabajo es comprobar la lista de invitados y decidir si ec2.amazonaws.com tiene permiso para "entrar" (asumir el rol).
* **Política de Identidad:** Es la **pulsera VIP** que recibe una vez dentro. Esta pulsera es la que le da los permisos (ej. "acceso a la zona VIP", "bebidas gratis"), que serían nuestros permisos s3:GetObject.

### 3. Paso 1: Definir la Política de Confianza (El Portero)

Usamos data "aws\_iam\_policy\_document" para definir quién puede asumir el rol.

{% code title="main.tf (confianza)" %}
```hcl
# 1. Definir la POLÍTICA DE CONFIANZA (QUIÉN puede asumir el rol)
data "aws_iam_policy_document" "rol_confianza_ec2" {
  statement {
    effect = "Allow"
    actions = ["sts:AssumeRole"]

    principals {
      type        = "Service"
      identifiers = ["ec2.amazonaws.com"]
    }
  }
}
```
{% endcode %}

### 4. Paso 2: Crear el Rol

Ahora creamos el aws\_iam\_role y le pasamos la Política de Confianza (el portero) a su argumento assume\_role\_policy.

{% code title="main.tf (rol)" %}
```hcl
# 2. Crear el Rol
resource "aws_iam_role" "rol_ec2_s3_readonly" {
  name               = "rol-ec2-s3-solo-lectura"
  assume_role_policy = data.aws_iam_policy_document.rol_confianza_ec2.json
}
```
{% endcode %}

### 5. Paso 3: Definir la Política de Identidad (La Pulsera VIP)

Ahora definimos qué podrá hacer el rol (leer de S3).

{% code title="main.tf (política identidad)" %}
```hcl
# 3. Definir la POLÍTICA DE IDENTIDAD (QUÉ puede hacer el rol)
data "aws_iam_policy_document" "politica_identidad_s3" {
  statement {
    effect   = "Allow"
    actions  = [ "s3:Get*", "s3:List*" ]
    resources = ["*"] # De nuevo, para un lab
  }
}

# Creamos la política gestionada
resource "aws_iam_policy" "politica_ec2_s3_readonly" {
  name   = "politica-ec2-s3-solo-lectura"
  policy = data.aws_iam_policy_document.politica_identidad_s3.json
}
```
{% endcode %}

### 6. Paso 4: Adjuntar la Política de Identidad al Rol

Conectamos la "Pulsera VIP" (Política de Identidad) al "Rol".

{% code title="main.tf (attach policy)" %}
```hcl
# 4. Adjuntar la Política de Identidad al Rol
resource "aws_iam_role_policy_attachment" "adjuntar_s3_al_rol" {
  role       = aws_iam_role.rol_ec2_s3_readonly.name
  policy_arn = aws_iam_policy.politica_ec2_s3_readonly.arn
}
```
{% endcode %}

### 7. Paso 5: Crear el Perfil de Instancia (Instance Profile)

Las instancias EC2 no pueden asumir un Rol directamente; asumen un **Perfil de Instancia**, que es un contenedor para un Rol.

{% code title="main.tf (instance profile)" %}
```hcl
# 5. Crear el Perfil de Instancia para EC2
resource "aws_iam_instance_profile" "perfil_ec2" {
  name = "perfil-instancia-s3-solo-lectura"
  role = aws_iam_role.rol_ec2_s3_readonly.name
}
```
{% endcode %}

### 8. Paso: Ejecutar y Verificar

{% stepper %}
{% step %}
### terraform init & terraform apply

* Ejecute terraform init y luego terraform apply.
* Responda yes para aplicar.
{% endstep %}

{% step %}
### Verificación en la consola

* Vaya a AWS -> IAM -> Roles y encuentre rol-ec2-s3-solo-lectura.
* Verifique la pestaña **Permisos**: debería ver politica-ec2-s3-solo-lectura.
* Verifique la pestaña **Relaciones de confianza**: ec2.amazonaws.com debe aparecer como principal de confianza.
* Vaya a **Perfiles de Instancia** y verifique que perfil-instancia-s3-solo-lectura existe.
{% endstep %}

{% step %}
### terraform destroy

* Limpie los recursos con terraform destroy.
{% endstep %}
{% endstepper %}

### 9. (Adelanto Módulo 6) ¿Cómo se usa esto?

Cuando creemos una instancia EC2 en el Módulo 6, pasaremos el nombre del perfil de instancia:

{% code title="Ejemplo: aws_instance" %}
```
```
{% endcode %}

```hcl
resource "aws_instance" "servidor_web" {
  ami               = "ami-..."
  instance_type     = "t2.micro"
  iam_instance_profile = aws_iam_instance_profile.perfil_ec2.name
}
```

La instancia arrancará, asumirá automáticamente este rol y obtendrá credenciales temporales que le permitirán leer de S3 sin necesidad de claves de acceso.

## Módulo 4.5: Buenas Prácticas de Credenciales, MFA y Estado

### 1. El Problema: Gestión de Secretos

El mayor riesgo de seguridad en IaC es la **gestión de secretos**. ¿Cómo manejamos contraseñas, claves de API y otros datos sensibles?

#### Anti-Patrón 1: Secretos en Texto Plano

¡NUNCA HAGA ESTO!

{% code title="Ejemplo MAL (hardcode)" %}
```hcl
resource "aws_db_instance" "db" {
  username = "admin"
  password = "MiContraseñaSuperSecreta123" # Hardcodeado en Git
}
```
{% endcode %}

#### Anti-Patrón 2: Secretos en Variables (sin cuidado)

{% code title="variables.tf (malo)" %}
```hcl
variable "db_password" {
  type = string
}
```
{% endcode %}

{% code title="terraform.tfvars (malo)" %}
```hcl
db_password = "MiContraseñaSuperSecreta123" # Hardcodeado en Git
```
{% endcode %}

### 2. Buena Práctica 1: sensitive = true

Terraform nos da una herramienta para mitigar la exposición accidental de secretos en los logs.

{% code title="variables.tf (sensitive)" %}
```hcl
variable "db_password" {
  description = "Contraseña para la BBDD"
  type        = string
  sensitive   = true
}
```
{% endcode %}

* ¿Qué hace esto? Cuando ejecute plan o apply, Terraform mostrará: password = (sensitive value)
* **ADVERTENCIA:** Esto _NO_ cifra el secreto. El secreto **seguirá estando en texto plano en terraform.tfstate**.

### 3. Buena Práctica 2: Seguridad del Estado (.tfstate)

Dado que el estado puede contener secretos en texto plano, protegerlo es su máxima prioridad.

1. **.gitignore (Mandatorio):** Añada \*.tfstate y \*.tfvars a su .gitignore.
2. **Backend Remoto Seguro (Módulo 11):** Use un backend remoto (por ejemplo S3) con **Cifrado del Lado del Servidor (SSE)** habilitado.
3. **No usar aws\_iam\_access\_key:** Terraform ofrece aws\_iam\_access\_key para crear claves. **No lo use.** Terraform almacenará secret\_access\_key en texto plano en el estado. Es mejor que los usuarios generen sus claves manualmente o usen un sistema como HashiCorp Vault.

### 4. Buena Práctica 3: Forzar la Autenticación Multifactor (MFA)

No podemos _configurar_ un dispositivo MFA virtual para un usuario con Terraform (es un proceso interactivo), pero podemos (y debemos) **forzar su uso** mediante políticas.

Objetivo: Crear una política que Deniegue (Deny) _todas_ las acciones si el usuario no se ha autenticado con MFA.

{% code title="main.tf (política forzar MFA)" %}
```hcl
data "aws_iam_policy_document" "forzar_mfa_doc" {
  statement {
    sid    = "DenegarTodoSinMFA"
    effect = "Deny"

    # "not_actions" deniega cualquier acción excepto las listadas
    not_actions = [
      "iam:CreateVirtualMFADevice",
      "iam:EnableMFADevice",
      "iam:ListMFADevices",
      "iam:ResyncMFADevice",
      "sts:GetSessionToken"
    ]

    resources = ["*"]

    condition {
      test     = "BoolIfExists"
      variable = "aws:MultiFactorAuthPresent"
      values   = ["false"]
    }
  }
}

resource "aws_iam_policy" "forzar_mfa_policy" {
  name   = "politica-forzar-mfa"
  policy = data.aws_iam_policy_document.forzar_mfa_doc.json
}

# Ejemplo de adjunto a un grupo existente (asumiendo aws_iam_group.grupo_admin)
resource "aws_iam_group_policy_attachment" "adjuntar_mfa_al_grupo" {
  group      = aws_iam_group.grupo_admin.name
  policy_arn = aws_iam_policy.forzar_mfa_policy.arn
}
```
{% endcode %}

* Resultado: Un usuario de este grupo que intente usar la consola sin MFA recibirá "Acceso Denegado". Las únicas acciones permitidas son las necesarias para configurar su propio dispositivo MFA.



## Módulo 4.6: Cifrado Básico con AWS KMS

### 1. ¿Qué es el Cifrado?

El cifrado (o encriptación) es el proceso de codificar datos para que solo las partes autorizadas puedan leerlos.

* Cifrado en Tránsito (In-Transit): ej. HTTPS/TLS.
* Cifrado en Reposo (At-Rest): ej. cifrar volúmenes EBS o buckets S3.

AWS KMS se centra en el cifrado en reposo.

### 2. ¿Qué es AWS KMS (Key Management Service)?

* Servicio gestionado para creación y control de claves criptográficas.
* Resuelve la compleja gestión de claves (creación, rotación, almacenamiento, control de acceso).
* Utiliza HSM validados por FIPS 140-2.

Conceptos clave:

* CMK (Customer Master Key): clave maestra.
* Clave de Datos (Data Key): clave generada por KMS para cifrar datos, KMS devuelve la versión en texto plano y la versión cifrada por la CMK.

### 3. Cifrado con Terraform y KMS

Servicios AWS se integran con KMS; pasos generales:

1. Crear una CMK (opcional).
2. Indicar al servicio (S3, EBS, RDS) que use la CMK.

Lab: Crear una Clave KMS y Cifrar un Bucket S3

#### Paso 1: Crear la Clave KMS (main.tf)

{% code title="main.tf (kms key)" %}
```hcl
# 1. Crear la Clave Maestra de Cliente (CMK)
resource "aws_kms_key" "mi_clave" {
  description                = "Clave KMS para cifrar el bucket S3"
  deletion_window_in_days    = 10
  tags = {
    Name = "clave-s3"
  }
}
```
{% endcode %}

#### Paso 2: Crear el Bucket S3 y Aplicar el Cifrado (main.tf)

{% code title="main.tf (bucket + cifrado)" %}
```hcl
# 2. Crear el Bucket S3
resource "aws_s3_bucket" "bucket_cifrado" {
  bucket = "mi-bucket-super-secreto-kms-123"
}

# 3. Aplicar la configuración de cifrado al bucket
resource "aws_s3_bucket_server_side_encryption_configuration" "cifrado_del_bucket" {
  bucket = aws_s3_bucket.bucket_cifrado.id

  rule {
    apply_server_side_encryption_by_default {
      kms_master_key_id = aws_kms_key.mi_clave.arn
      sse_algorithm     = "aws:kms"
    }
  }
}
```
{% endcode %}

#### Paso 3: (Importante) Dar Permisos para Usar la Clave

Por defecto, nadie puede usar la clave. Debe definir la política de la clave KMS.

{% code title="main.tf (política KMS simplificada)" %}
```hcl
data "aws_iam_policy_document" "politica_clave_kms" {
  statement {
    sid    = "PermitirAdministracionDeLaCuenta"
    effect = "Allow"
    actions = ["kms:*"]
    resources = ["*"]

    principals {
      type        = "AWS"
      identifiers = ["arn:aws:iam::111122223333:root"] # Sustituya por su account ID
    }
  }
}

resource "aws_kms_key_policy" "politica_clave" {
  key_id = aws_kms_key.mi_clave.id
  policy = data.aws_iam_policy_document.politica_clave_kms.json
}
```
{% endcode %}

#### Paso: Ejecutar

{% stepper %}
{% step %}
### terraform init & terraform apply

* Ejecute terraform init y luego terraform apply.
* Responda yes para aplicar.
{% endstep %}

{% step %}
### Verificación en la consola

* Vaya a AWS -> KMS y verá su nueva clave.
* Vaya a S3 -> su bucket -> Propiedades. Verá que el cifrado por defecto está activado usando su clave KMS.
{% endstep %}

{% step %}
### terraform destroy

* Limpie los recursos. Es posible que necesite eliminar objetos del bucket antes de destruirlo.
{% endstep %}
{% endstepper %}

## Módulo 4.7: Auditoría con AWS CloudTrail (Lab)

### 1. ¿Qué es AWS CloudTrail?

Si IAM es el "guardia de seguridad", **CloudTrail es el "libro de registro"**.

* Registra todas las llamadas API realizadas en su cuenta.
* Útil para auditoría y forense (quién, cuándo, qué acción, recurso, IP).

### 2. Lab: Crear un "Trail" (Rastro de Auditoría)

Crear un bucket S3 para los logs y un Trail de CloudTrail que escriba en él.

#### Paso 1: Crear el Bucket S3 para los Logs (main.tf)

Este bucket debe tener una política que permita a CloudTrail escribir en él.

{% code title="main.tf (bucket para logs)" %}
```hcl
# 1. Bucket S3 para almacenar los logs de CloudTrail
resource "aws_s3_bucket" "bucket_cloudtrail" {
  bucket = "mi-rastro-de-auditoria-12345"
}

# 2. Política de Bucket: permite a CloudTrail escribir
data "aws_iam_policy_document" "politica_bucket_cloudtrail" {
  # Permite a CloudTrail verificar que el bucket existe
  statement {
    sid     = "AWSCloudTrailAclCheck"
    effect  = "Allow"
    actions = ["s3:GetBucketAcl"]
    resources = [aws_s3_bucket.bucket_cloudtrail.arn]

    principals {
      type        = "Service"
      identifiers = ["cloudtrail.amazonaws.com"]
    }
  }

  # Permite a CloudTrail escribir los objetos de log
  statement {
    sid     = "AWSCloudTrailWrite"
    effect  = "Allow"
    actions = ["s3:PutObject"]
    resources = ["${aws_s3_bucket.bucket_cloudtrail.arn}/*"]

    principals {
      type        = "Service"
      identifiers = ["cloudtrail.amazonaws.com"]
    }
  }
}

# 3. Adjuntar la política al bucket
resource "aws_s3_bucket_policy" "politica_bucket" {
  bucket = aws_s3_bucket.bucket_cloudtrail.id
  policy = data.aws_iam_policy_document.politica_bucket_cloudtrail.json
}
```
{% endcode %}

#### Paso 2: Crear el Trail de CloudTrail (main.tf)

{% code title="main.tf (cloudtrail)" %}
```hcl
# 4. Crear el Trail de CloudTrail
resource "aws_cloudtrail" "mi_rastro" {
  name                        = "rastro-de-organizacion"
  s3_bucket_name              = aws_s3_bucket.bucket_cloudtrail.id
  depends_on                  = [ aws_s3_bucket_policy.politica_bucket ]
  is_multi_region_trail       = true
  include_global_service_events = true
}
```
{% endcode %}

### 3. Paso: Ejecutar y Verificar

{% stepper %}
{% step %}
### terraform init & terraform apply

* Terraform creará el bucket, aplicará la política y luego creará el Trail.
{% endstep %}

{% step %}
### Verificación

* Vaya a AWS -> CloudTrail -> Trails y verá su rastro-de-organizacion.
* Vaya a S3 -> mi-rastro-de-auditoria-12345. En 5-15 minutos empezarán a aparecer archivos .json.gz con los logs de la API.
{% endstep %}

{% step %}
### terraform destroy

* Limpie los recursos. Nota: es posible que necesite borrar los archivos de log del bucket S3 manualmente antes de que Terraform pueda destruir el bucket.
{% endstep %}
{% endstepper %}

### 4. Material de Apoyo

#### 🎥 Videos Recomendados

* [**¿Qué es AWS CloudTrail? (Video Oficial)**](https://www.google.com/search?q=https://www.youtube.com/watch?v%3Dk-N43mp-q-Q)

#### 📖 Documentos Clave

* [**Documentación de aws\_cloudtrail**](https://www.google.com/search?q=%5Bhttps://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudtrail%5D\(https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudtrail\))
* [**Documentación de aws\_s3\_bucket\_policy**](https://www.google.com/search?q=%5Bhttps://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket_policy%5D\(https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket_policy\))
