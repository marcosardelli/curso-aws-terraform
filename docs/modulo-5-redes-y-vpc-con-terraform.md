# Módulo 5 - REDES Y VPC CON TERRAFORM

## Módulo 5.1: Lab - Creación de VPC y Subredes

### 1. Introducción: La Red como Código

En el Módulo 2, aprendimos conceptualmente que la **VPC (Virtual Private Cloud)** es nuestro centro de datos privado virtual. Es el contenedor de red aislado donde vivirá toda nuestra infraestructura<sup>1</sup>.

Las **Subredes (Subnets)** son las particiones de esa red, y cada una está vinculada a una **Zona de Disponibilidad (AZ)** específica<sup>2</sup>.

En este laboratorio, dejaremos de dibujar diagramas y comenzaremos a _codificar_ nuestra red. Construiremos la base para una arquitectura de alta disponibilidad (Multi-AZ).

### 2. Objetivo del Laboratorio

Desplegar una arquitectura de red fundacional y robusta que sirva de base para los módulos siguientes.

* **Nuestra Meta:** Crear una VPC con **seis subredes** distribuidas en **dos Zonas de Disponibilidad (AZs)**.
  * **2 Subredes Públicas:** Para recursos de cara a Internet (como Balanceadores de Carga).
  * **2 Subredes Privadas:** Para nuestros servidores de aplicación (EC2).
  * **2 Subredes Aisladas/Datos:** Para nuestras bases de datos (RDS).

Este patrón es una buena práctica estándar de AWS Well-Architected.

### 3. Estructura de Archivos

1. Cree una nueva carpeta de proyecto (ej. terraform-vpc).
2. Cree los archivos estándar: providers.tf, variables.tf, main.tf, outputs.tf.

### 4–9. Pasos del Lab (Resumen ejecutable)

{% stepper %}
{% step %}
### Configurar Providers y Variables

Configuramos el provider de AWS y definimos los rangos de IP (CIDR) que usaremos.

providers.tf

```terraform
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

variables.tf

```terraform
variable "aws_region" {
  description = "Región de AWS para el despliegue"
  type        = string
  default     = "us-east-1"
}

variable "vpc_cidr" {
  description = "Bloque CIDR principal para la VPC"
  type        = string
  default     = "10.0.0.0/16" # Nos da 65,536 IPs
}

variable "public_subnet_cidrs" {
  description = "Bloques CIDR para las subredes públicas"
  type        = list(string)
  default     = ["10.0.1.0/24", "10.0.2.0/24"] # 2 subredes en 2 AZs
}

variable "private_subnet_cidrs" {
  description = "Bloques CIDR para las subredes de aplicación"
  type        = list(string)
  default     = ["10.0.10.0/24", "10.0.11.0/24"]
}

variable "data_subnet_cidrs" {
  description = "Bloques CIDR para las subredes de base de datos"
  type        = list(string)
  default     = ["10.0.20.0/24", "10.0.21.0/24"]
}
```
{% endstep %}

{% step %}
### Obtener Zonas de Disponibilidad (Data Source)

No queremos "hardcodear" las AZs (ej. us-east-1a). Usamos un data source para pedírselas a AWS.

main.tf

```terraform
# Data source para obtener la lista de AZs disponibles en la región actual
data "aws_availability_zones" "available" {
  state = "available"
}
```
{% endstep %}

{% step %}
### Crear la VPC

main.tf (continuación)

```terraform
# El recurso VPC principal
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "vpc-principal"
  }
}
```
{% endstep %}

{% step %}
### Crear las Subredes (con count)

Usamos count para iterar sobre nuestras listas de variables y crear las subredes públicas, privadas y de datos.

main.tf (continuación)

```terraform
# --- Subredes Públicas ---
resource "aws_subnet" "public" {
  count                   = length(var.public_subnet_cidrs)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "subnet-publica-${count.index + 1}"
  }
}

# --- Subredes Privadas (App) ---
resource "aws_subnet" "private" {
  count                   = length(var.private_subnet_cidrs)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.private_subnet_cidrs[count.index]
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = false

  tags = {
    Name = "subnet-privada-app-${count.index + 1}"
  }
}

