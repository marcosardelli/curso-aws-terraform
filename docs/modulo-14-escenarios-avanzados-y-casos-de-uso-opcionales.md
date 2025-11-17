# Módulo 14 - ESCENARIOS AVANZADOS Y CASOS DE USO (OPCIONALES)

## Módulo 14.1: Infraestructura Multi-Región y Despliegue Híbrido

### 1. Introducción

Terraform no se limita a una sola región de AWS ni siquiera a una sola nube. Su fortaleza principal es la gestión de sistemas complejos y distribuidos.

### 2. Infraestructura Multi-Región

* **¿Qué es?** Desplegar su aplicación en más de una región de AWS (ej. us-east-1 en N. Virginia y eu-west-1 en Irlanda).
* **¿Por qué?**
  * **Recuperación de Desastres (DR):** Para sobrevivir al fallo de una región entera (un desastre a gran escala).
  * **Latencia:** Para servir a los usuarios desde la región más cercana a ellos (ej. usuarios europeos van a Irlanda, usuarios de EE. UU. van a N. Virginia).
* **¿Cómo se hace en Terraform? (Providers Múltiples)**
  * Como vimos en el Módulo 7 (Replicación S3), Terraform gestiona esto usando **alias de provider**.
  * Se define un provider para cada región y luego se especifica qué provider usar para cada recurso.

#### Ejemplo de Código (Multi-Región)

```terraform
# providers.tf
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}

# Provider Primario (N. Virginia)
provider "aws" {
  region = "us-east-1"
  alias  = "primario"
}

# Provider Secundario (Irlanda)
provider "aws" {
  region = "eu-west-1"
  alias  = "secundario"
}

# --- main.tf ---

# 1. Este recurso se crea en us-east-1 (por defecto)
resource "aws_instance" "servidor_primario" {
  provider      = aws.primario
  ami           = "ami-..."
  instance_type = "t2.micro"
  tags = {
    Name = "servidor-primario-va"
  }
}

# 2. Este recurso se crea en eu-west-1
resource "aws_instance" "servidor_secundario_dr" {
  provider      = aws.secundario
  ami           = "ami-..." # ¡Debe ser una AMI de eu-west-1!
  instance_type = "t2.micro"
  tags = {
    Name = "servidor-secundario-irlanda"
  }
}
```

### 3. Despliegue Híbrido (On-Premises y Cloud)

* **¿Qué es?** Una arquitectura que conecta su centro de datos privado (on-premises) con la nube de AWS.
* **¿Por qué?** Para migrar cargas de trabajo gradualmente (migración "lift-and-shift") o para mantener datos sensibles on-prem mientras se usa la nube para cómputo elástico.
* **¿Cómo se hace en Terraform?**
  * Terraform gestiona la conectividad: aws\_vpn\_connection (VPN Site-to-Site), aws\_dx\_connection (Direct Connect).
  * Terraform puede gestionar recursos on-prem usando otros providers (ej. provider "vsphere", provider "f5bigip").
  * Un solo terraform apply puede configurar firewall on-prem, la conexión VPN y la VPC en la nube en una sola operación.

### 4. Material de Apoyo



