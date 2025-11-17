# Módulo 7 - ALMACENAMIENTO EN AWS CON TERRAFORM

## Módulo 7.1: Lab - Creación de Buckets S3

### 1. Introducción al aws\_s3\_bucket

El **Bucket S3** es el recurso de almacenamiento de objetos más fundamental de AWS. Es el "contenedor" donde almacenaremos archivos, logs, activos de sitios web, backups y, como veremos en el Módulo 11, nuestro propio estado de Terraform.

El recurso de Terraform aws\_s3\_bucket es engañosamente simple al principio, pero tiene docenas de sub-recursos para gestionar configuraciones complejas como políticas, versionado, cifrado y sitios web estáticos.

### 2. Objetivo del Laboratorio

Crear un bucket S3 simple, privado y correctamente etiquetado, listo para ser configurado en los siguientes laboratorios.

### 3. Estructura de Archivos

* Cree una nueva carpeta de proyecto (ej. terraform-s3).
* Cree los archivos estándar: providers.tf, variables.tf, main.tf, outputs.tf.

### 4. Paso 1: Configurar Providers y Variables

```hcl
# providers.tf
terraform {
  required_version = ">= 1.9.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}
```

```hcl
# variables.tf
variable "aws_region" {
  description = "Región de AWS para el despliegue"
  type        = string
  default     = "us-east-1"
}

variable "nombre_del_bucket" {
  description = "El nombre base para el bucket S3 (se le añadirá un sufijo)"
  type        = string
  default     = "mi-app-data"
}

variable "entorno" {
  description = "Entorno (dev, test, prod)"
  type        = string
  default     = "dev"
}
```

### 5. Paso 2: Crear el Bucket S3 (main.tf)

Los nombres de los buckets S3 deben ser **únicos globalmente**. Usaremos el recurso random\_string para garantizar la unicidad.

```hcl
# providers.tf (¡AÑADIR UN NUEVO PROVIDER!)
terraform {
  # ... (bloque aws) ...
  required_providers {
    # ...
    # Añadimos el provider 'random'
    random = {
      source  = "hashicorp/random"
      version = "~> 3.0"
    }
  }
}
```

```hcl
# main.tf

# 1. Generar un sufijo aleatorio
resource "random_string" "sufijo_bucket" {
  length  = 8
  special = false  # Sin caracteres especiales
  upper   = false  # Solo minúsculas
}

# 2. Crear el Bucket S3
resource "aws_s3_bucket" "mi_bucket" {
  # Concatenamos el nombre de la variable y el sufijo aleatorio
  # ej: "mi-app-data-dev-a1b2c3d4"
  bucket = "${var.nombre_del_bucket}-${var.entorno}-${random_string.sufijo_bucket.result}"

  tags = {
    Name    = var.nombre_del_bucket
    Entorno = var.entorno
  }
}
```

### 6. Paso 3: Exponer las Salidas (outputs.tf)

```hcl
# outputs.tf
output "bucket_id" {
  description = "El nombre (ID) único global del bucket S3"
  value       = aws_s3_bucket.mi_bucket.id
}

output "bucket_arn" {
  description = "El ARN del bucket S3"
  value       = aws_s3_bucket.mi_bucket.arn
}
```

### 7. Paso 4: Ejecutar y Verificar

{% stepper %}
{% step %}
### Terraform: inicializar y aplicar

* terraform init: ¡Importante! Terraform descargará el nuevo provider hashicorp/random además del de aws.
* terraform plan: Verá un plan para crear random\_string.sufijo\_bucket y aws\_s3\_bucket.mi\_bucket.
* terraform apply: Escriba yes.
{% endstep %}

{% step %}
### Verificación en la consola AWS

* Vaya a la Consola de AWS -> S3.
* Verá su nuevo bucket con el nombre aleatorio.
* Haga clic en él. Vaya a la pestaña "Permisos".
* Verá que, por defecto, el bucket tiene "Bloquear todo el acceso público: Activado".
{% endstep %}

{% step %}
### Limpieza

