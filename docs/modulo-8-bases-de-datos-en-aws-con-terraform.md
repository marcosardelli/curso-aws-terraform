# Módulo 8 - BASES DE DATOS EN AWS CON TERRAFORM

## Módulo 8: BASES DE DATOS EN AWS CON TERRAFORM

## Módulo 8.1: Introducción a RDS (MySQL, PostgreSQL, SQL Server)

### 1. El Problema: Bases de Datos en EC2 (IaaS)

Una forma de tener una base de datos en AWS es lanzar una instancia EC2 (Módulo 6) y ejecutar `yum install mysql`.

Esto es un anti-patrón y una pesadilla operativa, conocida como "infraestructura indiferenciada" (undifferentiated heavy lifting). Usted sería responsable de:

* Instalar y parchear el Sistema Operativo.
* Instalar y parchear el software de la base de datos.
* Configurar la Alta Disponibilidad (Replicación).
* Configurar los Backups (Snapshots).
* Escalar la instancia si se queda sin CPU/RAM.

### 2. La Solución: Amazon RDS (Relational Database Service)

RDS es un servicio gestionado (PaaS - Plataforma como Servicio).

* ¿Qué es? Es un servicio que facilita la configuración, operación y escalado de bases de datos relacionales en la nube.
* ¿Qué problema resuelve? AWS se encarga de todo el "trabajo pesado":
  * Aprovisionamiento del servidor.
  * Parcheo del SO y del motor de la BBDD.
  * Backups automáticos.
  * Replicación Multi-AZ con un solo clic (para Alta Disponibilidad).
  * Escalado sencillo (vertical y horizontal).
* Su Responsabilidad: Solo gestionar su esquema, sus datos y optimizar sus consultas.

### 3. DBMS Soportados

RDS no es una base de datos; es un servicio que gestiona motores de bases de datos populares. Usted puede elegir:

* Amazon Aurora (compatible con MySQL y PostgreSQL).
* PostgreSQL.
* MySQL.
* MariaDB.
* Oracle.
* Microsoft SQL Server.

### 4. Conceptos Clave de RDS

* Instancia de BBDD (DB Instance): servidor de base de datos. Tiene tipo de instancia (ej. db.t3.micro) y almacenamiento (EBS).
* Multi-AZ (Alta Disponibilidad): crea automáticamente una réplica síncrona en otra AZ; failover automático en < 1 minuto.
* Réplicas de Lectura (Read Replicas): copias asíncronas para escalar lecturas.
* Grupo de Subredes de BBDD (DB Subnet Group): recurso que indica en qué subredes/AZs puede operar la instancia. Es obligatorio.

### 5. Material de Apoyo y Siguientes Pasos

* Página Oficial de Amazon RDS: https://aws.amazon.com/es/rds/
* Documentación: ¿Qué es RDS?: https://docs.aws.amazon.com/es\_es/AmazonRDS/latest/UserGuide/Welcome.html

***

## Módulo 8.2: Lab - Creación de Instancias RDS (y Gestión de Secretos)

### 1. Introducción

Este es el laboratorio más importante para la gestión de recursos con estado (stateful). Vamos a crear una instancia de base de datos PostgreSQL.

### 2. El Problema del Secreto: La Contraseña

El recurso `aws_db_instance` requiere un username y un password.

Anti-Patrón 1 (¡Nunca!):

```hcl
resource "aws_db_instance" "db" {
  password = "MiPasswordSecreto123" # Hardcodeado en Git
}
```

Anti-Patrón 2 (Peligroso):

```hcl
variable "db_password" { type = string; sensitive = true }
resource "aws_db_instance" "db" { password = var.db_password }
```

Aunque mejor, la contraseña todavía se almacenará en texto plano en `terraform.tfstate`.

### 3. La Solución: AWS Secrets Manager

Buena práctica: no gestionar la contraseña en el código.

1. Generar una contraseña aleatoria.
2. Almacenar esa contraseña en AWS Secrets Manager.
3. Hacer que `aws_db_instance` obtenga la contraseña directamente desde Secrets Manager.

La contraseña nunca toca nuestro código HCL ni nuestro archivo de estado.

### 4. Objetivo del Laboratorio

1. Crear un secreto en AWS Secrets Manager para guardar una contraseña generada aleatoriamente.
2. Crear una instancia RDS (PostgreSQL) que use ese secreto.

### 5. Paso 1: Configurar Providers (providers.tf)

Archivo example:

