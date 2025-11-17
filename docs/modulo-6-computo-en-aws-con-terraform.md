# Módulo 6 - CÓMPUTO EN AWS CON TERRAFORM

## Módulo 6.1: Lab - Creación de Instancias EC2 con Terraform

### 1. Introducción al aws\_instance

Hasta ahora, hemos construido la "ciudad" (VPC), las "calles" (Tablas de Rutas) y los "distritos" (Subredes). Ahora es el momento de construir nuestra primera "casa": una **Instancia EC2**.

El recurso aws\_instance nos permite definir y gestionar un servidor virtual. Es el recurso de cómputo fundamental de AWS. Para lanzar una instancia, debemos tomar varias decisiones: ¿qué software (AMI)? ¿qué hardware (tipo de instancia)? ¿en qué red (subred)? ¿y qué firewall (grupo de seguridad)?

### 2. Objetivo del Laboratorio

Lanzar una única instancia EC2 (un servidor web básico) en una de nuestras **subredes públicas** para que podamos acceder a ella desde Internet (con fines de prueba).

### 3. Prerrequisitos

Este laboratorio se basa **directamente** en el código de nuestro Módulo 5. Debe tener su proyecto de VPC (terraform-vpc) listo, ya que _referenciaremos_ sus salidas (outputs).

* Una VPC (aws\_vpc)
* Subredes públicas (aws\_subnet.public)
* Un Security Group (aws\_security\_group.web) que permita HTTP (80) y SSH (22)

### 4. Estructura de Archivos

Continuaremos en el mismo proyecto (terraform-vpc) y simplemente añadiremos nuestro código a main.tf y outputs.tf.

### 5. Paso 1: Añadir el Recurso aws\_instance (main.tf)

Añada este bloque de código a su archivo main.tf.

```terraform
# main.tf (añadir)
# ... (VPC, Subredes, SGs, ALB... del Módulo 5) ...
# 6.1 Crear una instancia EC2
resource "aws_instance" "servidor_web_publico_1" {
  # 1. ¿Qué software? (Amazon Machine Image)
  # Este ID es para Amazon Linux 2 en us-east-1.
  # ¡Lo mejoraremos en el próximo módulo!
  ami = "ami-0c7217cdde317cfec"

  # 2. ¿Qué hardware?
  instance_type = "t2.micro" # Incluido en la capa gratuita

  # 3. ¿En qué red? (Integración con Módulo 5)
  # 'aws_subnet.public.*.id' es una LISTA.
  # Usamos el índice [0] para coger la primera subred pública.
  subnet_id = aws_subnet.public[0].id

  # 4. ¿Qué firewall? (Integración con Módulo 5)
  # Usamos el SG que ya creamos para la web.
  vpc_security_group_ids = [aws_security_group.web.id]

  # 5. ¡Necesitamos una clave para SSH! (Lo veremos en el Módulo 6.3)
  # key_name = "mi-clave-ssh" # Por ahora, lo dejamos comentado

  # 6. User Data (Lo veremos en el Módulo 6.4)
  # user_data = "..." # Por ahora, lo dejamos vacío

  tags = {
    Name = "servidor-web-publico-1"
  }
}
```

### 6. Paso 2: Añadir Salidas (outputs.tf)

Queremos saber la IP pública de nuestro servidor.

```terraform
# outputs.tf (añadir)
output "ip_publica_servidor_web" {
  description = "La IP pública de nuestra instancia EC2"
  value       = aws_instance.servidor_web_publico_1.public_ip
}
```

### 7. Paso 3: Ejecutar y Verificar

{% stepper %}
{% step %}
### Ejecutar comandos básicos

* terraform init (por si acaso, aunque no hay nuevos providers).
* terraform plan: Verá un plan para 1 to add (el aws\_instance).
* terraform apply: Escriba yes. Terraform creará la instancia en unos 30-60 segundos.
{% endstep %}

{% step %}
### Verificar desde Terraform y la consola

* Salida: Mire la salida de Terraform. Verá ip\_publica\_servidor\_web = "54.x.x.x".
* Consola de AWS: Vaya a EC2 -> Instancias. Verá su servidor-web-publico-1 en estado "running".
* Red: Compruebe que está en la VPC y Subred correctas. Compruebe su pestaña "Seguridad" y vea que el sg-web está adjunto.
{% endstep %}

{% step %}
### Prueba en el navegador