* terraform destroy: Limpie los recursos.
{% endstep %}
{% endstepper %}

***

## Módulo 7.2: Lab - Políticas de Acceso a S3 (Seguridad)

### 1. Introducción: Las Capas de Seguridad de S3

S3 tiene un modelo de seguridad increíblemente granular (y complejo). La seguridad se aplica en varias capas:

1. **Políticas de IAM (Módulo 4):** ¿Qué puede hacer un Usuario o Rol?
2. **Bloqueo de Acceso Público (BAP):** Un "interruptor maestro" a nivel de bucket (o de cuenta).
3. **Políticas de Bucket (Bucket Policies):** Política JSON adjunta al bucket.
4. **Listas de Control de Acceso (ACLs):** Método antiguo; buena práctica: desactivarlas.

### 2. Objetivo del Laboratorio

1. Gestionar explícitamente el **Bloqueo de Acceso Público (BAP)** usando aws\_s3\_bucket\_public\_access\_block.
2. Crear una **Política de Bucket** que fuerce el cifrado (HTTPS) en todas las peticiones.

### 3. Paso 1: El Bloqueo de Acceso Público (main.tf)

Aunque AWS lo activa por defecto, es buena práctica definirlo explícitamente.

```hcl
# main.tf (añadir al lab 7.1)
# 1. Gestionar explícitamente el Bloqueo de Acceso Público
resource "aws_s3_bucket_public_access_block" "mi_bucket_pab" {
  bucket                    = aws_s3_bucket.mi_bucket.id
  block_public_acls         = true
  block_public_policy       = true
  ignore_public_acls        = true
  restrict_public_buckets   = true
}
```

### 4. Paso 2: Deshabilitar ACLs (main.tf)

```hcl
# main.tf (añadir)
# 2. Deshabilitar las ACLs
resource "aws_s3_bucket_ownership_controls" "mi_bucket_oc" {
  bucket = aws_s3_bucket.mi_bucket.id

  rule {
    # Establece que el 'Propietario del Bucket' es dueño de todos los objetos.
    object_ownership = "BucketOwnerEnforced"
  }
}
```

### 5. Paso 3: Forzar HTTPS (Política de Bucket) (main.tf)

Esta política deniega cualquier acción si la conexión no usa HTTPS.

```hcl
# main.tf (añadir)
# 3. Definir la Política de Bucket (usando 'data' como aprendimos)
data "aws_iam_policy_document" "politica_forzar_https" {
  statement {
    sid    = "ForzarSoloHTTPS"
    effect = "Deny"

    # ¡Denegar si no es seguro!
    actions   = ["s3:*"]
    resources = [
      aws_s3_bucket.mi_bucket.arn,
      "${aws_s3_bucket.mi_bucket.arn}/*"
    ]

    # El 'Principal' es "todos"
    principals {
      type        = "*"
      identifiers = ["*"]
    }

    # Denegar si 'aws:SecureTransport' NO es 'true'
    condition {
      test     = "Bool"
      variable = "aws:SecureTransport"
      values   = ["false"]
    }
  }
}

# 4. Adjuntar la Política de Bucket
resource "aws_s3_bucket_policy" "mi_bucket_policy" {
  bucket = aws_s3_bucket.mi_bucket.id
  policy = data.aws_iam_policy_document.politica_forzar_https.json
}
```

### 6. Paso 4: Ejecutar y Verificar

{% stepper %}
{% step %}
* terraform init (por si acaso).
* terraform apply: Escriba yes. Terraform añadirá los nuevos recursos (BAP, OC, Política) al bucket existente.
{% endstep %}

{% step %}
Verificación en la consola AWS:

* Vaya a la Consola de AWS -> S3 -> su bucket.
* Pestaña "Permisos":
  * Verá que el "Bloqueo de acceso público" está "Activado" (gestionado por Terraform).
  * Verá que la "Propiedad de objetos" es "BucketOwnerEnforced".
  * Verá la "Política de bucket" con el JSON que deniega el tráfico HTTP.
{% endstep %}