```hcl
terraform {
  required_version = ">= 1.9.0"
  required_providers {
    aws = { source = "hashicorp/aws" version = "~> 5.0" }
    random = { source = "hashicorp/random" version = "~> 3.0" }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

### 6. Paso 2: Generar y Almacenar el Secreto (main.tf)

Este proyecto asume que usted ya tiene la VPC y las subredes del Módulo 5. Vamos a leer esos datos.

Ejemplo (fragmento):

```hcl
# LEER DATOS DE LA RED (Módulo 5)
data "aws_vpc" "vpc_existente" {
  tags = { Name = "vpc-principal" }
}

data "aws_subnets" "subredes_datos" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.vpc_existente.id]
  }
  tags = { Name = "subnet-privada-datos-*" }
}

# GESTIÓN DE SECRETOS
resource "random_password" "db_password" {
  length  = 20
  special = true
}

resource "aws_secretsmanager_secret" "db_secret" {
  name = "prod/db/main/password"
  tags = { Entorno = "prod" }
}

resource "aws_secretsmanager_secret_version" "db_secret_version" {
  secret_id     = aws_secretsmanager_secret.db_secret.id
  secret_string = jsonencode({
    username = "admin_db"
    password = random_password.db_password.result
  })
}
```

### 7. Paso 3: Crear la Instancia RDS (main.tf)

Core resource (incompleto a propósito — falta subnet group y security group):

```hcl
resource "aws_db_instance" "db_principal" {
  identifier                     = "db-principal-prod"
  engine                         = "postgres"
  engine_version                 = "15.6"
  instance_class                 = "db.t3.micro"
  allocated_storage              = 20
  storage_type                   = "gp3"
  manage_master_user_password    = true
  master_username                = "admin_db"
  master_user_secret_kms_key_id  = aws_secretsmanager_secret.db_secret.kms_key_id
  depends_on                     = [aws_secretsmanager_secret_version.db_secret_version]
  publicly_accessible            = false
  skip_final_snapshot            = true
  tags = { Name = "db-principal" }
}
```

¡Atención! Este laboratorio es incompleto a propósito. `terraform apply` fallará porque faltan `aws_db_subnet_group` y `vpc_security_group_ids`. El siguiente laboratorio (8.3) soluciona esto.

***

## Módulo 8.3: Lab - Configuración de Subredes (Subnet Group) y Security Groups para RDS

### 1. Introducción

En el laboratorio 8.2, `terraform apply` fallaría porque una instancia RDS debe vivir dentro de una VPC. RDS requiere dos componentes de red explícitos:

* aws\_db\_subnet\_group: lista de subredes (mínimo 2 AZs).
* vpc\_security\_group\_ids: firewall (Security Group) que protege la BBDD.

### 2. Objetivo del Laboratorio

Completar la instancia RDS del Lab 8.2 añadiendo los componentes de red necesarios:

* Crear un `aws_db_subnet_group` usando subredes privadas de datos.
* Crear un `aws_security_group` para la BBDD y adjuntarlo.
* Conectar ambos a la `aws_db_instance`.

### 3. Paso a Paso

{% stepper %}
{% step %}
### Crear el Grupo de Subredes (main.tf)

```hcl
resource "aws_db_subnet_group" "db_subnet_group" {
  name       = "db-subnet-group-prod"
  subnet_ids = data.aws_subnets.subredes_datos.ids
  tags = { Name = "db-subnet-group" }
}
```

Este recurso indica en qué subredes puede operar RDS.
{% endstep %}

{% step %}
### Crear el Security Group (main.tf)

Le damos acceso PostgreSQL (5432) solo desde el Security Group de la aplicación:

```hcl
data "aws_security_group" "sg_app" {
  tags = { Name = "sg-app-web" }
}