* [**Providers de Terraform (Terraform Registry)**](https://registry.terraform.io/browse/providers)
  * Explore esta página y busque "VMware", "Cisco", "F5" para ver cuántos proveedores de hardware on-prem existen.



***

## Módulo 14.2: Terraform con Contenedores (Amazon EKS)

### 1. Introducción: ¿Qué es EKS?

**Amazon EKS (Elastic Kubernetes Service)** es el servicio gestionado de AWS para ejecutar **Kubernetes**.

* **Kubernetes (K8s):** Plataforma de orquestación de contenedores.
* **EKS (El Servicio):** AWS gestiona el Plano de Control (Control Plane), usted gestiona los nodos de trabajo.

### 2. El Rol de Terraform: Infraestructura "Día 0"

Terraform aprovisiona la infraestructura base (Día 0) de un clúster de EKS. Un clúster EKS requiere múltiples recursos que deben estar perfectamente interconectados:

* aws\_eks\_cluster: El plano de control gestionado.
* aws\_iam\_role (para el Clúster): Rol que el clúster asume para gestionar recursos de AWS.
* aws\_iam\_role (para los Nodos): Rol que las instancias EC2 asumen.
* aws\_eks\_node\_group: Grupos de instancias EC2 (workers).
* VPC y Seguridad: VPC con subredes públicas/privadas y grupos de seguridad.

### 3. Buena Práctica: ¡Usar el Módulo Oficial!

La comunidad terraform-aws-modules mantiene un módulo de EKS probado y que sigue las mejores prácticas. No se recomienda escribir todos los recursos a mano.

#### Lab Conceptual: Desplegar EKS con el Módulo

{% stepper %}
{% step %}
### Obtener la VPC existente

Usamos data sources para referenciar una VPC y sus subredes existentes.

```terraform
# providers.tf
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}
provider "aws" { region = "us-east-1" }

# main.tf
data "aws_vpc" "vpc_existente" {
  tags = {
    Name = "vpc-principal"
  }
}

data "aws_subnets" "subredes_privadas" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.vpc_existente.id]
  }
  tags = {
    Name = "subnet-privada-app-*"
  }
}
```
{% endstep %}

{% step %}
### Llamar al módulo de EKS

Instanciamos el módulo oficial, pasando VPC y subnets.

```terraform
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "20.10.0" # ¡Fijar siempre la versión!

  cluster_name    = "mi-cluster-eks-prod"
  cluster_version = "1.29"

  vpc_id     = data.aws_vpc.vpc_existente.id
  subnet_ids = data.aws_subnets.subredes_privadas.ids

  eks_managed_node_groups = {
    grupo_principal = {
      instance_types = ["t3.medium"]
      min_size       = 1
      max_size       = 3
    }
  }

  tags = {
    Entorno = "prod"
  }
}
```
{% endstep %}

{% step %}
### Salidas

```terraform
output "eks_cluster_endpoint" {
  description = "El endpoint del API server de K8s"
  value       = module.eks.cluster_endpoint
}
```
{% endstep %}
{% endstepper %}

### 4. El Patrón GitOps: Terraform (Día 0) + ArgoCD (Día 1+)

* Día 0 (Terraform): Plataforma crea el clúster EKS vacío.
* Día 1+ (ArgoCD): Equipo de Aplicaciones gestiona Deployments/Services en un repo Git separado.
* Terraform puede instalar ArgoCD (por ejemplo via provider "helm") dentro del clúster.
* Resultado: ArgoCD sincroniza los YAMLs del repositorio al clúster; separa responsabilidades entre Plataforma y Aplicaciones.

### 5. Material de Apoyo

#### Documentos Clave

* [**Módulo de EKS de Terraform (Terraform Registry)**](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest)
* [**Guía: Terraform y ArgoCD (Manual Empresarial)**](https://terrateam.io/blog/argocd-terraform-integration-guide-for-end-to-end-gitops)



***

## Módulo 14.3: Infraestructura Serverless (Lambda y API Gateway)

### 1. ¿Qué es Serverless (Sin Servidor)?

Serverless es un modelo donde el proveedor de la nube (AWS) gestiona la infraestructura del servidor. Usted proporciona su código (Función Lambda) y AWS se encarga de ejecutarlo y escalarlo.

* FaaS (Function-as-a-Service): Modelo principal.
* Componentes clave: AWS Lambda y API Gateway.

### 2. Objetivo del Laboratorio (Conceptual)

Crear una API REST "Hola Mundo" totalmente serverless:

1. Un endpoint GET /hello.
2. Que invoque una función Lambda.
3. Que devuelva "¡Hola desde Lambda!".

### 3. Lab: Código de una API Serverless

```terraform
# providers.tf (necesita 'aws' y 'archive')
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
    archive = { source = "hashicorp/archive", version = "~> 2.0" }
  }
}
provider "aws" {
  region = "us-east-1"
}
```

Código de la función (archivo en carpeta lambda\_code/hello.js):

```javascript
// lambda_code/hello.js
exports.handler = async (event) => {
  return {
    statusCode: 200,
    body: JSON.stringify('¡Hola desde Lambda y Terraform!'),
  };
};
```

{% stepper %}
{% step %}
### Rol de IAM para la Lambda

```terraform
data "aws_iam_policy_document" "confianza_lambda" {
  statement {
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["lambda.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "rol_lambda" {
  name               = "rol-ejecucion-lambda-hello"
  assume_role_policy = data.aws_iam_policy_document.confianza_lambda.json
  # (Adjuntaríamos la política 'AWSLambdaBasicExecutionRole' para CloudWatch logs)
}
```
{% endstep %}

{% step %}
### Empaquetar y crear la Lambda

```terraform
data "archive_file" "lambda_zip" {
  type        = "zip"
  source_dir  = "lambda_code"
  output_path = "hello.zip"
}

resource "aws_lambda_function" "hello_lambda" {
  function_name     = "funcion-hello-world"
  filename          = data.archive_file.lambda_zip.output_path
  source_code_hash  = data.archive_file.lambda_zip.output_base64sha256
  role              = aws_iam_role.rol_lambda.arn
  handler           = "hello.handler"
  runtime           = "nodejs18.x"
}
```
{% endstep %}

{% step %}
### Crear API Gateway (v2 HTTP), integración, ruta y stage

```terraform
resource "aws_apigatewayv2_api" "api" {
  name          = "api-hello-world"
  protocol_type = "HTTP"
}

resource "aws_apigatewayv2_integration" "integracion_lambda" {
  api_id           = aws_apigatewayv2_api.api.id
  integration_type = "AWS_PROXY"
  integration_uri  = aws_lambda_function.hello_lambda.invoke_arn
}

resource "aws_apigatewayv2_route" "ruta_hello" {
  api_id   = aws_apigatewayv2_api.api.id
  route_key = "GET /hello"
  target    = "integrations/${aws_apigatewayv2_integration.integracion_lambda.id}"
}

resource "aws_apigatewayv2_stage" "prod" {
  api_id      = aws_apigatewayv2_api.api.id
  name        = "v1"
  auto_deploy = true
}
```
{% endstep %}

{% step %}
### Permiso y salida

```terraform
resource "aws_lambda_permission" "permiso_api" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.hello_lambda.function_name
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_apigatewayv2_api.api.execution_arn}/*/*"
}

output "api_endpoint" {
  value = aws_apigatewayv2_api.api.api_endpoint
}
```
{% endstep %}
{% endstepper %}

### 4. Ejecutar y Verificar

{% stepper %}
{% step %}
terraform init (Descargará archive).
{% endstep %}

{% step %}
terraform apply — escriba "yes".
{% endstep %}

{% step %}
Verificación:

* En Lambda verá funcion-hello-world.
* En API Gateway verá api-hello-world.
* Tome api\_endpoint de la salida y vaya a: \<api\_endpoint>/hello
* Debería ver: "¡Hola desde Lambda y Terraform!"
{% endstep %}
{% endstepper %}

***

## Módulo 14.4: Integración con Sistemas de Mensajería (SQS y SNS)

### 1. Introducción: Arquitecturas Desacopladas

Los servicios deben desacoplarse usando colas y tópicos: el productor envía mensajes, el consumidor los procesa cuando esté listo.

### 2. Amazon SQS (Simple Queue Service)

* **¿Qué es?** Una Cola.
* **Patrón:** Uno-a-Uno.
* **Uso:** Procesamiento de trabajos (job processing).
* **Recurso de Terraform:** aws\_sqs\_queue

```terraform
resource "aws_sqs_queue" "cola_de_trabajo" {
  name                       = "cola-procesamiento-imagenes.fifo"
  fifo_queue                 = true
  visibility_timeout_seconds = 300
  tags = {
    Entorno = "prod"
  }
}
```

### 3. Amazon SNS (Simple Notification Service)

* **¿Qué es?** Un Tópico.
* **Patrón:** Uno-a-Múltiples (Pub/Sub).
* **Uso:** Notificación de eventos.
* **Recursos de Terraform:** aws\_sns\_topic, aws\_sns\_topic\_subscription

```terraform
# 1. El Tópico
resource "aws_sns_topic" "pedidos_completados" {
  name = "topico-pedidos-completados"
  tags = {
    Entorno = "prod"
  }
}

# 2. Las Suscripciones

# Suscriptor 1: Una Lambda que envía facturas
resource "aws_sns_topic_subscription" "sub_facturacion" {
  topic_arn = aws_sns_topic.pedidos_completados.arn
  protocol  = "lambda"
  endpoint  = aws_lambda_function.facturacion.arn
}

# Suscriptor 2: Una SQS que alerta al almacén
resource "aws_sns_topic_subscription" "sub_almacen" {
  topic_arn = aws_sns_topic.pedidos_completados.arn
  protocol  = "sqs"
  endpoint  = aws_sqs_queue.cola_almacen.arn
}

# Suscriptor 3: Un email para el administrador
resource "aws_sns_topic_subscription" "sub_email" {
  topic_arn = aws_sns_topic.pedidos_completados.arn
  protocol  = "email"
  endpoint  = "admin@mi-empresa.com"
}
```





***

## Módulo 14.5: Data Lakes, Seguridad Avanzada y Aplicaciones Multicanal

### 1. Orquestación de Data Lakes

* **¿Qué es un Data Lake?** Estrategia para un repositorio centralizado de datos brutos y procesados.
* **Componentes y Recursos de Terraform:**
  1. Almacenamiento: aws\_s3\_bucket.
  2. Catálogo de Datos: aws\_glue\_catalog\_database, aws\_glue\_crawler.
  3. Seguridad: aws\_lakeformation\_permissions.
  4. Consulta: aws\_athena\_workgroup.

Terraform orquesta bucket, crawler y catálogo para que trabajen juntos.

### 2. Automatización de Seguridad Avanzada

* Infraestructura Inmutable: AMIs "Doradas" (Packer) y despliegue inmutable.
* Gobernanza como Código (GaC): aws\_organizations\_policy (SCPs), aws\_config\_config\_rule, aws\_securityhub\_account.
* Gestión de Secretos Avanzada: Integración con HashiCorp Vault.

### 3. Despliegue de Aplicaciones Multicanal

* **¿Qué es?** Comunicaciones por Email, SMS, Push.
* **Servicio de AWS:** Amazon Pinpoint.
* **Recursos de Terraform:** aws\_pinpoint\_app, aws\_pinpoint\_sms\_channel, aws\_pinpoint\_email\_channel, aws\_pinpoint\_apns\_channel.