{% step %}
* terraform destroy: Limpie todo.
{% endstep %}
{% endstepper %}

***

## Módulo 7.3: Gestión de Objetos: Versionado y Replicación

### 1. ¿Qué es el Versionado de S3?

* Evita la pérdida por sobrescritura o eliminación accidental.
* aws\_s3\_bucket\_versioning mantiene versiones previas y usa "Delete Markers" en borrados.

### 2. ¿Qué es la Replicación (CRR)?

* Copia objetos (y versiones) de un bucket en una Región a otro bucket en otra Región.
* Beneficios: Recuperación de desastres y reducción de latencia para usuarios en otras regiones.

### 3. Lab: Habilitar Versionado y Replicación

**Objetivo:** Configurar un bucket "fuente" y uno "destino" y habilitar versionado y replicación.

#### Paso 1: Configurar Múltiples Providers (providers.tf)

```hcl
# providers.tf (MODIFICAR)
terraform {
  # ... (mismos required_providers) ...
}

# Provider Primario (Este)
provider "aws" {
  region = "us-east-1"
  alias  = "primario"
}

# Provider Secundario (Destino)
provider "aws" {
  region = "eu-west-1" # Irlanda
  alias  = "secundario"
}
```

#### Paso 2: Crear los Dos Buckets (main.tf)

```hcl
# main.tf

# 1. Bucket FUENTE (en us-east-1)
resource "aws_s3_bucket" "fuente" {
  provider = aws.primario
  bucket   = "mi-bucket-fuente-crr-12345"
}

# 2. Bucket DESTINO (en eu-west-1)
resource "aws_s3_bucket" "destino" {
  provider = aws.secundario
  bucket   = "mi-bucket-destino-crr-12345"
}

# 3. HABILITAR VERSIONADO (Obligatorio para Replicación)
resource "aws_s3_bucket_versioning" "fuente_versionado" {
  provider = aws.primario
  bucket   = aws_s3_bucket.fuente.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_versioning" "destino_versionado" {
  provider = aws.secundario
  bucket   = aws_s3_bucket.destino.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

#### Paso 3: Configurar el Rol de IAM para la Replicación

```hcl
# main.tf (continuación)

