# Módulo 15 - PROYECTO FINAL\_ INFRAESTRUCTURA AUTOMATIZADA EN AWS

## Módulo 15.1: Definición del Alcance del Proyecto

### 1. Introducción: El Proyecto "Capstone"

¡Bienvenido al Módulo Final! En este proyecto, dejará de lado los laboratorios aislados y construirá una pila de infraestructura completa, de extremo a extremo (end-to-end). Aplicará los conceptos de redes, cómputo, bases de datos, almacenamiento, seguridad, estado remoto, modularidad y CI/CD en un solo proyecto cohesivo.

### 2. El Objetivo: Una Arquitectura Web de 3 Capas

El objetivo es desplegar una **arquitectura web de 3 capas, tolerante a fallos, escalable y segura** en AWS. Esta es la arquitectura estándar de la industria para la mayoría de las aplicaciones web.

### 3. Componentes de la Arquitectura

Nuestra infraestructura final incluirá:

{% stepper %}
{% step %}
### Red (Módulo 5)

* 1 VPC desplegada en 2 Zonas de Disponibilidad (Multi-AZ).
* 6 Subredes: 2 públicas (para ALB), 2 privadas de aplicación (para EC2) y 2 privadas de datos (para RDS).
* 1 Internet Gateway (IGW) y 1 NAT Gateway (para permitir la salida de las instancias privadas).
* Tablas de rutas (Route Tables) para enrutar el tráfico.
{% endstep %}

{% step %}
### Seguridad (Módulo 5 y 13)

* 3 Security Groups: sg-alb (para el ALB), sg-app (para las instancias EC2) y sg-db (para la BBDD), usando el patrón de referencia de SGs.
{% endstep %}

{% step %}
### Cómputo (Módulo 6)

* 1 Application Load Balancer (ALB) en las subredes públicas.
* 1 Auto Scaling Group (ASG) que lanza instancias t3.micro en las subredes privadas.
* 1 Launch Template (Plantilla de Lanzamiento) que usa user\_data para instalar un servidor web (ej. Nginx).
{% endstep %}

{% step %}
### Base de Datos (Módulo 8)

* 1 Instancia de Amazon RDS (PostgreSQL o MySQL) en modo **Multi-AZ** para alta disponibilidad.
* 1 db\_subnet\_group que la ubica en las subredes de datos.
* 1 Secreto en **AWS Secrets Manager** para la contraseña de la BBDD.
{% endstep %}

{% step %}
### Almacenamiento (Módulo 7)

* 1 Bucket S3 (s3-assets) para almacenar activos estáticos (ej. imágenes), configurado como privado y seguro.
{% endstep %}

{% step %}
### Monitoreo (Módulo 9)

* 1 Alarma de CloudWatch para la CPU del ASG.
* 1 Alarma de CloudWatch para la memoria (FreeableMemory) del RDS.
* 1 Dashboard de CloudWatch que muestre el estado de la aplicación.
* 1 Presupuesto (aws\_budgets\_budget) para monitorear el coste total del proyecto.
{% endstep %}
{% endstepper %}

### 4. Requisitos de Automatización y Modularidad

Esta infraestructura **NO** se construirá en un solo archivo main.tf monolítico.

* **Modularidad (Módulo 10):** El proyecto debe seguir un **patrón de composición de módulos**. El Módulo Raíz (root) debe estar limpio y llamar a módulos hijos (locales o del Registry) para vpc, alb, asg y rds.
* **Estado Remoto (Módulo 11):** El proyecto **DEBE** configurarse con un backend S3 y bloqueo de DynamoDB desde el principio. No se permite el estado local.
* **CI/CD (Módulo 12):** El proyecto **DEBE** ser desplegado a través de un pipeline de **GitHub Actions**. El pipeline incluirá validación, tfsec, plan en el PR y apply en la fusión (merge).

***

## Módulo 15.2: Paso 1 - Configuración del Backend y Módulos de Red

### 1. Tarea: El "Día 0" - Backend y Red

El primer paso es establecer la base.

{% stepper %}
{% step %}
### Bootstrap del Backend (Módulo 11)

* Antes de empezar, asegúrese de tener su bucket S3 (terraform-state) y su tabla DynamoDB (terraform-locks) creados (usando su proyecto terraform-backend-setup).
{% endstep %}

{% step %}
### Estructura del Repositorio (Módulo 10)

* Cree su nuevo repositorio de Git para el proyecto final.
* Configure su backend.tf en la raíz para que apunte a su bucket de estado (ej. key = "proyectofinal/prod/terraform.tfstate").
* Configure su providers.tf (con aws y random).
{% endstep %}

{% step %}
### Módulo de Red (Módulo 5 y 10)