# --- Subredes Aisladas (Datos) ---
resource "aws_subnet" "data" {
  count                   = length(var.data_subnet_cidrs)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.data_subnet_cidrs[count.index]
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = false

  tags = {
    Name = "subnet-privada-datos-${count.index + 1}"
  }
}
```

Notas:

* count.index es un índice (0, 1) usado durante la iteración.
* var.public\_subnet\_cidrs\[count.index] asigna el CIDR correspondiente a cada iteración/AZ.
{% endstep %}

{% step %}
### Exponer las Salidas (outputs.tf)

Exponemos los IDs de los recursos que creamos para que otros módulos los necesiten.

outputs.tf

```terraform
output "vpc_id" {
  description = "El ID de la VPC principal"
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "Lista de IDs de las subredes públicas"
  value       = aws_subnet.public.*.id
}

output "private_subnet_ids" {
  description = "Lista de IDs de las subredes privadas"
  value       = aws_subnet.private.*.id
}
```
{% endstep %}

{% step %}
### Ejecutar y Verificar

* terraform init: Descarga el provider aws y el data source de AZs.
* terraform plan: Revise el plan. Debería ver que Terraform va a crear 7 recursos: 1 VPC y 6 Subredes.
* terraform apply: Escriba yes para crear la red.
* Verificación en la Consola AWS:
  * VPC -> verá vpc-principal.
  * Subredes -> verá sus 6 nuevas subredes; compruebe AZs y que las subredes públicas tienen "Asignar IP pública automáticamente" en "Sí".
* terraform destroy: Limpie todos los recursos.
{% endstep %}
{% endstepper %}

***

## Módulo 5.2: Lab - Conectividad de Red (Gateways y Route Tables)

### 1. Introducción

En el laboratorio anterior, construimos las "habitaciones" (subredes) de nuestra casa (VPC). Sin embargo, no hemos construido ni la "puerta de entrada" (a Internet) ni los "pasillos" (rutas) que conectan todo.

* Una **Subred Pública** se define por tener una ruta a un **Internet Gateway (IGW)**.
* Una **Subred Privada** (que necesita acceder a Internet para parches, etc.) se define por tener una ruta a un **NAT Gateway (NGW)**.

### 2. Objetivo del Laboratorio

Basándonos en el main.tf del laboratorio 5.1, añadiremos los componentes de enrutamiento:

* Crear un **Internet Gateway (IGW)** para la VPC.
* Crear un **NAT Gateway (NGW)** y su **IP Elástica**.
* Crear una **Tabla de Rutas Pública** y asociarla a las subredes públicas.
* Crear una **Tabla de Rutas Privada** y asociarla a las subredes privadas.

{% stepper %}
{% step %}
### Crear el Internet Gateway

main.tf (añadir a lo existente)

```terraform
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "igw-principal"
  }
}
```
{% endstep %}

{% step %}
### Crear el NAT Gateway (y EIP)

* Un NAT Gateway debe vivir en una Subred Pública.
* Requiere una IP Elástica (EIP).

main.tf (continuación)

```terraform
# IP Elástica (Requerida por el NAT Gateway)
resource "aws_eip" "nat" {
  domain = "vpc"
  depends_on = [aws_internet_gateway.igw]
}

# NAT Gateway
resource "aws_nat_gateway" "ngw" {
  subnet_id     = aws_subnet.public[0].id
  allocation_id = aws_eip.nat.id

  tags = {
    Name = "nat-gateway-principal"
  }
}
```

Advertencia: Los NAT Gateways tienen coste por hora y por GB procesado. Haga destroy cuando termine.
{% endstep %}

{% step %}
### Crear Tablas de Rutas

main.tf (continuación)

```terraform
# Tabla de Rutas PÚBLICA
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
    Name = "rt-publica"
  }
}

# Tabla de Rutas PRIVADA (para Apps)
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block    = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.ngw.id
  }

  tags = {
    Name = "rt-privada-app"
  }
}