# 4. Rol de IAM que S3 asumirá
data "aws_iam_policy_document" "confianza_s3" {
  statement {
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["s3.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "rol_replicacion" {
  provider            = aws.primario
  name                = "rol-s3-replicacion-crr"
  assume_role_policy  = data.aws_iam_policy_document.confianza_s3.json
}

# 5. Política de Identidad: da permisos al rol
data "aws_iam_policy_document" "politica_replicacion_doc" {
  # Permite leer de la fuente
  statement {
    actions = ["s3:Get*", "s3:List*"]
    resources = [
      aws_s3_bucket.fuente.arn,
      "${aws_s3_bucket.fuente.arn}/*"
    ]
  }

  # Permite escribir en el destino
  statement {
    actions = ["s3:ReplicateObject", "s3:ReplicateDelete"]
    resources = ["${aws_s3_bucket.destino.arn}/*"]
  }
}

resource "aws_iam_policy" "politica_replicacion" {
  provider = aws.primario
  name     = "politica-s3-replicacion-crr"
  policy   = data.aws_iam_policy_document.politica_replicacion_doc.json
}

# 6. Adjuntar la política al rol
resource "aws_iam_role_policy_attachment" "adjuntar_replicacion" {
  provider   = aws.primario
  role       = aws_iam_role.rol_replicacion.name
  policy_arn = aws_iam_policy.politica_replicacion.arn
}
```

#### Paso 4: Configurar la Replicación (main.tf)

```hcl
# main.tf (continuación)

# 7. Configuración de la Replicación
resource "aws_s3_bucket_replication_configuration" "replicar" {
  provider = aws.primario

  role   = aws_iam_role.rol_replicacion.arn
  bucket = aws_s3_bucket.fuente.id

  rule {
    id     = "regla-replicacion-total"
    status = "Enabled"

    destination {
      bucket = aws_s3_bucket.destino.arn
    }

    filter {}
  }
}
```

### 4. Paso 5: Ejecutar y Verificar

{% stepper %}
{% step %}
* terraform init (para los providers alias).
* terraform apply: Escriba yes.
{% endstep %}

{% step %}
Verificación:

* S3 -> Bucket fuente. Pestaña "Gestión": verá Versionado activado y la Regla de Replicación.
* S3 -> Bucket destino (en eu-west-1): verá Versionado activado.
* Prueba: suba test.txt al bucket fuente; espere y verifique que aparece en el destino.
{% endstep %}

{% step %}
Limpieza:

* terraform destroy: tendrá que vaciar buckets o usar -target en algunos casos.
{% endstep %}
{% endstepper %}

Nota del formador: introduce providers múltiples y la dependencia de servicio (S3 necesita un rol para replicar).

***

## Módulo 7.4: Lab - Configuración de EBS para Instancias EC2

### 1. Introducción: Almacenamiento de Bloques (EBS)

Opciones para añadir discos:

1. aws\_ebs\_volume + aws\_volume\_attachment.
2. ebs\_block\_device dentro de aws\_instance o aws\_launch\_template (usaremos esta opción).

### 2. Objetivo del Laboratorio

Modificar aws\_launch\_template para:

1. Aumentar el tamaño del volumen raíz a 30GB.
2. Añadir un segundo volumen EBS de 50GB.

### 3. Paso 1: Modificar la Plantilla de Lanzamiento (main.tf)

```hcl
# main.tf (MODIFICAR la plantilla del Módulo 6.5)
resource "aws_launch_template" "plantilla_app_web" {
  name                     = "plantilla-app-web-nginx"
  image_id                 = data.aws_ami.amazon_linux_2.id
  instance_type            = "t2.micro"
  key_name                 = aws_key_pair.mi_clave.key_name
  vpc_security_group_ids   = [aws_security_group.app.id]
  user_data                = filebase64("install_nginx.sh")

  # Gestionamos explícitamente los discos
  block_device_mappings {
    device_name = "/dev/xvda" # raíz
    ebs {
      volume_size           = 30  # raíz: 30 GB
      volume_type           = "gp3"
      delete_on_termination = true
    }
  }

  block_device_mappings {
    device_name = "/dev/sdb" # segundo disco de datos
    ebs {
      volume_size           = 50
      volume_type           = "gp3"
      delete_on_termination = true
      encrypted             = true
    }
  }

  tags = {
    Name = "plantilla-app-web"
  }
}
```

### 4. Paso 2: Ejecutar y Verificar

{% stepper %}
{% step %}
* terraform apply: Escriba yes.
  * Terraform creará una nueva versión de la plantilla.
  * El ASG (si usa $Latest) iniciará un rolling update.
{% endstep %}

{% step %}
Verificación en consola AWS:

* EC2 -> Instancias: seleccione la nueva instancia -> pestaña "Almacenamiento": verá /dev/xvda (30GB) y /dev/sdb (50GB).
{% endstep %}

{% step %}
Conexión (opcional):

* lsblk mostrará los dispositivos.
* El segundo disco está presente pero no formateado; necesita mkfs/mount si desea usarlo (esto puede automatizarse en user\_data).
{% endstep %}

{% step %}
* terraform destroy: Limpie todo.
{% endstep %}
{% endstepper %}

***

## Módulo 7.5: Lab - Montaje de EFS en Instancias

### 1. Introducción: Almacenamiento Compartido (EFS)

EFS es un sistema de archivos que puede montarse simultáneamente en múltiples instancias EC2, resolviendo el problema de compartir archivos entre instancias en un ASG.

### 2. Objetivo del Laboratorio

1. Crear un aws\_efs\_file\_system.
2. Crear aws\_efs\_mount\_target para cada subred privada.
3. Actualizar user\_data de la launch\_template para montar el EFS en /var/www/uploads.

### 3. Paso 1: Crear el Sistema de Archivos EFS (main.tf)

```hcl
# main.tf (añadir)
resource "aws_efs_file_system" "efs_compartido" {
  creation_token = "efs-app-web-compartido"
  encrypted      = true

  lifecycle_policy {
    transition_to_ia = "AFTER_30_DAYS"
  }

  tags = {
    Name = "efs-compartido-app"
  }
}
```

### 4. Paso 2: Crear los Objetivos de Montaje (Mount Targets) (main.tf)

```hcl
# main.tf (continuación)

# 2. Crear un Security Group para EFS
resource "aws_security_group" "efs" {
  name   = "sg-efs"
  vpc_id = aws_vpc.main.id

  # Permite NFS (puerto 2049) desde el SG de la app
  ingress {
    from_port       = 2049
    to_port         = 2049
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "sg-efs"
  }
}

# 3. Crear los Objetivos de Montaje (uno por subred privada)
resource "aws_efs_mount_target" "efs_montaje" {
  count         = length(aws_subnet.private.*.id)
  file_system_id = aws_efs_file_system.efs_compartido.id
  subnet_id     = aws_subnet.private[count.index].id
  security_groups = [aws_security_group.efs.id]
}
```

### 5. Paso 3: Actualizar el Script user\_data (install\_nginx.sh)

```bash
#!/bin/bash
# install_nginx.sh (MODIFICAR)

# Recibimos el ID del EFS como un argumento
EFS_ID=$1

# 1. Instalar Nginx y el cliente NFS (efs-utils)
yum update -y
yum install -y nginx amazon-efs-utils

# 2. Crear el directorio de montaje
mkdir -p /var/www/uploads

# 3. Montar el EFS usando el mount helper de EFS (con TLS)
mount -t efs -o tls ${EFS_ID}:/ /var/www/uploads

# 4. Iniciar Nginx
systemctl start nginx
systemctl enable nginx

# 5. Crear una página de bienvenida en el EFS
echo "<h1>¡Hola, Mundo desde EFS! (Instancia: $(hostname -f))</h1>" > /var/www/uploads/index.html

# 6. (Opcional) Configuración adicional de Nginx para servir /var/www/uploads
```

### 6. Paso 4: Actualizar la Plantilla de Lanzamiento (main.tf)

```hcl
# main.tf (MODIFICAR la 'plantilla_app_web')

resource "aws_launch_template" "plantilla_app_web" {
  # ... (ami, instance_type, key_name, vpc_sg, etc.) ...

  # Usamos 'templatefile' en lugar de 'file' y pasamos el ID del EFS
  user_data = base64encode(
    templatefile("install_nginx.sh", {
      EFS_ID = aws_efs_file_system.efs_compartido.id
    })
  )

  # ... (block_device_mappings, etc.) ...

  # Asegurar que EFS y sus mount targets existan antes de lanzar instancias
  depends_on = [aws_efs_mount_target.efs_montaje]
}
```

### 7. Paso 5: Ejecutar y Verificar

{% stepper %}
{% step %}
* terraform init (por la función templatefile).
* terraform apply: Escriba yes.
{% endstep %}

{% step %}
Verificación:

* EC2 -> ASG: espere a que las instancias estén "healthy".
* EFS: verá el sistema de archivos y los objetivos de montaje en las subredes privadas.
{% endstep %}

{% step %}
Prueba práctica:

* Abra http://app.mi-empresa.com y verá la página index.html desde EFS.
* SSH a la Instancia A:
  *   echo "

      ## Hola desde Instancia A

      " > /var/www/uploads/test.html
* SSH a la Instancia B:
  * cat /var/www/uploads/test.html -> verá el contenido creado en A (compartido).
{% endstep %}

{% step %}
* terraform destroy: Limpie todo.
{% endstep %}
{% endstepper %}

***

## Módulo 7.6: Configuración de Ciclo de Vida en S3

### 1. El Problema: Costes de Almacenamiento

Los datos antiguos ocupan almacenamiento caro si no se gestionan. La solución es moverlos automáticamente a clases más baratas según su antigüedad.

### 2. La Solución: Clases de Almacenamiento y Ciclo de Vida

S3 ofrece múltiples clases (Standard, Standard-IA, Glacier Instant Retrieval, Glacier Flexible Retrieval, Glacier Deep Archive). aws\_s3\_bucket\_lifecycle\_configuration permite mover objetos automáticamente.

### 3. Lab: Transición Automática de Objetos

**Objetivo:** Crear una política que:

1. Mueva logs a Standard-IA tras 30 días.
2. Mueva a Glacier Deep Archive tras 90 días.
3. Elimine tras 7 años (2555 días).

#### Paso 1: Configurar el Bucket (main.tf)

```hcl
# main.tf
# ... (bucket y provider del Lab 7.1) ...

resource "aws_s3_bucket_lifecycle_configuration" "bucket_logs_lifecycle" {
  bucket = aws_s3_bucket.mi_bucket.id

  rule {
    id     = "regla-ciclo-vida-logs"
    status = "Enabled"

    # 1. Transición a Standard-IA
    transition {
      days         = 30
      storage_class = "STANDARD_IA"
    }

    # 2. Transición a Deep Archive
    transition {
      days         = 90
      storage_class = "DEEP_ARCHIVE"
    }

    # 3. Expiración (Borrado)
    expiration {
      days = 2555
    }
  }
}
```

### 4. Paso 2: Ejecutar y Verificar

{% stepper %}
{% step %}
* terraform apply: Escriba yes.
{% endstep %}

{% step %}
Verificación:

* S3 -> su bucket -> Pestaña "Gestión": verá la regla regla-ciclo-vida-logs y la línea de tiempo de transiciones.
{% endstep %}

{% step %}
* terraform destroy.
{% endstep %}
{% endstepper %}

***

## Módulo 7.7: Bloqueo de Eliminación en S3 (Object Lock)

### 1. Introducción: Almacenamiento Inmutable (WORM)

S3 Object Lock aplica reglas WORM: objetos no pueden ser eliminados o sobrescritos durante una retención definida. Útil para cumplimiento normativo.

### 2. Lab: Configurar un Bucket Inmutable

**¡ADVERTENCIA!** Object Lock solo se habilita en la creación del bucket.

#### Paso 1: Crear el Bucket con Object Lock (main.tf)

```hcl
# main.tf (proyecto nuevo)

resource "aws_s3_bucket" "bucket_inmutable" {
  bucket = "bucket-inmutable-cumplimiento-12345"

  # El bloqueo de objetos REQUIERE versionado. Debe definirse DENTRO del bucket.
  versioning {
    enabled = true
  }
}

resource "aws_s3_bucket_object_lock_configuration" "bloqueo" {
  bucket                = aws_s3_bucket.bucket_inmutable.id
  object_lock_enabled   = "Enabled" # 'Enabled' es la única opción
}
```

### 3. Paso 2: Ejecutar y Verificar

{% stepper %}
{% step %}
* terraform apply: Escriba yes.
{% endstep %}

{% step %}
Verificación:

* S3 -> su bucket -> Propiedades: verá "Bloqueo de objetos: Habilitado".
* Puede subir un archivo y aplicar retención (Compliance/Governance).
* Nota: si Terraform crea aws\_s3\_object con retención en modo Compliance, terraform destroy fallará porque el objeto no puede eliminarse.
{% endstep %}

{% step %}
* terraform destroy: en este escenario no habrá objetos con retención aplicada, por lo que el destroy funcionará.
{% endstep %}
{% endstepper %}

***

## Módulo 7.8: Automatización de Backups (Snapshots de EBS)

### 1. El Problema: Backups de EBS

EBS snapshots son backups incrementales almacenados en S3. Queremos automatizarlos.

### 2. La Solución: AWS DLM (Data Lifecycle Manager)

DLM automatiza la creación y retención de snapshots. Usaremos aws\_dlm\_lifecycle\_policy.

### 3. Lab: Crear una Política de DLM

**Objetivo:** Política que:

1. Encuentre volúmenes con etiqueta Backup=True.
2. Cree un snapshot diario a las 2 AM.
3. Retenga los últimos 7 snapshots.

#### Paso 1: Crear el Rol de IAM para DLM (main.tf)

```hcl
# main.tf (proyecto nuevo)

data "aws_iam_policy_document" "confianza_dlm" {
  statement {
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["dlm.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "rol_dlm" {
  name               = "rol-dlm-lifecycle"
  assume_role_policy = data.aws_iam_policy_document.confianza_dlm.json
}

# AWS proporciona una política gestionada ideal para DLM
resource "aws_iam_role_policy_attachment" "dlm_permisos" {
  role       = aws_iam_role.rol_dlm.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSDataLifecycleManagerServiceRole"
}
```

#### Paso 2: Crear la Política de DLM (main.tf)

```hcl
# main.tf (continuación)

resource "aws_dlm_lifecycle_policy" "backup_diario" {
  description  = "Política de backup diario para volúmenes etiquetados"
  iam_role_arn = aws_iam_role.rol_dlm.arn

  policy_details {
    resource_types = ["VOLUME"]

    target_tags = {
      "Backup" = "True"
    }

    schedule {
      name = "schedule-diario-2am"

      # 'cron(minuto hora día-mes mes día-semana año)'
      create_rule {
        cron_expression = "cron(0 2 * * ? *)"
      }

      retain_rule {
        count = 7
      }

      tags_to_add = {
        "SnapshotOwner" = "DLM"
      }
    }
  }

  state = "ENABLED"
}
```

### 4. Paso 3: Ejecutar y Verificar

{% stepper %}
{% step %}
* terraform apply: Escriba yes.
{% endstep %}

{% step %}
Verificación:

* Consola AWS -> EC2 -> Lifecycle Manager: verá backup\_diario.
* Puede etiquetar un volumen con Backup=True y esperar a las 2 AM UTC o ejecutar la política manualmente para probar.
{% endstep %}

{% step %}
* terraform destroy.
{% endstep %}
{% endstepper %}

***

## Módulo 7.9: Buenas Prácticas de Almacenamiento Seguro (Resumen)

Resumen de buenas prácticas por tipo de almacenamiento.

### 1. S3 (Objetos)

* Bloquear Todo: use aws\_s3\_bucket\_public\_access\_block en cada bucket.
* Deshabilitar ACLs: aws\_s3\_bucket\_ownership\_controls con BucketOwnerEnforced.
* Forzar HTTPS: policy que deniegue si aws:SecureTransport=false.
* Versionado: habilite aws\_s3\_bucket\_versioning para datos críticos.
* Cifrado: aws\_s3\_bucket\_server\_side\_encryption\_configuration (aws:kms o KMS personalizado).
* Optimizar Costes: aws\_s3\_bucket\_lifecycle\_configuration.

### 2. EBS (Bloques)

* Cifrar siempre: encrypted = true.
* Usar gp3: mejor rendimiento/coste que gp2.
* delete\_on\_termination: configurar según el tipo de instancia (Cattle vs Pet).
* Automatizar Backups: AWS DLM (aws\_dlm\_lifecycle\_policy).

### 3. EFS (Archivos)

* Cifrar siempre en reposo y en tránsito (mount con TLS).
* Seguridad de red: mount targets en subredes privadas; SG que permita solo 2049/tcp desde SGs de la app.
* Optimizar costes: lifecycle\_policy para EFS-IA.

### 4. Material de Apoyo

* [**Pilar de Seguridad de S3 (AWS Well-Architected)**](https://www.google.com/search?q=https://docs.aws.amazon.com/wellarchitected/latest/storage-lens/security.html)
* [**Mejores Prácticas de Cifrado de EBS (AWS Docs)**](https://www.google.com/search?q=https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-encryption.html)
* [**Mejores Prácticas de Seguridad de EFS (AWS Docs)**](https://www.google.com/search?q=https://docs.aws.amazon.com/efs/latest/ug/security.html)