* **NO** escriba la VPC a mano. Use el módulo terraform-aws-modules/vpc/aws del Registry (como en el Lab 10.3).
* Su main.tf (raíz) comenzará con esto:

```hcl
# main.tf
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.8.1" # (O la última versión estable)
  name = "vpc-proyecto-final"
  cidr = "10.10.0.0/16"
  azs = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.10.1.0/24", "10.10.2.0/24"]
  public_subnets  = ["10.10.101.0/24", "10.10.102.0/24"]
  # Subredes separadas para la BBDD
  database_subnets = ["10.10.201.0/24", "10.10.202.0/24"]
  # Crear el IGW
  enable_internet_gateway = true
  # Crear el NGW (en 1 AZ para ahorrar costes)
  enable_nat_gateway       = true
  single_nat_gateway       = true

  tags = {
    Entorno  = "prod"
    Proyecto = "capstone"
  }
}
```
{% endstep %}
{% endstepper %}

***

## Módulo 15.3: Paso 2 - Módulos de Cómputo (ALB y ASG)

### 1. Tarea: Desplegar la Capa de Aplicación (Módulo 6)

Ahora, cree un **módulo local** (ej. ./modules/compute) o añada el código al raíz (menos ideal) para desplegar el ALB y el ASG.

```hcl
# main.tf (continuación)

# --- Seguridad (Módulo 5) ---
# (Debe crear los SGs 'alb' y 'app' primero,
# pasando el module.vpc.vpc_id)
resource "aws_security_group" "sg_alb" {
  name   = "sg-alb-prod"
  vpc_id = module.vpc.vpc_id
  # ... (Reglas: 80/443 desde 0.0.0.0/0) ...
}

resource "aws_security_group" "sg_app" {
  name   = "sg-app-prod"
  vpc_id = module.vpc.vpc_id
  # ... (Reglas: 8080 desde sg_alb, 22 desde IP de admin) ...
}

# --- ALB (Módulo 6.6) ---
resource "aws_lb" "alb" {
  name               = "alb-prod"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.sg_alb.id]
  # Usar las subredes PÚBLICAS de la VPC
  subnets = module.vpc.public_subnets
}

resource "aws_lb_target_group" "app_tg" {
  name    = "tg-app-prod"
  port    = 8080
  protocol = "HTTP"
  vpc_id  = module.vpc.vpc_id

  health_check {
    path = "/"
  }
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.alb.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app_tg.arn
  }
}

# --- ASG (Módulo 6.5) ---
data "aws_ami" "amazon_linux" {
  # ... (Buscar AMI)
}

resource "aws_launch_template" "app_lt" {
  name_prefix = "app-lt-"
  image_id    = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"

  # Usar el script de user_data (nginx en puerto 8080)
  user_data = filebase64("user_data.sh")

  # Usar el SG de la app
  vpc_security_group_ids = [aws_security_group.sg_app.id]

  # (Configuración de clave SSH, rol de IAM)
}

resource "aws_autoscaling_group" "app_asg" {
  name_prefix       = "asg-app-"
  min_size          = 2
  max_size          = 5
  desired_capacity  = 2

  # Usar las subredes PRIVADAS de la VPC
  vpc_zone_identifier = module.vpc.private_subnets

  # Conectar al Launch Template
  launch_template {
    id      = aws_launch_template.app_lt.id
    version = "$Latest"
  }

  # ¡Conectar al ALB!
  target_group_arns = [aws_lb_target_group.app_tg.arn]
}
```

***

## Módulo 15.4 y 15.5: Pasos 3 y 4 - Módulos de Almacenamiento (S3 y RDS)

### 1. Tarea: Desplegar la Capa de Datos (Módulo 7 y 8)

#### S3 (Almacenamiento de Activos)

Use el módulo local s3-seguro que creamos en el Módulo 10.2.

```hcl
# main.tf (continuación)
module "s3_assets" {
  source = "./modules/s3-seguro" # (Suponiendo que lo copió)
  nombre_base_bucket = "mi-proyecto-final-assets"
  etiquetas = {
    Entorno  = "prod"
    Proyecto = "capstone"
  }
}
```

#### RDS (Base de Datos)

Despliegue la instancia RDS, aplicando _todas_ las buenas prácticas del Módulo 8.