resource "aws_security_group" "sg_db" {
  name        = "sg-db-prod"
  description = "Permite acceso a BBDD solo desde la capa de app"
  vpc_id      = data.aws_vpc.vpc_existente.id

  ingress {
    description    = "Acceso PostgreSQL desde App"
    from_port      = 5432
    to_port        = 5432
    protocol       = "tcp"
    security_groups = [data.aws_security_group.sg_app.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "sg-db" }
}
```
{% endstep %}

{% step %}
### Actualizar la Instancia RDS para usar la red (main.tf)

Añadir estas líneas a `aws_db_instance.db_principal`:

```hcl
db_subnet_group_name     = aws_db_subnet_group.db_subnet_group.name
vpc_security_group_ids   = [aws_security_group.sg_db.id]
multi_az                 = true
```

Esto permite Multi-AZ y conecta la instancia a la VPC.
{% endstep %}

{% step %}
### Exponer el Endpoint (outputs.tf)

```hcl
output "db_endpoint" {
  description = "El Endpoint (hostname) de la instancia RDS"
  value       = aws_db_instance.db_principal.endpoint
}

output "db_port" {
  description = "El puerto de la instancia RDS"
  value       = aws_db_instance.db_principal.port
}

output "db_secret_arn" {
  description = "El ARN del secreto en Secrets Manager"
  value       = aws_secretsmanager_secret.db_secret.arn
}
```
{% endstep %}

{% step %}
### Ejecutar y Verificar

1. `terraform init`
2. `terraform apply` (escriba `yes`). Aprovisionar una instancia RDS puede tardar 10–15 minutos.
3. Verificación en AWS Console: RDS → Bases de datos → ver `db-principal-prod`, su VPC, `db-subnet-group-prod`, `sg-db-prod`, y que Multi-AZ esté "Sí". En Secrets Manager verá `prod/db/main/password`.
4. `terraform destroy` para limpiar.
{% endstep %}
{% endstepper %}

***

## Módulo 8.4: Conexión entre RDS y Aplicaciones EC2

### 1. Introducción

Tenemos una aplicación (ASG de EC2) y una base de datos RDS. La app necesita Hostname (Endpoint), Puerto, Usuario y Contraseña. No podemos hardcodearlos; los inyectamos en EC2 al arranque mediante `user_data` y `templatefile`.

Ejemplo de uso en la aplicación:

```javascript
const dbConfig = {
  host: process.env.DB_HOST,
  port: process.env.DB_PORT,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
};
```

### 2. Objetivo del Laboratorio

Modificar `aws_launch_template` de la aplicación (Módulo 6) para:

1. Obtener Endpoint y Puerto de RDS (outputs).
2. Obtener el Secreto (contraseña) desde Secrets Manager.
3. Pasar estos valores al `user_data` como variables de entorno.

### 3. Paso a Paso

{% stepper %}
{% step %}
### Añadir Permisos de IAM al Rol de EC2 (main.tf)

La instancia EC2 necesita permiso para leer el secreto:

```hcl
data "aws_iam_policy_document" "leer_secreto_db" {
  statement {
    effect = "Allow"
    actions = ["secretsmanager:GetSecretValue"]
    resources = [ aws_secretsmanager_secret.db_secret.arn ]
  }
}

resource "aws_iam_policy" "politica_leer_secreto" {
  name   = "politica-leer-secreto-db-prod"
  policy = data.aws_iam_policy_document.leer_secreto_db.json
}

resource "aws_iam_role_policy_attachment" "adjuntar_secreto_al_rol_ec2" {
  role       = aws_iam_role.rol_ec2_s3_readonly.name
  policy_arn = aws_iam_policy.politica_leer_secreto.arn
}
```

(asumiendo que el rol existe y se llama `rol_ec2_s3_readonly`)
{% endstep %}

{% step %}
### Actualizar el Script user\_data (install\_nginx.sh)

Ejemplo modificado (variables pasadas desde `templatefile`):

```bash
#!/bin/bash
DB_HOST_ENDPOINT=${db_endpoint}
DB_PORT_NUMERO=${db_port}
DB_SECRET_ARN=${db_secret_arn}
AWS_REGION_NAME=${aws_region}

yum update -y
yum install -y nginx unzip curl jq

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
./aws/install

SECRET_JSON=$(aws secretsmanager get-secret-value --secret-id ${DB_SECRET_ARN} --region ${AWS_REGION_NAME} --query SecretString --output text)
DB_USER=$(echo $SECRET_JSON | cut -d'"' -f4)      # Hacky
DB_PASSWORD=$(echo $SECRET_JSON | cut -d'"' -f8)  # Hacky

systemctl start nginx
systemctl enable nginx

cat > /usr/share/nginx/html/index.html <<EOF
<h1>¡Hola, Mundo desde Terraform!</h1>
<h2>Instancia: $(hostname -f)</h2>
<h3>Conectado a la BBDD (simulado):</h3>
<ul>
  <li><b>Endpoint:</b> ${DB_HOST_ENDPOINT}</li>
  <li><b>Puerto:</b> ${DB_PORT_NUMERO}</li>
  <li><b>Usuario:</b> ${DB_USER}</li>
  <li><b>Contraseña:</b> (Oculto, pero empieza con: ${DB_PASSWORD:0:3}...)</li>
</ul>
EOF
```

(En producción, parsee JSON con `jq` de forma segura).
{% endstep %}

{% step %}
### Actualizar la Plantilla de Lanzamiento (main.tf)

Usar `templatefile` para inyectar valores:

```hcl
resource "aws_launch_template" "plantilla_app_web" {
  # ...
  user_data = base64encode(templatefile("install_nginx.sh", {
    db_endpoint   = aws_db_instance.db_principal.endpoint
    db_port       = aws_db_instance.db_principal.port
    db_secret_arn = aws_secretsmanager_secret.db_secret.arn
    aws_region    = var.aws_region
  }))

  iam_instance_profile {
    name = aws_iam_instance_profile.perfil_ec2.name
  }
}
```
{% endstep %}

{% step %}
### Ejecutar y Verificar

1. `terraform apply` — se crearán/actualizarán recursos IAM y Launch Template. ASG realizará un rolling update.
2. Verificación (2–3 minutos): Abrir ALB (`http://app.mi-empresa.com`) y ver la página HTML mostrando Endpoint, Puerto y Usuario. Confirma que EC2 asumió su rol IAM, leyó el secreto y recuperó la contraseña.
{% endstep %}
{% endstepper %}

***

## Módulo 8.5: Lab - Uso de DynamoDB como NoSQL

### 1. Introducción: RDS (SQL) vs. DynamoDB (NoSQL)

* RDS: para datos relacionales estructurados.
* DynamoDB: NoSQL serverless, clave-valor/documento, ideal para datos masivos, telemetría, sesiones, etc.
* Diferencias clave: facturación, escalado, latencia, modelo de datos.

### 2. Objetivo del Laboratorio

Crear una tabla DynamoDB simple para sesiones de usuario.

* Clave de Partición: `session_id` (String)

### 3. Paso 1: Crear la Tabla DynamoDB (main.tf)

```hcl
resource "aws_dynamodb_table" "tabla_sesiones" {
  name         = "app-sesiones-${var.entorno}"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "session_id"

  attribute {
    name = "session_id"
    type = "S"
  }

  tags = {
    Name    = "tabla-sesiones"
    Entorno = var.entorno
  }
}
```

### 4. Paso 2: Exponer Salidas (outputs.tf)

```hcl
output "dynamodb_table_name" {
  description = "El nombre de la tabla DynamoDB"
  value       = aws_dynamodb_table.tabla_sesiones.name
}

output "dynamodb_table_arn" {
  description = "El ARN de la tabla DynamoDB"
  value       = aws_dynamodb_table.tabla_sesiones.arn
}
```

### 5. Paso 3: Ejecutar y Verificar

1. `terraform init` y `terraform apply`.
2. Consola AWS → DynamoDB → Tablas → verá `app-sesiones-dev`.
3. Prueba opcional: Crear elemento con `session_id` y atributos arbitrarios (demuestra schemaless).
4. `terraform destroy`.

***

## Módulo 8.6: Configuración de Backups y Snapshots

### 1. Introducción

Terraform configura políticas de backup; AWS ejecuta los backups.

### 2. Backups de RDS (Automáticos y Manuales)

* Backups automatizados: snapshot diario + logs para PITR. Configurable con `backup_retention_period`.
* Snapshots manuales: `aws_db_snapshot`, conservados incluso si borra la instancia.

#### Lab: Habilitar Backups Automáticos en RDS

Modificar `aws_db_instance`:

```hcl
resource "aws_db_instance" "db_principal" {
  # ...
  backup_retention_period = 7
  backup_window           = "03:00-04:00"
  maintenance_window      = "Sun:05:00-Sun:06:00"
  skip_final_snapshot     = true   # En producción, usar false
}
```

* En producción, `skip_final_snapshot = false` para tomar un snapshot final antes de eliminar.

### 3. Backups de DynamoDB (PITR)

DynamoDB: activar Point-in-Time Recovery:

```hcl
resource "aws_dynamodb_table" "tabla_sesiones" {
  # ...
  point_in_time_recovery {
    enabled = true
  }
  # tags, etc.
}
```

### 4. Ejecutar y Verificar

1. `terraform apply`.
2. RDS Console → instancia → "Mantenimiento y backups" → verá ventana y retención.
3. DynamoDB Console → tabla → "Backups" → verá PITR activada.
4. `terraform destroy`.

***

## Módulo 8.7: Políticas de Escalado en Bases de Datos

### 1. Introducción

Dos tipos de escalado de BBDD:

* Escalado Vertical (scale-up): más CPU/RAM.
* Escalado Horizontal (scale-out): réplicas de lectura.

### 2. Escalado Vertical de RDS

Modificar `instance_class`:

```hcl
instance_class = "db.t3.medium"
```

* `terraform apply` causará downtime a menos que `multi_az = true`, que reduce el impacto (failover inteligente).

### 3. Escalado Horizontal de RDS (Réplicas de Lectura)

Ejemplo:

```hcl
resource "aws_db_instance" "db_replica" {
  identifier         = "db-principal-replica-1"
  replicate_source_db = aws_db_instance.db_principal.id
  instance_class     = "db.t3.medium"
  multi_az           = false
}
```

* Usar el endpoint de la réplica para SELECT y el de la primaria para writes.

### 4. Escalado de DynamoDB

* `billing_mode = "PAY_PER_REQUEST"`: serverless, escalado automático.
* `billing_mode = "PROVISIONED"`: aprovisionar RCU/WCU y usar Application Auto Scaling.

Ejemplo de Auto Scaling (lectura):

```hcl
resource "aws_dynamodb_table" "tabla_provisionada" {
  name         = "tabla-provisionada"
  billing_mode = "PROVISIONED"
  read_capacity  = 5
  write_capacity = 5
  hash_key = "id"
  attribute { name = "id"; type = "S" }
}

resource "aws_appautoscaling_target" "dynamo_read_target" {
  max_capacity        = 100
  min_capacity        = 5
  resource_id         = "table/${aws_dynamodb_table.tabla_provisionada.name}"
  scalable_dimension  = "dynamodb:table:ReadCapacityUnits"
  service_namespace   = "dynamodb"
}

resource "aws_appautoscaling_policy" "dynamo_read_policy" {
  name       = "dynamo-read-scaling-policy"
  resource_id = aws_appautoscaling_target.dynamo_read_target.resource_id
  scalable_dimension = aws_appautoscaling_target.dynamo_read_target.scalable_dimension
  service_namespace  = aws_appautoscaling_target.dynamo_read_target.service_namespace
  policy_type = "TargetTrackingScaling"

  target_tracking_scaling_policy_configuration {
    target_value = 70.0
    predefined_metric_specification {
      predefined_metric_type = "DynamoDBReadCapacityUtilization"
    }
  }
}
```

***

## Módulo 8.8: Lab - Implementación de Aurora Serverless

### 1. Introducción

Aurora Serverless v2 ofrece compatibilidad SQL (PostgreSQL/MySQL) con escalado automático por ACUs; paga por segundo.

### 2. Objetivo del Laboratorio

Crear un clúster Aurora Serverless v2 (PostgreSQL).

### 3. Paso 1: Crear el Clúster Serverless (main.tf)

Asumiendo VPC, `db_subnet_group` y `sg_db` ya creados:

```hcl
resource "aws_rds_cluster" "aurora_serverless" {
  cluster_identifier = "cluster-aurora-serverless-prod"
  engine             = "aurora-postgresql"
  engine_mode        = "provisioned"    # Serverless v2 se configura aquí
  engine_version     = "15.6"
  manage_master_user_password   = true
  master_username               = "admin_db"
  master_user_secret_kms_key_id = aws_secretsmanager_secret.db_secret.kms_key_id

  db_subnet_group_name  = aws_db_subnet_group.db_subnet_group.name
  vpc_security_group_ids = [aws_security_group.sg_db.id]

  backup_retention_period = 7

  serverlessv2_scaling_configuration {
    min_capacity = 0.5
    max_capacity = 2.0
  }

  skip_final_snapshot = true
  tags = { Name = "cluster-aurora-serverless" }

  depends_on = [ aws_secretsmanager_secret_version.db_secret_version ]
}
```

> Nota: Aurora Serverless v2 gestiona la computación automáticamente; no requiere `aws_rds_cluster_instance` manual para este modo.

### 4. Paso 2: Exponer Salidas (outputs.tf)

```hcl
output "aurora_cluster_endpoint" {
  description = "El Endpoint del CLÚSTER (Escritura)"
  value       = aws_rds_cluster.aurora_serverless.endpoint
}

output "aurora_cluster_reader_endpoint" {
  description = "El Endpoint de LECTURA"
  value       = aws_rds_cluster.aurora_serverless.reader_endpoint
}
```

### 5. Paso 3: Ejecutar y Verificar

1. `terraform apply` (puede tardar 15–20 minutos).
2. Consola RDS: verá el clúster y la instancia escritora gestionada; endpoints de escritura y lectura; capacidad Serverless v2.
3. `terraform destroy`.

***

## Módulo 8.9: Monitoreo de Bases de Datos con CloudWatch

### 1. Introducción

RDS y DynamoDB envían métricas a CloudWatch; crear alarmas y notificaciones es esencial.

### 2. Métricas Clave de RDS

* CPUUtilization
* DatabaseConnections
* FreeableMemory
* ReadIOPS / WriteIOPS

#### Lab: Crear una Alarma de RDS para Conexiones Altas

Crear SNS topic y alarma:

```hcl
resource "aws_sns_topic" "alertas_db" {
  name = "alertas-db-produccion"
}

resource "aws_cloudwatch_metric_alarm" "alarma_conexiones_db" {
  alarm_name          = "alarma-conexiones-altas-db"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "DatabaseConnections"
  namespace           = "AWS/RDS"
  period              = "300"
  statistic           = "Average"
  threshold           = "100"
  dimensions = { DBInstanceIdentifier = aws_db_instance.db_principal.id }
  alarm_description = "Alarma cuando las conexiones a la BBDD son demasiado altas"
  alarm_actions      = [aws_sns_topic.alertas_db.arn]
  ok_actions         = [aws_sns_topic.alertas_db.arn]
}
```

> Para recibir emails, añadir `aws_sns_topic_subscription` con `protocol = "email"` y confirmar la suscripción.

### 3. Métricas Clave de DynamoDB

* ThrottledRequests
* ConsumedReadCapacityUnits / ConsumedWriteCapacityUnits
* UserErrors / SystemErrors

#### Lab: Crear una Alarma de DynamoDB para "Throttling"

```hcl
resource "aws_cloudwatch_metric_alarm" "alarma_throttling_dynamo" {
  alarm_name          = "alarma-dynamodb-throttling"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "1"
  metric_name         = "ThrottledRequests"
  namespace           = "AWS/DynamoDB"
  period              = "60"
  statistic           = "Sum"
  threshold           = "0"
  dimensions = { TableName = aws_dynamodb_table.tabla_sesiones.name }
  alarm_description = "¡ALERTA! DynamoDB está estrangulando peticiones."
  alarm_actions      = [aws_sns_topic.alertas_db.arn]
  ok_actions         = [aws_sns_topic.alertas_db.arn]
}
```

***

## Módulo 8.10: Buenas Prácticas de Gestión de Datos en la Nube

### 1. Resumen

Buenas prácticas clave para bases de datos en la nube.

### 2. Seguridad: El Pilar Cero

1. NUNCA use contraseñas hardcodeadas.
2. Use AWS Secrets Manager para almacenar y rotar credenciales.
3. Evite almacenar secretos en `terraform.tfstate`; si lo hace, cifre el estado en S3.
4. Bases de datos nunca deben estar en subred pública (`publicly_accessible = false`).
5. Firewalls estrictos: patrón SG-App -> SG-DB.
6. Cifrado en reposo: `storage_encrypted = true` en RDS (DynamoDB cifrado por defecto).
7. Considere autenticación IAM para RDS (`iam_database_authentication_enabled = true`) como opción avanzada.

### 3. Fiabilidad (Reliability)

1. RDS: usar `multi_az = true` en producción.
2. RDS: habilitar backups y PITR (`backup_retention_period`).
3. DynamoDB: habilitar PITR (`point_in_time_recovery { enabled = true }`).
4. Protección contra eliminación: en producción, `deletion_protection = true`.

### 4. Optimización de Costes

1. Elegir el servicio adecuado (RDS vs DynamoDB) según el caso de uso.
2. DynamoDB: empezar con `PAY_PER_REQUEST`. Cambiar a `PROVISIONED` + Auto Scaling si conviene.
3. RDS: usar Aurora Serverless para entornos intermitentes o desarrollo.
4. Apagar recursos de desarrollo fuera del horario laboral.

### 5. Rendimiento

1. RDS: usar Réplicas de Lectura para descargar SELECTs.
2. DynamoDB: usar DAX para caché en lecturas masivas.
3. Monitorear con CloudWatch y crear alarmas para métricas clave (FreeableMemory, ThrottledRequests, etc.).

***