# Nota: Las subredes 'data' usarán la route table 'main' (aislada).
```
{% endstep %}

{% step %}
### Asociar las Tablas de Rutas a las Subredes

main.tf (continuación)

```terraform
# Asociar las Subredes Públicas
resource "aws_route_table_association" "public" {
  count         = length(var.public_subnet_cidrs)
  subnet_id     = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# Asociar las Subredes Privadas (App)
resource "aws_route_table_association" "private" {
  count         = length(var.private_subnet_cidrs)
  subnet_id     = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private.id
}
```

(Las subredes 'data' se quedan sin asociación explícita y mantienen la ruta 'main' por defecto, aislada.)
{% endstep %}

{% step %}
### Ejecutar y Verificar

* terraform init (si es la primera vez).
* terraform plan: verá IGW, EIP, NGW, 2x RT, y asociaciones.
* terraform apply: escriba yes.
* Verificación en la Consola AWS:
  * VPC -> Tablas de Ruta: rt-publica debe tener 0.0.0.0/0 -> igw-...
  * Asociaciones de Subred: sus subredes públicas deben estar asociadas.
  * rt-privada-app debe tener 0.0.0.0/0 -> nat-...
* terraform destroy: Limpie todo (especialmente NGW y EIP).
{% endstep %}
{% endstepper %}

***

## Módulo 5.3: Lab - Firewalls de VPC (Security Groups y NACLs)

### 1. Introducción: Las Capas de Seguridad

Resumen:

* NACLs: nivel de Subred, stateless, Allow/Deny.
* Security Groups (SG): nivel de Instancia, stateful, solo Allow.

Buena práctica: gestionar la mayoría con Security Groups y dejar NACLs por defecto salvo necesidad de Deny.

### 2. Objetivo del Laboratorio

* Crear un Security Group para un servidor web (sg-web).
* Crear un SG para una base de datos (sg-db).
* Demostrar que los SGs pueden referenciarse entre sí.
* Inspeccionar (sin modificar) las NACLs por defecto.

{% stepper %}
{% step %}
### Variables para la Seguridad

variables.tf (añadir)

```terraform
variable "mi_ip_publica" {
  description = "Mi IP pública (para acceso SSH)"
  type        = string
  # Puede encontrar su IP en https://www.whatismyip.com/
  # Ej: "80.10.20.30/32"
  default     = "80.10.20.30/32"
}
```
{% endstep %}

{% step %}
### Security Group para el Servidor Web

main.tf (añadir)

```terraform
resource "aws_security_group" "web" {
  name        = "sg-servidor-web"
  description = "Permite tráfico HTTP/HTTPS y SSH"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTPS desde cualquier lugar"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTP desde cualquier lugar"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "SSH solo desde mi IP"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.mi_ip_publica]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "sg-web"
  }
}
```
{% endstep %}

{% step %}
### Security Group para la Base de Datos

main.tf (continuación)

```terraform
resource "aws_security_group" "db" {
  name        = "sg-base-de-datos"
  description = "Permite acceso solo desde la capa de aplicación"
  vpc_id      = aws_vpc.main.id

  ingress {
    description     = "MySQL desde el SG de los servidores web"
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.web.id] # Referencia a otro SG
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "sg-db"
  }
}
```

Nota: la regla del SG de la base de datos referencia el SG del web; AWS rastrea las IPs internas.
{% endstep %}

{% step %}
### Gestionar la NACL por Defecto

No creamos una NACL nueva; gestionamos la por defecto si se desea etiquetarla o añadir reglas de Deny (ejemplo comentado).

main.tf (continuación)

```terraform
resource "aws_default_network_acl" "default" {
  default_network_acl_id = aws_vpc.main.default_network_acl_id

  # Por defecto, las reglas permiten todo (Allow All).
  # Podríamos añadir reglas de 'deny' aquí si quisiéramos bloquear una IP.
  #
  # Ejemplo (no aplicar si no es necesario):
  # rule {
  #   rule_number = 200
  #   protocol    = "tcp"
  #   rule_action = "deny"
  #   cidr_block  = "1.2.3.4/32"
  #   from_port   = 0
  #   to_port     = 0
  # }

  tags = {
    Name = "nacl-default"
  }
}
```
{% endstep %}

{% step %}
### Ejecutar y Verificar

* terraform init y terraform apply.
* Verificación en la Consola AWS:
  * VPC -> Grupos de Seguridad: comprobar sg-web (80,443,22) y que la regla 22 es su IP.
  * sg-db: comprobar que la entrada 3306 tiene como origen el ID del SG sg-web.
  * VPC -> ACLs de Red: verá la NACL por defecto con etiqueta.
* terraform destroy: Limpie todo.
{% endstep %}
{% endstepper %}

***

## Módulo 5.4: Lab - Despliegue de Balanceadores de Carga (ELB)

### 1. Introducción

Un Balanceador de Carga (ELB) distribuye el tráfico entrante entre múltiples servidores. Nos enfocaremos en el **Application Load Balancer (ALB)**.

### 2. Componentes de un ALB

* aws\_lb: el ALB.
* aws\_lb\_target\_group: grupo de destino.
* aws\_lb\_listener: oyente que conecta puerto público con target group.

### 3. Objetivo del Laboratorio

Crear un ALB público que escuche en el puerto 80 (HTTP) y esté listo para enviar tráfico a nuestros futuros servidores de aplicaciones.

### 4–9. Pasos del Lab

{% stepper %}
{% step %}
### Usar la VPC y el SG de los Labs Anteriores

Asumimos que ya existen:

* aws\_vpc.main
* aws\_subnet.public
* aws\_security\_group.web
{% endstep %}

{% step %}
### Crear el Balanceador de Carga (ALB)

main.tf (añadir)

```terraform
resource "aws_lb" "alb_principal" {
  name               = "alb-principal-app"
  internal           = false
  load_balancer_type = "application"
  subnets            = aws_subnet.public.*.id
  security_groups    = [aws_security_group.web.id]

  tags = {
    Name = "alb-principal"
  }
}
```
{% endstep %}

{% step %}
### Crear el Grupo de Destino (Target Group)

main.tf (continuación)

```terraform
resource "aws_lb_target_group" "app" {
  name     = "tg-app-principal"
  vpc_id   = aws_vpc.main.id
  port     = 8080
  protocol = "HTTP"

  health_check {
    path                = "/health"
    protocol            = "HTTP"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 3
    unhealthy_threshold = 2
  }

  tags = {
    Name = "tg-app"
  }
}
```
{% endstep %}

{% step %}
### Crear el Oyente (Listener)

main.tf (continuación)

```terraform
resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.alb_principal.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}
```

Nota: en producción añadiríamos también listener 443 con ACM y redirección HTTP->HTTPS.
{% endstep %}

{% step %}
### Exponer la Salida del ALB

outputs.tf (añadir)

```terraform
output "alb_dns_name" {
  description = "El nombre DNS público del ALB"
  value       = aws_lb.alb_principal.dns_name
}
```
{% endstep %}

{% step %}
### Ejecutar y Verificar

* terraform init y terraform apply.
* Verificación en la Consola AWS:
  * EC2 -> Balanceadores de Carga: verá alb-principal-app en las subredes públicas.
  * Oyentes: puerto 80.
  * Grupos de Destino: tg-app-principal (Health Checks configurados). El apartado Destinos estará vacío hasta crear instancias EC2 (Módulo 6).
* Prueba: acceder al alb\_dns\_name. Debería recibir "503 Service Temporarily Unavailable" (esperado si no hay targets).
* terraform destroy: Limpie todo.
{% endstep %}
{% endstepper %}

***

## Módulo 5.5: Lab - DNS y Enrutamiento con Amazon Route 53

### 1. Introducción

Obtuvimos un nombre DNS para nuestro ALB. Para un nombre amigable usamos Route 53. Cuando apunte a recursos AWS use registros ALIAS (mejor que CNAME).

### 2. Objetivo del Laboratorio

* Buscar (data source) una Zona Alojada en Route 53.
* Crear un registro ALIAS que apunte nuestro subdominio al ALB.

### 3–8. Pasos del Lab

{% stepper %}
{% step %}
### Prerrequisito: Zona Alojada (Hosted Zone)

Este laboratorio asume que ya tiene un dominio gestionado como Zona Alojada pública en Route 53.
{% endstep %}

{% step %}
### Definir Variables

variables.tf (añadir)

```terraform
variable "nombre_dominio" {
  description = "El nombre de la Zona Alojada en Route 53 (ej: mi-empresa.com)"
  type        = string
  default     = "mi-empresa-ejemplo.com"
}