```hcl
# main.tf (continuación)

# --- Seguridad para la BBDD ---
resource "aws_security_group" "sg_db" {
  name   = "sg-db-prod"
  vpc_id = module.vpc.vpc_id

  # Regla de ENTRADA: PostgreSQL (5432)
  # Solo desde el SG de la APLICACIÓN
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.sg_app.id]
  }

  # ... (regla de salida) ...
}

# --- Grupo de Subredes de BBDD ---
resource "aws_db_subnet_group" "db_subnet_group" {
  name       = "db-subnet-group-prod"
  # Usar las subredes de DATOS de la VPC
  subnet_ids = module.vpc.database_subnets
}

# --- Secreto de la BBDD ---
resource "random_password" "db_password" {
  # ...
}

resource "aws_secretsmanager_secret" "db_secret" {
  # ...
}

resource "aws_secretsmanager_secret_version" "db_secret_version" {
  # ...
}

# (Código del Lab 8.2)

# --- Instancia RDS ---
resource "aws_db_instance" "db" {
  identifier              = "db-prod"
  engine                  = "postgres"
  instance_class          = "db.t3.micro"
  allocated_storage       = 20

  # Conectividad
  vpc_security_group_ids  = [aws_security_group.sg_db.id]
  db_subnet_group_name    = aws_db_subnet_group.db_subnet_group.name
  publicly_accessible     = false

  # Alta Disponibilidad (HA)
  multi_az = true

  # Gestión de Secretos
  manage_master_user_password = true
  master_username             = "admin_db"
  master_user_secret_kms_key_id = aws_secretsmanager_secret.db_secret.kms_key_id

  # Backups
  backup_retention_period = 7
  skip_final_snapshot     = false # ¡Producción!
  deletion_protection     = true  # ¡Producción!

  tags = {
    Entorno  = "prod"
    Proyecto = "capstone"
  }

  depends_on = [
    aws_secretsmanager_secret_version.db_secret_version
  ]
}
```

***

## Módulo 15.6: Paso 5 - Monitoreo como Código (Módulo 9)

### 1. Tarea: Añadir Alarmas y Presupuesto

No basta con crear la infraestructura; debemos monitorearla.

```hcl
# main.tf (continuación)

# --- Tópico SNS para Alertas ---
resource "aws_sns_topic" "alertas" {
  name = "alertas-produccion"
}

resource "aws_sns_topic_subscription" "email_admin" {
  topic_arn = aws_sns_topic.alertas.arn
  protocol  = "email"
  endpoint  = "admin@su-empresa.com" # ¡Confirmar manualmente!
}

# --- Alarma de CPU del ASG ---
resource "aws_cloudwatch_metric_alarm" "asg_cpu_alta" {
  alarm_name          = "alarma-cpu-alta-app-prod"
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  statistic           = "Average"
  period              = 300
  evaluation_periods  = 2
  comparison_operator = "GreaterThanThreshold"
  threshold           = 80

  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.app_asg.name
  }

  alarm_actions = [aws_sns_topic.alertas.arn]
}

# --- Alarma de Memoria de RDS ---
resource "aws_cloudwatch_metric_alarm" "rds_memoria_baja" {
  alarm_name          = "alarma-memoria-baja-db-prod"
  metric_name         = "FreeableMemory"
  namespace           = "AWS/RDS"
  statistic           = "Average"
  period              = 300
  evaluation_periods  = 2
  comparison_operator = "LessThanThreshold"
  threshold           = 100000000 # 100MB (en bytes)

  dimensions = {
    DBInstanceIdentifier = aws_db_instance.db.id
  }

  alarm_actions = [aws_sns_topic.alertas.arn]
}

# --- Presupuesto (Budget) ---
resource "aws_budgets_budget" "presupuesto_proyecto" {
  name        = "proyecto-final-capstone"
  budget_type = "COST"
  limit_amount = "100.0" # $100 limit
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  notification {
    notification_type = "FORECASTED" # Alerta sobre el coste *proyectado*
    comparison_operator = "GREATER_THAN"
    threshold = 100
    threshold_type = "PERCENTAGE"
    subscriber_sns_topic_arns = [aws_sns_topic.alertas.arn]
  }
}
```

***

## Módulo 15.7 y 15.8: Pasos 6 y 7 - Modularización y Estado

### 1. Tarea: Refactorizar y Asegurar

Este es el momento de aplicar la ingeniería de software.

{% stepper %}
{% step %}
### Modularización (Módulo 10)

* El main.tf que hemos construido es grande. Refactorícelo.
* Cree una estructura de carpetas modules/ (como en el Lab 10.9).
* Cree un módulo alb (con 3 recursos).
* Cree un módulo asg (con 2 recursos).
* Cree un módulo rds (con 5-6 recursos).
* Su main.tf raíz debe quedar "limpio", conteniendo solo los bloques module que se llaman entre sí, pasando outputs a variables.
{% endstep %}

{% step %}
### Estado Remoto (Módulo 11)