* Copie la IP pública de la salida y péguela en su navegador (http://54.x.x.x).
* Resultado esperado: El navegador se quedará "cargando" y mostrará un error de "Tiempo de espera agotado".
* Explicación: Hemos permitido el tráfico por el firewall (SG), pero la instancia no tiene ningún software (como Nginx o Apache) ejecutándose en el puerto 80 para responder.
{% endstep %}

{% step %}
### Limpieza

* terraform destroy: Limpie la instancia.
{% endstep %}
{% endstepper %}

***

## Módulo 6.2: Selección de AMIs y Tipos de Instancia (data "aws\_ami")

### 1. El Problema de las AMIs "Hardcodeadas"

En el laboratorio anterior, usamos ami = "ami-0c7217cdde317cfec". Esto es frágil.

* Es específico de la región: ese ID de AMI solo existe en us-east-1.
* Se vuelve obsoleto: Amazon publica nuevas AMIs con parches y la nuestra se quedará antigua.

### 2. La Solución: Búsqueda Dinámica con data "aws\_ami"

Use un Data Source para buscar la AMI más reciente que cumpla filtros.

```terraform
# main.tf (o un archivo data.tf)
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}
```

Resultado: data.aws\_ami.amazon\_linux\_2.id contendrá el ID de la AMI más reciente en la región actual.

### 3. Lab: Actualización de aws\_instance

Actualice la instancia para usar la AMI dinámica.

```terraform
# main.tf (MODIFICAR)
data "aws_ami" "amazon_linux_2" { ... } # (ver arriba)

resource "aws_instance" "servidor_web_publico_1" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  # ... (subnet_id, vpc_security_group_ids, etc. se quedan igual)
  tags = { Name = "servidor-web-publico-1" }
}
```

Nota: terraform apply probablemente planeará destruir y reemplazar la instancia (-/+), porque el atributo ami ha cambiado. Esto es esperado.

### 4. Tipos de Instancia (Hardware)

El argumento instance\_type define CPU, memoria y red.

* Familias comunes:
  * t: burstable (t3.micro, t3.small)
  * m: multipropósito (m5.large)
  * c: cómputo (c5.large)
  * r: memoria (r5.large)
  * g: Graviton (ARM)
* Buena práctica: comience con t3.micro o t3.small y monitorice con CloudWatch.

### 5. Material de Apoyo

* Documentación de data "aws\_ami": https://www.google.com/search?q=%5Bhttps://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/ami%5D(https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/ami)
* Tipos de Instancia de Amazon EC2: https://aws.amazon.com/es/ec2/instance-types/

***

## Módulo 6.3: Manejo de Claves SSH en Terraform (aws\_key\_pair)

### 1. El Problema: Acceso a Instancias Privadas

Para instancias en subredes privadas, en producción se usan bastion hosts o Session Manager. Para el curso usaremos pares de claves SSH.

### 2. ¿Qué es un Par de Claves SSH?

* Clave privada (id\_rsa): la guarda usted.
* Clave pública (id\_rsa.pub): se copia en el servidor.

### 3. Lab: Creación y Carga de un Par de Claves

#### Generar un par de claves (en su terminal local)

Ejecute en su terminal (no en Terraform):

```bash
ssh-keygen -t ed25519 -C "mi-clave-aws-terraform"
# Acepte el path por defecto (ej. ~/.ssh/id_ed25519)
```

Esto crea id\_ed25519 (privada) e id\_ed25519.pub (pública).

#### Añadir el recurso aws\_key\_pair (main.tf)

Defina una variable con la ruta y suba la clave pública:

```terraform
# main.tf (añadir)
variable "ruta_clave_publica_ssh" {
  description = "Ruta a su clave pública SSH (ej. ~/.ssh/id_ed25519.pub)"
  type        = string
  default     = "~/.ssh/id_ed25519.pub"
}

resource "aws_key_pair" "mi_clave" {
  key_name   = "mi-clave-terraform"
  public_key = file(var.ruta_clave_publica_ssh)
}
```

La función file(path) lee el contenido de un archivo local.

#### Actualizar la instancia EC2 para usar la clave

```terraform
# main.tf (MODIFICAR)
resource "aws_instance" "servidor_web_publico_1" {
  ami                     = data.aws_ami.amazon_linux_2.id
  instance_type           = "t2.micro"
  subnet_id               = aws_subnet.public[0].id
  vpc_security_group_ids  = [aws_security_group.web.id]
  key_name                = aws_key_pair.mi_clave.key_name
  tags = {
    Name = "servidor-web-publico-1"
  }
}
```

### 4. Paso 4: Ejecutar y Verificar

{% stepper %}
{% step %}
* terraform init
* terraform apply (yes). Terraform creará el aws\_key\_pair y recreará la instancia (-/+).
{% endstep %}

{% step %}
Verificación en consola:

* Consola AWS -> EC2 -> Pares de Claves: verá mi-clave-terraform.
{% endstep %}

{% step %}
Probar conexión SSH (en su terminal):

```bash
ssh -i ~/.ssh/id_ed25519 ec2-user@54.x.x.x
```

Si el sg-web permite el puerto 22 desde su IP, debería conectarse.
{% endstep %}

{% step %}
* terraform destroy: Limpie todo (instancia y par de claves).
{% endstep %}
{% endstepper %}

***

## Módulo 6.4: Configuración de User Data (Despliegue de Aplicaciones)

### 1. El Problema: Instancias Vacías

Las instancias sin software retornan timeout en el navegador. Necesitamos instalar software automáticamente al arrancar.

### 2. La Solución: user\_data

user\_data acepta un script (p. ej. Bash) que se ejecuta una sola vez al primer arranque para bootstrap.

### 3. Lab: Lanzar un Servidor Web Nginx

Objetivo: lanzar una instancia EC2 con clave SSH, AMI dinámica y user\_data que instale Nginx.

#### Paso 1: Crear el Script de User Data

Cree un archivo llamado install\_nginx.sh en su proyecto:

```bash
#!/bin/bash
# install_nginx.sh

# Actualizar los paquetes e instalar Nginx
yum update -y
yum install -y nginx

# Iniciar el servicio Nginx
systemctl start nginx
systemctl enable nginx

# Crear una página de bienvenida simple
echo "<h1>¡Hola, Mundo desde Terraform! (Instancia: $(hostname -f))</h1>" > /usr/share/nginx/html/index.html
```

#### Paso 2: Actualizar aws\_instance para usar user\_data

Use file() para leer el script.

```terraform
# main.tf (MODIFICAR)
resource "aws_instance" "servidor_web_publico_1" {
  ami                    = data.aws_ami.amazon_linux_2.id
  instance_type          = "t2.micro"
  subnet_id              = aws_subnet.public[0].id
  vpc_security_group_ids = [aws_security_group.web.id]
  key_name               = aws_key_pair.mi_clave.key_name

  user_data = file("install_nginx.sh")

  tags = { Name = "servidor-web-publico-1" }
}
```

### 4. Paso 3: Ejecutar y Verificar

{% stepper %}
{% step %}
* terraform apply (yes). Terraform (probablemente) recreará la instancia (-/+).
{% endstep %}

{% step %}
* Espere 1-2 minutos para que el script se ejecute.
* Obtener ip\_publica\_servidor\_web desde terraform output.
* Abrir http://54.x.x.x en el navegador.
{% endstep %}

{% step %}
* Resultado esperado: verá "¡Hola, Mundo desde Terraform! (Instancia: ...)".
* terraform destroy: Limpie todo.
{% endstep %}
{% endstepper %}

***

## Módulo 6.5: Lab - Auto Scaling Groups (ASG)

### 1. El Problema: Una Sola Instancia (Una Mascota)

Una sola instancia falla o se sobrecarga. Necesitamos instancias anónimas y reemplazables: el patrón "Cattle".

### 2. La Solución Moderna: aws\_launch\_template

Use aws\_launch\_template para definir la "receta" de una instancia (AMI, tipo, clave, SG, user\_data) sin crearla inmediatamente.

### 3. Objetivo del Laboratorio

1. Crear aws\_launch\_template que defina nuestro servidor Nginx.
2. Crear aws\_autoscaling\_group que use la plantilla y lance instancias en subredes privadas.
3. (En el próximo lab) conectar el ASG a un Load Balancer.

### 4. Paso 1: Crear la Plantilla de Lanzamiento (main.tf)

Reúna AMI, tipo, clave, SG, user\_data:

```terraform
# main.tf (AÑADIR - puede eliminar el aws_instance.servidor_web_publico_1)
resource "aws_launch_template" "plantilla_app_web" {
  name               = "plantilla-app-web-nginx"
  image_id           = data.aws_ami.amazon_linux_2.id
  instance_type      = "t2.micro"
  key_name           = aws_key_pair.mi_clave.key_name
  vpc_security_group_ids = [aws_security_group.web.id]

  # filebase64() es más robusto que file() para user_data
  user_data = filebase64("install_nginx.sh")

  tags = { Name = "plantilla-app-web" }
}
```

### 5. Paso 2: Crear el Auto Scaling Group (ASG) (main.tf)

```terraform
# main.tf (continuación)
resource "aws_autoscaling_group" "asg_app_web" {
  name               = "asg-app-web"
  min_size           = 2
  max_size           = 5
  desired_capacity   = 2

  # Lanzamos el ASG en las subredes PRIVADAS, no en las públicas.
  vpc_zone_identifier = aws_subnet.private.*.id

  launch_template {
    id      = aws_launch_template.plantilla_app_web.id
    version = "$Latest"
  }

  # (En el próximo lab, añadiremos la conexión al ELB aquí)
}
```

### 6. Paso 3: Ejecutar y Verificar

{% stepper %}
{% step %}
* terraform apply (yes). Terraform creará la plantilla y el ASG. El ASG tardará 1-2 minutos en lanzar sus 2 instancias.
{% endstep %}

{% step %}
Verificación:

* Consola AWS -> EC2 -> Auto Scaling Groups: verá asg-app-web con capacidad deseada 2.
* Pestaña "Instancias": verá 2 instancias InService.
* Verifique que estén en subredes privadas (sin IP pública).
{% endstep %}

{% step %}
Prueba de auto-reparación:

* Termine una de las instancias manualmente (Actions -> Terminate).
* Espere 1-2 minutos: el ASG lanzará una nueva instancia para volver a desired\_capacity = 2.
{% endstep %}

{% step %}
* terraform destroy: Limpie todo.
{% endstep %}
{% endstepper %}

***

## Módulo 6.6: Lab - Conexión de ASG con Balanceador de Carga

### 1. Introducción

Unimos Módulo 5 (ALB y target groups) con Módulo 6.5 (ASG). Conectaremos el ASG al Target Group para que el ALB envíe tráfico a las instancias.

### 2. Objetivo del Laboratorio

Usar target\_group\_arns en aws\_autoscaling\_group para registrar automáticamente instancias en el Grupo de Destino (aws\_lb\_target\_group).

### 3. Paso 1: Actualizar el Auto Scaling Group (main.tf)

Modificar aws\_autoscaling\_group para incluir target\_group\_arns, health\_check\_type y health\_check\_grace\_period:

```terraform
# main.tf (MODIFICAR el 'aws_autoscaling_group')
resource "aws_autoscaling_group" "asg_app_web" {
  name               = "asg-app-web"
  min_size           = 2
  max_size           = 5
  desired_capacity   = 2
  vpc_zone_identifier = aws_subnet.private.*.id

  launch_template {
    id      = aws_launch_template.plantilla_app_web.id
    version = "$Latest"
  }

  # Registrar automáticamente sus instancias en el TG del Lab 5.4
  target_group_arns = [ aws_lb_target_group.app.arn ]

  # El ASG esperará a que el ELB marque la instancia como 'healthy'
  health_check_type         = "ELB"
  health_check_grace_period = 300 # 5 minutos
}
```

### 4. Paso 2: Actualizar la Seguridad (¡Crítico!)

Problema de seguridad: si tanto el ALB como las instancias permiten 0.0.0.0/0:80, un atacante puede saltarse el ALB y golpear las instancias directamente.

Solución: separar SGs.

```terraform
# main.tf (REFACTORIZACIÓN DE SEGURIDAD)

# 1. SG para el ALB
resource "aws_security_group" "alb" {
  name        = "sg-alb"
  description = "Permite tráfico HTTP/HTTPS desde Internet"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "sg-alb" }
}

# 2. SG para las instancias (solo desde el ALB)
resource "aws_security_group" "app" {
  name        = "sg-app-web"
  description = "Permite tráfico solo desde el ALB"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port       = 8080    # El puerto del Target Group
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "sg-app" }
}

# 3. Actualizar el ALB para usar sg-alb
resource "aws_lb" "alb_principal" {
  # ... (resto igual) ...
  security_groups = [aws_security_group.alb.id]
}

# 4. Actualizar el Launch Template para usar sg-app
resource "aws_launch_template" "plantilla_app_web" {
  # ... (resto igual) ...
  vpc_security_group_ids = [aws_security_group.app.id]
}
```

Nota: también hay que alinear puertos entre install\_nginx.sh y el target group (8080 vs 80). Para simplicidad, asuma que both escuchan el mismo puerto (o ajuste install\_nginx.sh o el TG acorde).

### 5. Paso 3: Ejecutar y Verificar (La Prueba Final)

{% stepper %}
{% step %}
* terraform apply (yes). Terraform realizará los cambios.
{% endstep %}

{% step %}
* Espere 2-3 minutos para que el ASG lance instancias y el ELB las marque como healthy.
* Consola AWS -> EC2 -> Grupos de Destino -> tg-app-principal -> pestaña "Destinos": verá 2 instancias en estado "healthy".
{% endstep %}

{% step %}
Prueba final:

* Copie el alb\_dns\_name (o app.mi-empresa.com) en su navegador.
* Debería ver la página "¡Hola, Mundo desde Terraform!".
* Refresque varias veces: el hostname en el HTML cambiará entre instancias, demostrando load balancing.
{% endstep %}

{% step %}
* terraform destroy: Limpie toda la infraestructura.
{% endstep %}
{% endstepper %}

Nota del formador: la refactorización de SG (sg-alb y sg-app) es la parte más importante de seguridad en este módulo.

***

## Módulo 6.7: Generación de Imágenes AMI Personalizadas (Packer)

### 1. El Problema con user\_data

Instalar paquetes en boot (yum update, instalar nginx) tarda y hace que nuevas instancias tarden minutos en estar "healthy".

### 2. La Solución: Golden AMI (AMI Dorada)

Baking: crear una AMI personalizada con todo preinstalado (Nginx, parches, aplicación). Así las instancias arrancan rápido.

Flujo:

1. Lanzar instancia base.
2. Instalar y configurar.
3. Crear AMI de esa instancia.
4. Usar la AMI en el launch\_template.

### 3. La Herramienta: HashiCorp Packer

Packer automatiza la creación de AMIs.

* Escriba una plantilla (HCL o JSON) con provisioners (scripts).
* packer build ... crea la AMI y devuelve su ID.
* Guarde el ID en SSM Parameter Store o en un artefacto que Terraform pueda leer.

### 4. Flujo CI/CD Recomendado (Packer + Terraform)

1. Pipeline de construcción de AMI (ej. nocturno):
   * packer build template.pkr.hcl
   * Packer publica ami-nginx-app-v1.2 y guarda el ID en SSM.
2. Pipeline de despliegue:
   * terraform plan
   * Terraform lee el ID de AMI desde SSM y actualiza launch\_template / ASG con rolling updates.

### 5. Material de Apoyo

* Tutorial Packer (HashiCorp): https://developer.hashicorp.com/packer/tutorials/aws-get-started/aws-get-started-build-image

***

## Módulo 6.8: Casos Prácticos de Cómputo en AWS

### 1. Resumen de Patrones

* Patrón Pet (Mascota)
  * Recurso: aws\_instance
  * Uso: servidores únicos (bastion, controladores), gestionados manualmente.
* Patrón Cattle (Ganado)
  * Recursos: aws\_launch\_template + aws\_autoscaling\_group + aws\_lb\_target\_group
  * Uso: aplicaciones web elásticas y desechables.
  * Gestión: user\_data (frying) o AMIs golden (baking).
* Serverless
  * Recursos: aws\_lambda\_function, aws\_ecs\_service (Fargate)
  * Uso: ejecutar código/contenedores sin gestionar EC2.

### 2. Caso Práctico: Despliegue Blue/Green

Proceso para actualizar sin downtime:

1. Estado azul: ALB -> tg-azul -> asg-azul (v1.1).
2. Lanzar verde: crear launch\_template (v1.2), asg-verde, tg-verde.
3. Probar verde: reglas del listener o pruebas desde su IP.
4. Cambiar listener: apuntar ALB a tg-verde (instantáneo).
5. Limpiar: eliminar stack azul después de validar.

***

Fin del Módulo 6. Si quiere, puedo:

* Extraer fragmentos de Terraform en archivos listos para pegar.
* Generar un ejemplo de plantilla Packer básica.
* Convertir cualquier paso numerado adicional en steppers separados para su GitBook. ¿Qué prefiere a continuación?
