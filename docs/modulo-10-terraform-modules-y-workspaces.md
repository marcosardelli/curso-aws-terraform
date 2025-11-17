# Módulo 10 - TERRAFORM MODULES Y WORKSPACES

## Módulo 10.1: ¿Qué son los Módulos en Terraform?

### 1. Introducción: El Problema de "Copiar y Pegar"

Hasta ahora, hemos escrito todo nuestro código en un único conjunto de archivos (main.tf, variables.tf, etc.). Este conjunto de archivos es, en sí mismo, un **Módulo Raíz (Root Module)**.

Pero, ¿qué pasa cuando necesitamos desplegar una _segunda_ VPC, idéntica a la primera, para el entorno de staging? ¿O una tercera para producción?

El anti-patrón sería copiar y pegar las 200 líneas de código de nuestra VPC (Módulo 5). Esto viola el principio fundamental de la ingeniería de software: **DRY (Don't Repeat Yourself - No te Repitas)**.

### 2. Definición: Módulos como Funciones

Un **Módulo** en Terraform es un contenedor de uno o más recursos (resource) que se gestionan como un grupo.

La mejor analogía es pensar en los módulos como **funciones** en un lenguaje de programación:

| **Concepto de Programación** | **Concepto de Terraform**    | **Ejemplo**                        |
| ---------------------------- | ---------------------------- | ---------------------------------- |
| def mi\_funcion(a, b):       | module "mi\_modulo" { ... }  | module "vpc\_produccion" { ... }   |
| **Parámetros de Entrada**    | **Variables** (variables.tf) | vpc\_cidr = "10.0.0.0/16"          |
| **Cuerpo de la Función**     | **Recursos** (main.tf)       | resource "aws\_vpc" "this" { ... } |
| **Valor de Retorno**         | **Salidas** (outputs.tf)     | output "vpc\_id" { ... }           |

Un módulo es una "caja negra" que toma entradas (variables), crea un conjunto de recursos y devuelve salidas (outputs).

### 3. Módulo Raíz vs. Módulo Hijo

* **Módulo Raíz (Root Module):** Es el directorio donde usted ejecuta terraform apply. Todo proyecto tiene uno.
* **Módulo Hijo (Child Module):** Es un módulo que es _llamado_ desde otro módulo (normalmente el raíz) usando un bloque module.

Ejemplo (en el Módulo Raíz llamando a un Módulo Hijo):

```hcl
module "mi_vpc_personalizada" {
  source = "./modulos/vpc"        # Un directorio local
  cidr_block = "10.1.0.0/16"
  nombre_vpc = "vpc-dev"
}

resource "aws_instance" "test" {
  ami = "..."
  subnet_id = module.mi_vpc_personalizada.id_subred_publica
}
```

### 4. Beneficios Clave

1. **Reusabilidad (DRY):** Defina su "VPC Segura" una vez y llámela 10 veces con diferentes variables.
2. **Abstracción:** Un desarrollador de aplicaciones no necesita saber _cómo_ se construye una VPC de 6 subredes con NAT Gateways. Solo necesita llamar a module "vpc" { ... } y obtener los IDs de las subredes como salida.
3. **Organización:** Mantiene su Módulo Raíz limpio y legible. En lugar de 500 líneas de recursos, tiene 5 bloques module (vpc, rds, alb, etc.).
4. **Mantenimiento:** Si encuentra un bug o una mejora de seguridad, la arregla _una vez_ en el módulo. Todos los entornos que usan ese módulo se benefician en el próximo apply.

### 5. Material de Apoyo y Siguientes Pasos

#### Documentos Clave

* [**Introducción a Módulos (Terraform Docs)**](https://developer.hashicorp.com/terraform/language/modules) — La documentación conceptual oficial.

***

## Módulo 10.2: Lab - Creación de Módulos Reutilizables (Locales)

### 1. Introducción

En este laboratorio practicaremos la **abstracción**. Tomaremos un conjunto de recursos que hemos escrito antes y los convertiremos en nuestro primer Módulo Hijo local.

### 2. Objetivo del Laboratorio

Crear un módulo s3-seguro que encapsule la lógica de crear un bucket S3 que sea _siempre_ privado, versionado y cifrado.

* El Módulo (./modules/s3-seguro/) contendrá:
  * aws\_s3\_bucket
  * aws\_s3\_bucket\_versioning
  * aws\_s3\_bucket\_public\_access\_block
  * aws\_s3\_bucket\_server\_side\_encryption\_configuration
* El Módulo Raíz (.) llamará a este módulo.

### 3. Estructura de Carpetas

Cree esta estructura de carpetas y archivos:

terraform-modulo-s3/ ├── main.tf # Módulo Raíz: Llama al módulo ├── outputs.tf # Módulo Raíz: Muestra las salidas del módulo ├── providers.tf # Módulo Raíz: Configura AWS └── modules/ └── s3-seguro/ ├── main.tf # Módulo Hijo: Los 4 recursos de S3 ├── variables.tf # Módulo Hijo: Entradas (ej. bucket\_name) └── outputs.tf # Módulo Hijo: Salidas (ej. bucket\_arn)

### 4. Paso 1: Escribir el Módulo Hijo (La "Función")

Estos son los archivos dentro de modules/s3-seguro/.

modules/s3-seguro/variables.tf

```hcl
variable "nombre_base_bucket" {
  description = "El nombre base para el bucket S3 (se le añadirán sufijos)"
  type        = string
}

variable "etiquetas" {
  description = "Un mapa de etiquetas para aplicar al bucket"
  type        = map(string)
  default     = {}
}
```

modules/s3-seguro/main.tf

```hcl
# ¡Los providers se heredan del Módulo Raíz!

resource "aws_s3_bucket" "this" {
  bucket = "${var.nombre_base_bucket}-${random_string.sufijo.result}"
  tags = merge(
    var.etiquetas,
    { "Name" = var.nombre_base_bucket }
  )
}

resource "random_string" "sufijo" {
  length  = 6
  special = false
  upper   = false
}

resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_public_access_block" "this" {
  bucket                      = aws_s3_bucket.this.id
  block_public_acls           = true
  block_public_policy         = true
  ignore_public_acls          = true
  restrict_public_buckets     = true
}

resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
```

modules/s3-seguro/outputs.tf

```hcl
output "bucket_id" {
  description = "El ID (nombre único) del bucket creado"
  value       = aws_s3_bucket.this.id
}

output "bucket_arn" {
  description = "El ARN del bucket creado"
  value       = aws_s3_bucket.this.arn
}
```

¡Felicidades! Acaba de escribir un módulo reutilizable.

### 5. Paso 2: Escribir el Módulo Raíz (La "Llamada a la Función")

Estos son los archivos en el directorio principal (terraform-modulo-s3/).

providers.tf (Raíz)

```hcl
terraform {
  required_version = ">= 1.9.0"
  required_providers {
    aws = { source = "hashicorp/aws" version = "~> 5.0" }
    random = { source = "hashicorp/random" version = "~> 3.0" } # El módulo hijo usa random_string
  }
}

provider "aws" {
  region = "us-east-1"
}
```

main.tf (Raíz)

```hcl
module "bucket_logs_dev" {
  source = "./modules/s3-seguro"
  nombre_base_bucket = "app-logs-dev"
  etiquetas = {
    Entorno   = "dev"
    Propietario = "equipo-a"
  }
}

module "bucket_backups_prod" {
  source = "./modules/s3-seguro"
  nombre_base_bucket = "db-backups-prod"
  etiquetas = {
    Entorno   = "prod"
    Criticidad = "alta"
  }
}
```

outputs.tf (Raíz)

```hcl
output "arn_bucket_logs_dev" {
  description = "El ARN del bucket de logs de dev"
  value       = module.bucket_logs_dev.bucket_arn
}

output "id_bucket_backups_prod" {
  description = "El ID del bucket de backups de prod"
  value       = module.bucket_backups_prod.bucket_id
}
```

### 6. Paso 3: Ejecutar y Verificar

{% stepper %}
{% step %}
### Inicializar

1. Vaya a la carpeta raíz `terraform-modulo-s3/`.
2. `terraform init`: Terraform detectará y cargará los módulos (Initializing modules...).
{% endstep %}

{% step %}
### Planificar

`terraform plan`: Verá que Terraform planea crear los 2 `random_string` y los 8 recursos de S3 (2 buckets + 2x3 configs).
{% endstep %}

{% step %}
### Aplicar

`terraform apply` → escriba `yes`.
{% endstep %}

{% step %}
### Verificación

* Vaya a la consola de S3. Verá sus dos buckets (`app-logs-dev-xxx`, `db-backups-prod-xxx`).
* En cada bucket, en "Propiedades" verá que Versionado, Cifrado y Bloqueo Público están todos activados.
{% endstep %}

{% step %}
### Limpiar

`terraform destroy`.
{% endstep %}
{% endstepper %}

***

## Módulo 10.3: Lab - Uso de Módulos Oficiales del Registry

### 1. Introducción

Para componentes comunes (como VPCs) es preferible usar módulos del Terraform Registry en lugar de reinventar 200 líneas de HCL.

### 2. El Terraform Registry

* Sitio Web: https://registry.terraform.io/
* Es el repositorio público de Providers y Módulos.
* Módulos oficiales populares: `terraform-aws-modules/vpc/aws`.

### 3. Objetivo del Laboratorio

Reconstruir nuestra VPC del Módulo 5 usando el módulo oficial `terraform-aws-modules/vpc/aws` con menos de 20 líneas de código.

### 4. Estructura de Archivos

Una carpeta `terraform-modulo-vpc/` con:

* providers.tf
* main.tf (llamaremos al módulo)
* outputs.tf

### 5. Paso 1: Configurar Providers (providers.tf)

```hcl
terraform {
  required_version = ">= 1.9.0"
  required_providers {
    aws = { source = "hashicorp/aws" version = "~> 5.0" }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

### 6. Paso 2: Llamar al Módulo VPC (main.tf)

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.8.1"                 # ¡SIEMPRE FIJAR LA VERSIÓN!
  name    = "vpc-principal-modulo"
  cidr    = "10.0.0.0/16"
  azs     = ["us-east-1a", "us-east-1b"]

  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_internet_gateway = true
  enable_nat_gateway      = true
  single_nat_gateway      = true

  tags = {
    Proyecto = "curso-terraform"
    Fuente   = "modulo-registry"
  }
}
```

### 7. Paso 3: Usar las Salidas (outputs.tf)

```hcl
output "vpc_id" {
  description = "El ID de la VPC creada por el módulo"
  value       = module.vpc.vpc_id
}

output "ids_subredes_publicas" {
  description = "Lista de IDs de subredes públicas"
  value       = module.vpc.public_subnets
}

output "ids_subredes_privadas" {
  description = "Lista de IDs de subredes privadas"
  value       = module.vpc.private_subnets
}
```

### 8. Paso 4: Ejecutar y Verificar

{% stepper %}
{% step %}
`terraform init`: descargará el módulo en `.terraform/modules/`.
{% endstep %}

{% step %}
`terraform plan`: verá un plan grande (\~20 recursos).
{% endstep %}

{% step %}
`terraform apply` → escriba `yes`.
{% endstep %}

{% step %}
Verifique en la consola de VPC: `vpc-principal-modulo`, subredes, tablas de rutas y NAT Gateways.
{% endstep %}

{% step %}
`terraform destroy`.
{% endstep %}
{% endstepper %}

***

## Módulo 10.4: Estructura de Carpetas para Proyectos Grandes

### 1. Introducción

A medida que la infraestructura crece, un solo Módulo Raíz se vuelve inmanejable. La solución es dividir en múltiples componentes independientes (o "stacks"), cada uno con su propio archivo de estado.

### 2. Anti-Patrón: El Monolito

infra-monolito/

* main.tf (5000 líneas: VPC, RDS, EKS, ALBs...)
* variables.tf
* outputs.tf
* terraform.tfstate

Problemas: planificación lenta, un solo estado compartido, gran blast radius.

### 3. Patrón 1: Módulos Locales (Bueno para Proyectos Medianos)

infra-modular/

* main.tf (llama a módulos)
* modules/
  * vpc/
  * rds/
  * app/
* terraform.tfstate (único estado)

Aclara el código pero mantiene un solo estado.

### 4. Patrón 2: Estructura de Directorios por Componente (Mejor)

infra-componentes/

* 00-networking/
  * main.tf (Crea la VPC)
  * terraform.tfstate (estado solo para la RED)
* 01-data/
  * main.tf (Crea RDS, usa `data` para leer la VPC)
  * terraform.tfstate
* 02-app/
  * main.tf (ASG, ALB, usa `data`)
  * terraform.tfstate

Flujo de trabajo: equipos independientes, aislamiento total. Use `terraform_remote_state` para compartir salidas entre stacks (ver Módulo 11).

### 5. Patrón 3: Estructura de Directorios por Entorno (Recomendado)

infra-empresa/

* modules/
  * vpc/
  * rds/
  * app/
* live/
  * dev/
    * 00-networking/ (llama a modules/vpc)
    * 01-app/
    * backend.tf (estado en S3: key="dev/net.tfstate")
  * test/
    * ...
  * prod/
    * ...

Ventajas: aislamiento por componente y por entorno. Herramientas como Terragrunt ayudan a reducir boilerplate.

***

## Módulo 10.5: Concepto de Workspaces en Terraform

### 1. Introducción

Además de separar entornos por directorios, Terraform tiene **Workspaces** (CLI) que permiten usar el mismo código con múltiples archivos de estado.

### 2. ¡Confusión! CLI Workspaces vs. TFC Workspaces

* **Workspaces de Terraform Cloud (TFC):** unidad de ejecución en la nube (estado, variables, repo, políticas).
* **Workspaces de CLI (Open Source):** feature del CLI que permite múltiples archivos de estado dentro del mismo directorio. Por defecto estás en `default`.

### 3. El Flujo de Trabajo de CLI Workspaces

Ejemplo:

1. `terraform init` crea `terraform.tfstate` en el workspace `default`.
2. `terraform workspace new dev` crea `terraform.tfstate.d/dev/terraform.tfstate`.
3. `terraform workspace new prod` crea `terraform.tfstate.d/prod/terraform.tfstate`.
4. `terraform workspace select dev` hace que las operaciones usen el estado de `dev`.

Resumen: los CLI Workspaces son un separador de archivos de estado.

### 4. El Problema: ¿Cómo Diferenciar Entornos?

Use la variable mágica `terraform.workspace` para adaptar el comportamiento del código según el workspace.

Ejemplo:

```hcl
locals {
  configuracion_entorno = {
    "dev" = { instance_type = "t2.micro" instance_count = 1 }
    "prod" = { instance_type = "m5.large" instance_count = 10 }
  }
  config_actual = local.configuracion_entorno[terraform.workspace]
}

resource "aws_instance" "web" {
  count         = local.config_actual.instance_count
  instance_type = local.config_actual.instance_type
  ami           = "..."
  tags = { Entorno = terraform.workspace }
}
```

### 5. Material de Apoyo

* [Documentación de CLI Workspaces (Terraform Docs)](https://developer.hashicorp.com/terraform/language/state/workspaces)

***

## Módulo 10.6: Lab - Separación de Entornos (Workspaces vs. Directorios)

### 1. Introducción

Compararemos dos métodos para gestionar entornos (dev y prod): (1) estructura de directorios y (2) CLI Workspaces.

### 2. Objetivo del Laboratorio

Desplegar dos instancias EC2 (una para dev y otra para prod) con diferentes tipos de instancia, usando ambos métodos.

### 3. Método 1: Estructura de Directorios (Aislamiento Fuerte)

Estructura:

entornos-por-directorio/

* modules/
  * ec2-instancia/
    * main.tf
    * variables.tf
    * outputs.tf
* dev/
  * main.tf
  * terraform.tfvars
  * backend.tf
* prod/
  * main.tf
  * terraform.tfvars
  * backend.tf

Módulo `modules/ec2-instancia/`:

variables.tf

```hcl
variable "instance_type" { type = string }
variable "nombre" { type = string }
variable "ami" { type = string }
```

main.tf

```hcl
resource "aws_instance" "this" {
  ami           = var.ami
  instance_type = var.instance_type
  tags = { Name = var.nombre }
}
```

outputs.tf

```hcl
output "ip_publica" { value = aws_instance.this.public_ip }
```

Instanciación en `dev/` y `prod/` con `terraform.tfvars` diferentes.

Ejecutar:

{% stepper %}
{% step %}
cd dev terraform init terraform apply # crea t2.micro (estado: dev/terraform.tfstate)
{% endstep %}

{% step %}
cd ../prod terraform init terraform apply # crea t3.small (estado: prod/terraform.tfstate)
{% endstep %}
{% endstepper %}

Resultado: aislamiento total entre dev y prod.

### 4. Método 2: CLI Workspaces (Código Único)

Estructura:

entornos-por-workspace/

* main.tf
* variables.tf
* outputs.tf

Ejemplo main.tf (con `terraform.workspace`):

```hcl
locals {
  entornos = {
    "default" = { instance_type = "t2.micro" nombre = "servidor-default" }
    "dev"     = { instance_type = "t2.micro" nombre = "servidor-dev" }
    "prod"    = { instance_type = "t3.small" nombre = "servidor-PRODUCCION" }
  }
  config_actual = local.entornos[terraform.workspace]
}

resource "aws_instance" "this" {
  ami           = var.ami
  instance_type = local.config_actual.instance_type
  tags = {
    Name    = local.config_actual.nombre
    Entorno = terraform.workspace
  }
}
```

Ejecutar:

{% stepper %}
{% step %}
cd entornos-por-workspace terraform init
{% endstep %}

{% step %}
## Desplegar dev

terraform workspace new dev terraform apply -var="ami=ami-..." # estado: terraform.tfstate.d/dev/terraform.tfstate
{% endstep %}

{% step %}
## Desplegar prod

terraform workspace new prod terraform workspace select prod terraform apply -var="ami=ami-..." # estado: terraform.tfstate.d/prod/terraform.tfstate
{% endstep %}
{% endstepper %}

### 5. Conclusión: ¿Cuál Elegir?

| Criterio     | Método 1: Directorios                                | Método 2: CLI Workspaces                              |
| ------------ | ---------------------------------------------------- | ----------------------------------------------------- |
| Aislamiento  | Excelente (estados separados, blast radius separado) | Bueno (estados separados, mismo código)               |
| Complejidad  | Más archivos (boilerplate)                           | Código más complejo (ifs, maps, locals)               |
| Flexibilidad | Total (prod puede tener recursos que dev no tiene)   | Limitada (dev y prod deben compartir lógica)          |
| Veredicto    | Recomendado para empresas                            | Bueno para proyectos pequeños o para separar regiones |

***

## Módulo 10.7: Versionado de Módulos en Git

### 1. Introducción: Fuentes de Módulos

El argumento `source` en un bloque module puede apuntar a:

* Directorio local: `source = "./modules/mi-vpc"`
* Terraform Registry: `source = "terraform-aws-modules/vpc/aws"`
* GitHub: `source = "github.com/usuario/repo"`
* Git genérico: `source = "git::https://mi-git-interno.com/repo.git"`
* S3, HTTP, etc.

### 2. El Problema: El "Trineo del Infierno" (Sled Hell)

Si usas `source` apuntando a Git sin fijar referencia, Terraform por defecto clona la rama `main`, lo que puede introducir cambios rompientes de forma inesperada.

Ejemplo peligroso:

```hcl
module "vpc_dev" {
  source = "git::https://github.com/mi-empresa/terraform-aws-vpc.git"
}
```

### 3. La Solución: Fijación de Versiones (Version Pinning)

Fije el módulo a un tag o ref inmutable:

```hcl
module "vpc_dev" {
  source = "git::https://github.com/mi-empresa/terraform-aws-vpc.git?ref=v1.2.0"
}
```

Practique semver y actualice deliberadamente de `v1.2.0` a `v1.2.1`, `v1.3.0`, o `v2.0.0` según corresponda.

***

## Módulo 10.8: Estrategias de Mantenimiento de Módulos

### 1. Introducción: Módulos como Productos

Si crea módulos para que otros los consuman, actúa como dueño de producto: debe mantener una API estable y procesos de publicación.

### 2. El Contrato de API: variables.tf y outputs.tf

* Sea explícito en variables: `type`, `description`, `default` cuando aplica.
* Use `validation { ... }` para reglas de entrada.
* Exponga solo las salidas necesarias en `outputs.tf`.

### 3. El Ciclo de Vida del Mantenimiento

1. Desarrollo en una rama feature.
2. Pruebas (ej. `examples/`, `terraform test`).
3. Revisión por PR.
4. Merge a main.
5. Publicación: crear un Tag de Git inmutable (`git tag v1.2.1 && git push --tags`) y actualizar CHANGELOG.md.

### 4. Estrategias de Alcance (Scoping)

* Evite módulos monolito; prefiera múltiples módulos pequeños (vpc, eks, rds, alb).
* Prefiera composición en el Módulo Raíz en vez de anidar llamadas internas entre módulos.

***

## Módulo 10.9: Lab - Ejemplo de Proyecto Modularizado (Composición)

### 1. Introducción

Aplicaremos la **Composición de Módulos** para reconstruir una aplicación web simple (ALB + ASG) usando módulos locales.

### 2. Estructura de Carpetas

proyecto-composicion/

* main.tf (el "director de orquesta")
* outputs.tf
* providers.tf
* variables.tf
* modules/
  * vpc/
  * seguridad/
  * alb/
  * asg/

### 3. Paso 1: Código de los Módulos (Interfaz Abreviada)

* modules/vpc/ — variables: `vpc_cidr`, `public_cidrs`, `private_cidrs`; outputs: `vpc_id`, `public_subnet_ids`, `private_subnet_ids`.
* modules/seguridad/ — variables: `vpc_id`; outputs: `alb_sg_id`, `app_sg_id`.
* modules/alb/ — variables: `vpc_id`, `subnet_ids`, `sg_id`; outputs: `alb_dns_name`, `target_group_arn`.
* modules/asg/ — variables: `vpc_id`, `subnet_ids`, `sg_id`, `target_group_arn`; outputs: `asg_name`.

### 4. Paso 2: El Módulo Raíz (Composición) (main.tf)

```hcl
provider "aws" { region = "us-east-1" }

module "vpc" {
  source       = "./modules/vpc"
  vpc_cidr     = "10.0.0.0/16"
  public_cidrs = ["10.0.1.0/24", "10.0.2.0/24"]
  private_cidrs= ["10.0.10.0/24", "10.0.11.0/24"]
}

module "seguridad" {
  source = "./modules/seguridad"
  vpc_id = module.vpc.vpc_id
}

module "alb" {
  source     = "./modules/alb"
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.public_subnet_ids
  sg_id      = module.seguridad.alb_sg_id
}

module "asg" {
  source           = "./modules/asg"
  vpc_id           = module.vpc.vpc_id
  subnet_ids       = module.vpc.private_subnet_ids
  sg_id            = module.seguridad.app_sg_id
  target_group_arn = module.alb.target_group_arn
}
```

### 5. Paso 3: Salidas Raíz (outputs.tf)

```hcl
output "url_de_la_aplicacion" {
  description = "El DNS público de la aplicación"
  value       = module.alb.alb_dns_name
}
```

### 6. Paso 4: Ejecutar

{% stepper %}
{% step %}
`terraform init`: descargará/registrará los módulos locales.
{% endstep %}

{% step %}
`terraform plan`: Terraform calcula el grafo de dependencias (VPC → Seguridad → ALB/ASG).
{% endstep %}

{% step %}
`terraform apply`: crea la pila completa.
{% endstep %}

{% step %}
Verifique `url_de_la_aplicacion` en el navegador (debería mostrar Nginx).
{% endstep %}

{% step %}
`terraform destroy` limpia todo en orden correcto.
{% endstep %}
{% endstepper %}

Nota del formador: el main.tf raíz ahora es declarativo y legible; la complejidad está abstraída en módulos.

***

## Módulo 10.10: Buenas Prácticas en el Uso de Módulos

### 1. Resumen: Reglas de Oro

1. Alcance pequeño y propósito único: un módulo = una cosa bien hecha.
2. Fijar versiones: nunca use módulos sin versión.
   * Registry: `version = "~> 5.0"`
   * Git: `?ref=v1.2.0`
3. Composición sobre anidamiento: el Raíz debe orquestar los módulos.
4. API clara: todas las variables con `description` y `type`; use `validation {}`; exponga solo outputs necesarios.
5. No declarar providers en módulos hijos: los módulos hijos deben heredar providers del Raíz.
6. Documentación: cada módulo debe tener README.md (inputs, outputs, ejemplos). Use `terraform-docs` para automatizar.

***

Fin del Módulo 10.