* Asegúrese de que su archivo backend.tf (del Paso 1) esté presente y sea correcto.
* Ejecute terraform init y migre su estado local (si existe) al backend S3.
* Asegúrese de que su tabla DynamoDB (dynamodb\_table = "...") esté en la configuración del backend.
{% endstep %}
{% endstepper %}

***

## Módulo 15.9: Paso 8 - Pipeline de CI/CD (Módulo 12)

### 1. Tarea: Automatizar el Despliegue

La infraestructura solo debe desplegarse a través de Git.

* **Seguridad OIDC:** Configure el Rol de IAM OIDC (Lab 12.3) para que confíe en su repositorio de GitHub.
* **Secretos de GitHub:** Guarde el ARN del rol como un secreto (AWS\_ROLE\_TO\_ASSUME).
* **Crear el Workflow:** Cree el archivo .github/workflows/terraform.yml.

#### .github/workflows/terraform.yml (Completo)

```yaml
name: 'Proyecto Final Terraform CI/CD'
on:
  pull_request:
    branches: [ main ]
    paths: [ '**/*.tf' ]
  push:
    branches: [ main ]
    paths: [ '**/*.tf' ]

permissions:
  id-token: write
  contents: read
  pull-requests: write

jobs:
  # ----------------------------------------
  # VALIDAR (Se ejecuta en cada PR)
  # ----------------------------------------
  validate:
    name: '1. Validar y Seguridad'
    runs-on: ubuntu-latest
    steps:
      - name: 'Checkout'
        uses: actions/checkout@v4

      - name: 'Setup Terraform'
        uses: hashicorp/setup-terraform@v3

      - name: 'Terraform Init (validate)'
        run: terraform init -backend=false

      - name: 'Terraform Fmt Check'
        run: terraform fmt --check --recursive

      - name: 'Terraform Validate'
        run: terraform validate

      - name: 'Seguridad (tfsec)'
        uses: aquasecurity/tfsec-action@v1.0.0
        with:
          soft_fail: false # Falla el pipeline si hay errores

  # ----------------------------------------
  # PLAN (Se ejecuta en cada PR, después de Validar)
  # ----------------------------------------
  plan:
    name: '2. Terraform Plan'
    runs-on: ubuntu-latest
    needs: [validate]
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

      - name: 'Terraform Init (Plan)'
        run: terraform init

      - name: 'Terraform Plan'
        run: terraform plan -out=plan.tfplan

      - name: 'Upload Plan Artifact'
        uses: actions/upload-artifact@v4
        with:
          name: plan
          path: plan.tfplan
      # (Paso opcional para publicar el plan en el PR)

  # ----------------------------------------
  # APPLY (Se ejecuta en 'push to main', después de Plan)
  # ----------------------------------------
  apply:
    name: '3. Terraform Apply (a Prod)'
    runs-on: ubuntu-latest
    needs: [plan]
    # ¡Solo se ejecuta al fusionar (merge) a 'main'!
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

      - name: 'Terraform Init (Apply)'
        run: terraform init

      - name: 'Terraform Apply'
        run: terraform apply "plan.tfplan"
```

***

## Módulo 15.10: Paso 9 - Presentación y Documentación

### 1. Tarea: El Entregable Final

El código por sí solo no es suficiente. Un proyecto profesional debe estar documentado.

#### 1. Documentación (README.md)

Su repositorio **DEBE** incluir un README.md de alta calidad que incluya:

* **Propósito:** ¿Qué construye este proyecto?
* **Diagrama de Arquitectura:** Una imagen (como la del Módulo 15.1) que muestre la arquitectura.
* **Prerrequisitos:**
  * Cómo configurar el Backend (S3/DynamoDB).
  * Cómo configurar el Rol OIDC de GitHub.
* **Cómo Usarlo (Flujo de Trabajo):**
  * "Haga un PR para planificar".
  * "Fusione a main para desplegar".
* **Estructura de Módulos:** (Opcional) Una breve descripción de los módulos (vpc, rds, etc.).
* **Entradas (Inputs) y Salidas (Outputs):** (Generado con terraform-docs si es posible) Las variables principales y las salidas (como la URL del ALB).

#### 2. Presentación (Demo)

Prepare una presentación de 10-15 minutos para la clase:

* Muestre su repositorio de GitHub (el README.md).
* Muestre su main.tf modularizado.
* Haga un cambio (ej. cambiar max\_size del ASG de 5 a 6).
* Abra un PR.
* Muestre al pipeline ejecutando validate, tfsec y plan.
* Muestre el plan en el PR.
* Fusione el PR.
* Muestre el pipeline apply ejecutándose.
* Muestre el cambio en la Consola de AWS (el ASG ahora tiene max=6).
* ¡Ejecute terraform destroy (si procede)!

***