variable "subdominio_app" {
  description = "El subdominio para la app (ej: 'app')"
  type        = string
  default     = "app"
}
```
{% endstep %}

{% step %}
### Buscar la Zona Alojada (data source)

main.tf (añadir)

```terraform
data "aws_route53_zone" "mi_zona" {
  name = var.nombre_dominio
  # private_zone = false (implícito)
}
```
{% endstep %}

{% step %}
### Crear el Registro ALIAS

main.tf (continuación)

```terraform
resource "aws_route53_record" "www" {
  zone_id = data.aws_route53_zone.mi_zona.zone_id
  name    = "${var.subdominio_app}.${data.aws_route53_zone.mi_zona.name}"
  type    = "A"

  alias {
    name                   = aws_lb.alb_principal.dns_name
    zone_id                = aws_lb.alb_principal.zone_id
    evaluate_target_health = true
  }
}
```
{% endstep %}

{% step %}
### Ejecutar y Verificar

* terraform init y terraform apply.
* Consola AWS -> Route 53 -> Zonas Alojadas: verá app.mi-empresa.com como A (Alias).
* Prueba:
  * ping app.mi-empresa.com -> resolverá a IPs públicas del ALB.
  * curl http://app.mi-empresa.com -> mostrará el mismo 503 si no hay targets.
* terraform destroy: Limpie el registro.
{% endstep %}
{% endstepper %}

***

## Módulo 5.6: Módulos de Red y Buenas Prácticas

### 1. Introducción: La Deuda Técnica de la Red

Después de los labs, su main.tf puede ser largo y complejo. No copie/pegue para nuevos proyectos: use módulos.

### 2. Plantillas Reutilizables (Módulos)

Ejemplo conceptual usando el módulo oficial:

```terraform
module "mi_vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.8.0"

  name       = "vpc-produccion"
  cidr       = "10.0.0.0/16"
  azs        = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.10.0/24", "10.0.11.0/24"]
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24"]

  enable_nat_gateway  = true
  enable_dns_hostnames = true

  common_tags = {
    Proyecto = "mi-proyecto"
  }
}

resource "aws_instance" "web" {
  ami       = "..."
  subnet_id = module.mi_vpc.private_subnets[0]
}
```

Aprenderemos a escribir y usar módulos en el Módulo 10.

### 3. Validación de la Configuración de Red

* terraform output: comprobar IDs y salidas.
* Consola AWS: inspeccione VPC, Subredes, Tablas de Rutas y asociaciones.
* Pruebas de conectividad (Módulo 6) con instancias EC2 en subredes públicas/privadas.

### 4. Buenas Prácticas de Red Segura (Resumen)

1. Usar módulos probados (ej. terraform-aws-modules).
2. Seguridad por defecto (deny all) en SGs: permitir solo lo necesario.
3. Aislar bases de datos en subredes privadas; SGs solo permiten tráfico desde la app.
4. Minimizar costes de NAT Gateway: un NGW por AZ en lugar de uno por subred.
5. Etiquetar todo (Name, Entorno, Proyecto) para facilitar gestión y costes.

***

Si deseas, puedo:

* Extraer cada laboratorio a páginas separadas listas para GitBook.
* Convertir las secciones de "Pasos" en archivos .tf individuales organizados por recurso (providers.tf, variables.tf, main.tf, outputs.tf) listos para copiar/pegar.
