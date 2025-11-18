# Módulo 3 - Introducción a Terraform

## Módulo 3.1 - El Flujo de Trabajo de Terraform (init, plan, apply, destroy)

### 1. Introducción al Flujo de Trabajo Básico

Terraform se basa en un ciclo de vida simple, predecible y seguro. Casi todo lo que hagas con Terraform girará en torno a cuatro comandos fundamentales. Comprender este flujo de trabajo es la clave para dominar la herramienta.

El flujo de trabajo es: **Escribir -> init -> plan -> apply**.

### 2. Los 4 Comandos Esenciales

{% stepper %}
{% step %}
### terraform init (Inicializar)

* **¿Qué hace?** Este es _siempre_ el primer comando que debe ejecutar en un nuevo proyecto de Terraform (o después de añadir un nuevo provider).
* **Propósito:** Prepara el directorio de trabajo. Realiza dos tareas cruciales:
  1. **Descarga de Providers:** Lee su código HCL (específicamente el bloque terraform { required\_providers { ... } }), contacta el [Terraform Registry](https://registry.terraform.io/) y descarga los plugins necesarios (ej. hashicorp/aws) en un subdirectorio oculto llamado .terraform.
  2. **Inicialización del Backend:** Configura el "backend" donde se almacenará el [archivo de estado](https://developer.hashicorp.com/terraform/language/state). Por defecto, es un backend "local", pero en el Módulo 11 aprenderemos a configurarlo en S3.
* **Analogía:** Es el equipo de logística que llega al sitio de construcción, revisa los planos (el código) y descarga todos los materiales y herramientas especializadas (los providers) necesarios antes de que comience el trabajo.
* **¿Cuándo usarlo?**
  * Al iniciar un nuevo proyecto.
  * Al clonar un proyecto existente desde Git.
  * Cada vez que añada o cambie la versión de un provider.
{% endstep %}

{% step %}
### terraform plan (Planificar)

* **¿Qué hace?** Este es el comando más importante para la seguridad y la previsibilidad. **No realiza ningún cambio.**
* **Propósito:** Compara el código (el _estado deseado_) con el archivo de estado (el _estado conocido_) y con la infraestructura real (el _estado real_), y muestra exactamente lo que Terraform _hará_ si ejecutamos apply.
* **Interpretación del Resultado:** El plan mostrará un resumen con tres símbolos:
  *
    * (verde): **Crear (Create).** El recurso existe en el código pero no en el estado. Se creará.
  *
    * (rojo): **Destruir (Destroy).** El recurso existe en el estado pero no en el código. Se destruirá.
  * \~ (amarillo): **Modificar (Update in-place).** El recurso existe en ambos, pero un argumento (ej. una etiqueta) ha cambiado. Se actualizará.
* **Analogía:** Es el arquitecto jefe revisando los planos (su código) contra el edificio existente (estado real) y entregándole un informe detallado: "Vamos a construir un nuevo baño (crear), demoler la pared del garaje (destruir) y repintar la puerta de entrada (modificar)".
* **Buena Práctica:** **Siempre** ejecute terraform plan antes de apply. Hay que revisar su salida cuidadosamente. Es la  barrera de seguridad contra cambios accidentales o destructivos.
{% endstep %}

{% step %}
### terraform apply (Aplicar)

* **¿Qué hace?** Ejecuta el plan y realiza los cambios en el mundo real.
* **Propósito:** Conciliar el estado real de la infraestructura con el estado deseado en su código.
* **El Proceso:**
  1. Por defecto, terraform apply primero mostrará el _mismo plan_ que terraform plan.
  2. Luego, nos pedirá una **aprobación manual**. Deberemos escribir yes para continuar. (Esto es una última red de seguridad).
  3. Terraform ejecutará las llamadas API (usando los providers) para crear, modificar o destruir los recursos, respetando el orden de las dependencias.
  4. Una vez completado con éxito, **actualizará el archivo terraform.tfstate** con los nuevos atributos.
* **Analogía:** Es dar la orden al equipo de construcción: "Procedan. Ejecuten el plan que revisamos".
{% endstep %}

{% step %}
### terraform destroy (Destruir)

* **¿Qué hace?** Es lo contrario de apply. Lee su estado y destruye _toda_ la infraestructura gestionada por ese proyecto.
* **Propósito:** Limpiar los recursos de forma segura.
* **El Proceso:**
  1. Mostrará un plan (todo en rojo, con el símbolo -) de lo que va a destruir.
  2. Pedirá que escriba yes para confirmar.
* **Analogía:** Es la orden de demolición de todo el edificio. Es una acción muy destructiva, pero increíblemente útil en entornos de desarrollo y pruebas para limpiar y evitar costes.
{% endstep %}
{% endstepper %}

## Módulo 3.2: Anatomía de HCL: Archivos .tf, Recursos y Providers

### 1. ¿Qué es HCL?

Terraform utiliza un lenguaje de configuración propio llamado **HCL (HashiCorp Configuration Language)**. Está diseñado para ser legible por humanos y compatible con máquinas.

* **Características:**
  * **Declarativo:** Definimos el "qué", no el "cómo".
  * **Legible:** Utiliza bloques y argumentos clave = valor, que se ven limpios y son fáciles de entender.
  * **Compatible con JSON:** Terraform también puede leer archivos JSON, pero HCL es el formato preferido.

### 2. Estructura de un Archivo .tf

Todo el código HCL se escribe en archivos con la extensión .tf. Cuando ejecutamos terraform plan, Terraform lee _todos_ los archivos .tf del directorio actual y los trata como si fueran un solo documento.

Para mantener el orden, la **buena práctica** es dividir el código en archivos lógicos:

* **main.tf:** Aquí se colocan los recursos principales (la "lógica" de su stack).
* **providers.tf:** Aquí se definen los providers (ej. aws) y sus configuraciones (ej. region).
* **variables.tf:** Aquí se declaran las variables de entrada (parámetros).
* **outputs.tf:** Aquí se declaran las salidas (valores de retorno).
* **versions.tf:** Un sinónimo de providers.tf, a menudo usado para fijar la versión de Terraform y los providers.

### 3. La Sintaxis de HCL: Bloques, Argumentos y Comentarios

#### Bloques (Blocks)

Los bloques son los contenedores de HCL. Tienen un _tipo_ de bloque, pueden tener _etiquetas_ (labels) y un _cuerpo_ ({ ... }) que contiene argumentos.

Ejemplo 1: Un bloque de tipo "resource"

resource "aws\_instance" "web\_server" { ami = "ami-12345" # Un argumento instance\_type = "t2.micro" # Otro argumento }

Ejemplo 2: Un bloque de tipo "variable"

variable "aws\_region" { description = "Región de AWS para desplegar" type = string default = "us-east-1" }

#### Argumentos (Arguments)

Dentro de los bloques, se asignan valores a los argumentos usando la sintaxis clave = valor.

* **String (Cadena):** description = "Mi recurso"
* **Number (Número):** count = 3
* **Boolean (Booleano):** enabled = true
* **List (Lista):** subnet\_ids = \["subnet-1", "subnet-2"]
* **Map (Mapa):**

tags = { Name = "MiServidor" Entorno = "dev" }

#### Comentarios

* ## para comentarios de una sola línea.
* // también para comentarios de una sola línea.
* /\* ... \*/ para comentarios de múltiples líneas.

### 4. Los Bloques Fundamentales: terraform, provider y resource

#### Bloque terraform

Este bloque especial configura el propio Terraform.

* **Propósito:** Definir la versión de Terraform requerida y, lo más importante, los **providers** que necesita el proyecto.
* **Ejemplo (providers.tf):**

terraform {

#### Requiere que se use la versión 1.9.0 o superior

required\_version = ">= 1.9.0"

required\_providers { # Define que necesitamos el provider "aws" aws = { source = "hashicorp/aws" # Dónde encontrarlo (Terraform Registry) version = "\~> 5.0" # Fija la versión (muy recomendado) } } }

#### Bloque provider

Este bloque configura un provider específico (definido en el bloque terraform).

* **Propósito:** Configurar los detalles de autenticación o región para ese provider.
* **Ejemplo (providers.tf):**

#### Configura el provider "aws" que descargamos

provider "aws" { region = "us-east-1"

#### Nota: La autenticación (claves) se toma automáticamente del AWS CLI (aws configure)

}

#### Bloque resource

Este es el bloque principal. Describe un componente de infraestructura.

* **Propósito:** Definir un recurso que Terraform debe gestionar (crear, modificar, destruir).
* **Sintaxis:** resource "\<TIPO\_DE\_RECURSO>" "\<NOMBRE\_LOCAL>" { ... }
* **Ejemplo (main.tf):**

#### TIPO = "aws\_s3\_bucket"

#### NOMBRE LOCAL = "mi\_bucket\_de\_ejemplo"

resource "aws\_s3\_bucket" "mi\_bucket\_de\_ejemplo" {

#### Argumento

bucket = "mi-bucket-unico-123456789"

#### Argumento de tipo Mapa

tags = { Name = "MiBucket" Entorno = "dev" } }

* **"aws\_s3\_bucket":** Este es el **Tipo de Recurso**. El nombre lo define el provider de AWS.
* **"mi\_bucket\_de\_ejemplo":** Este es el **Nombre Local**. Lo inventamos nosotros. Se usa para _referenciar_ este recurso en otras partes del código (ej. aws\_s3\_bucket.mi\_bucket\_de\_ejemplo.id).

## Módulo 3.3: Variables: Parametrizando la Infraestructura

### 1. ¿Por qué Usar Variables?

En nuestro primer ejemplo, podríamos "hardcodear" (escribir directamente) el nombre del bucket S3 en el archivo main.tf:

#### main.tf (MAL EJEMPLO - Hardcodeado)

resource "aws\_s3\_bucket" "example" { bucket = "mi-bucket-unico-12345" }

**Problemas:**

1. **Reusabilidad Nula:** Si queremos desplegar este código en otro entorno (ej. "producción"), tendríamos que copiar y pegar el código y cambiar el nombre del bucket.
2. **Exposición de Secretos:** Si fuera una contraseña de base de datos, ¡estaría guardada en texto plano en Git!
3. **Difícil de Mantener:** Si el nombre se usa en 10 sitios, hay que cambiarlo 10 veces.

Las **Variables** son la solución. Permiten que su código sea modular, reutilizable y seguro. Son los "parámetros" de su plantilla de Terraform.

### 2. Declaración: variable

Las variables se declaran en un bloque variable, idealmente en el archivo variables.tf.

## variables.tf

variable "nombre\_del\_bucket" { description = "El nombre único para el bucket S3." type = string default = "mi-bucket-por-defecto-123" }

variable "entorno" { description = "Entorno (ej. dev, test, prod)" type = string

## Sin 'default', esta variable es OBLIGATORIA

}

variable "db\_password" { description = "Contraseña para la base de datos." type = string sensitive = true # ¡MUY IMPORTANTE! }

* **description:** Explica para qué sirve la variable. Esencial para la documentación.
* **type:** Define el tipo de dato. Ayuda a Terraform a validar la entrada. Los tipos comunes son: string, number, bool, list(string), map(string), object(...).
* **default:** Proporciona un valor por defecto. Si una variable tiene un default, se vuelve **opcional**. Si _no_ tiene default, es **obligatoria**.
* **sensitive = true:** Marca la variable como sensible (ej. contraseñas, claves API). Terraform ocultará su valor en las salidas de plan y apply para evitar que se filtre en los logs.

### 3. Uso: var.

Para usar el valor de una variable en su código HCL, se utiliza la sintaxis var.\<NOMBRE\_DE\_LA\_VARIABLE>.

## main.tf (BUEN EJEMPLO - Parametrizado)

resource "aws\_s3\_bucket" "example" {

## Usamos la variable en lugar de un valor hardcodeado

bucket = var.nombre\_del\_bucket tags = { Entorno = var.entorno } }

### 4. Asignación: ¿Cómo se da valor a las Variables?

Terraform carga las variables en un orden de precedencia específico (de menor a mayor). El último método gana.

1. (Prioridad más baja) default\
   Si la variable tiene un default en variables.tf, se usa ese valor.
2.  Archivos .tfvars\
    Puede crear un archivo llamado terraform.tfvars para definir sus valores.

    ## terraform.tfvars (Terraform carga este archivo automáticamente)

    nombre\_del\_bucket = "mi-bucket-de-desarrollo-98765" entorno = "dev"

    * **terraform.auto.tfvars:** También se cargan automáticamente y tienen prioridad sobre terraform.tfvars.
    * Archivos personalizados (-var-file): Puede tener archivos específicos (ej. dev.tfvars, prod.tfvars) y cargarlos manualmente: terraform apply -var-file="dev.tfvars"
3.  Argumentos de CLI (-var)\
    Puede pasar variables directamente en la línea de comandos. **Esto tiene alta prioridad.**

    terraform apply -var="nombre\_del\_bucket=bucket-desde-cli"
4.  Variables de Entorno (TF\_VAR\_)\
    Terraform leerá las variables de entorno de su sistema que tengan el prefijo TF\_VAR\_.

    ## En su terminal

    export TF\_VAR\_nombre\_del\_bucket="bucket-desde-variable-de-entorno" terraform apply
5.  Entrada Interactiva\
    Si una variable es **obligatoria** (no tiene default) y usted no le da un valor por ninguno de los métodos anteriores, Terraform **le preguntará interactivamente** en la terminal.

    $ terraform apply var.entorno Entorno (ej. dev, test, prod) Enter a value: dev

### 5. Material de Apoyo y Siguientes Pasos

#### Documentos Clave

* [**Documentación de Variables (Terraform Docs)**](https://developer.hashicorp.com/terraform/language/values/variables)
  * La guía oficial y completa sobre la declaración y uso de variables.
* [**Tipos de Variables (Constraints)**](https://developer.hashicorp.com/terraform/language/expressions/types)
  * Referencia de los tipos de datos (string, list, map, object).

## Módulo 3.4: Outputs y Locals: Referenciando y Organizando Datos

### 1. ¿Qué son los Outputs (Salidas)?

Mientras que las variables son los _parámetros de entrada_ de su código, los outputs son los _valores de retorno_.

* **Propósito:** Los Outputs (Salidas) se usan para exponer datos de su infraestructura al usuario de la CLI o para pasar datos a otros módulos de Terraform.
* **Ejemplo:** Después de crear una instancia EC2, quiere saber su dirección IP pública. O después de crear un bucket S3, quiere saber su _endpoint_ (URL).
* **Declaración:** Se declaran en un bloque output, idealmente en el archivo outputs.tf.

## outputs.tf

output "ip\_publica\_del\_servidor\_web" { description = "La dirección IP pública de la instancia EC2."

## 'value' es la expresión que se va a mostrar.

## Aquí referenciamos un atributo del recurso.

value = aws\_instance.web.public\_ip }

output "id\_del\_bucket\_s3" { description = "El ID (nombre) del bucket S3." value = aws\_s3\_bucket.example.id }

output "contraseña\_db\_secreta" { description = "La contraseña de la BBDD (generada aleatoriamente)." value = aws\_db\_instance.default.password sensitive = true # ¡Importante! }

* **Uso:** Cuando ejecute terraform apply, Terraform imprimirá estos valores al final.
* **sensitive = true:** Al igual que con las variables, si un output expone un dato sensible, márquelo como sensitive = true. Terraform lo ocultará en la salida (ej. contraseña\_db\_secreta = (sensitive)).
* **Comando terraform output:** Puede ver los outputs de su estado actual en cualquier momento ejecutando terraform output.

### 2. Referenciando Atributos de Recursos

La sintaxis clave usada en los outputs (y en todo Terraform) es la **expresión de referencia**:

\<TIPO\_DE\_RECURSO>.\<NOMBRE\_LOCAL>.

* **Ejemplo:** aws\_instance.web.public\_ip
  * aws\_instance: El Tipo de Recurso.
  * web: El Nombre Local que le dimos en main.tf.
  * public\_ip: El **Atributo**.

Cada recurso expone docenas de "Atributos". ¿Cómo saber cuáles están disponibles? **Consultando la documentación del provider.**

Si busca aws\_instance en la [documentación de AWS](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance), encontrará una sección llamada **"Attributes Reference"** que lista todo lo que puede referenciar (ej. id, arn, public\_ip, private\_ip, etc.).

### 3. ¿Qué son los Locals (Valores Locales)?

Los locals son un punto intermedio: son variables _internas_ que se definen y se usan _solo dentro_ del módulo o proyecto.

* **Propósito:** Ayudan a seguir el principio **DRY (Don't Repeat Yourself)**. Si tiene un valor o una expresión compleja que necesita usar en múltiples lugares, defínalo _una vez_ como un local.
* **Analogía:** Si las variables son los parámetros de una función y los outputs son los return, los locals son las variables que define _dentro_ de la función para hacerla más limpia.
* **Declaración:** Se definen en un solo bloque locals { ... }, usualmente en main.tf o variables.tf.

#### Ejemplo de locals

**El problema (WET):**

## main.tf (MAL EJEMPLO - Repetitivo)

variable "entorno" { default = "dev" }

resource "aws\_instance" "web" { tags = { Name = "servidor-web-dev" Entorno = "dev" } }

resource "aws\_s3\_bucket" "logs" { bucket = "logs-de-mi-app-dev" tags = { Entorno = "dev" } }

**La solución (DRY con locals):**

## main.tf (BUEN EJEMPLO - DRY con locals)

variable "entorno" { default = "dev" } variable "nombre\_proyecto" { default = "mi-app" }

## Definimos los valores locales UNA VEZ

locals {

## Concatenamos cadenas para formar nombres

nombre\_servidor\_web = "${var.nombre\_proyecto}-web-${var.entorno}" nombre\_bucket\_logs = "${var.nombre\_proyecto}-logs-${var.entorno}"

## Creamos un mapa de etiquetas común

common\_tags = { Proyecto = var.nombre\_proyecto Entorno = var.entorno } }

resource "aws\_instance" "web" {

## Usamos el local

tags = merge(local.common\_tags, { Name = local.nombre\_servidor\_web }) }

resource "aws\_s3\_bucket" "logs" {

## Usamos el local

bucket = local.nombre\_bucket\_logs

## Usamos el local

tags = local.common\_tags }

* **Uso:** Se referencian con local.\<NOMBRE\_LOCAL>.
* **Beneficio:** Ahora, si queremos cambiar la convención de nombres, solo la cambiamos en el bloque locals.

### 4. Material de Apoyo

#### Documentos Clave

* [**Documentación de Outputs (Terraform Docs)**](https://developer.hashicorp.com/terraform/language/values/outputs)
* [**Documentación de Locals (Terraform Docs)**](https://developer.hashicorp.com/terraform/language/values/locals)

## Módulo 3.5: El Estado: El "Cerebro" de Terraform (terraform.tfstate)

### 1. ¿Qué es el Estado de Terraform?

Como vimos en el Módulo 1, el **archivo de estado (state file)** es el componente más crítico de Terraform.

* **¿Qué es?** Es un archivo en formato **JSON** (por defecto terraform.tfstate) que Terraform crea y gestiona.
* **¿Qué hace?** Actúa como el "cerebro" o el "mapa" de Terraform.
* **Propósito Fundamental:** Mantiene un registro de la infraestructura que Terraform gestiona.
* **Contenido:**
  1. **Mapeo de Recursos:** Conecta su código (ej. resource "aws\_instance" "web") con el recurso real en la nube (ej. id: "i-12345abcdef").
  2. **Caché de Atributos:** Almacena una copia de todos los atributos del recurso (su IP, su ARN, etc.).
  3. **Dependencias:** Registra las dependencias entre recursos (ej. "la instancia i-123 depende de la subred subnet-abc").

### 2. ¿Por qué es tan Importante el Estado?

Sin el estado, Terraform no podría funcionar. Cuando usted ejecuta terraform plan, Terraform realiza una comparación de 3 vías:

1. **Código HCL (El Deseado):** Lo que usted _quiere_.
2. **Estado (.tfstate) (El Conocido):** Lo que Terraform _cree_ que existe.
3. **Realidad (API de AWS) (El Real):** Lo que _realmente_ existe.

El estado es la "fuente de verdad" de Terraform. Si la instancia i-123 existe en AWS, pero _no_ está en su archivo de estado, Terraform no sabe que la gestiona. Si la añade a su código HCL, Terraform intentará **crearla de nuevo** (y probablemente fallará con un conflicto de nombres).

### 3. El Grafo de Dependencias (DAG)

Terraform es inteligente. Si su código define una VPC, una Subred (que va _dentro_ de la VPC) y una Instancia (que va _dentro_ de la Subred), Terraform sabe que debe crearlos en el orden correcto.

resource "aws\_vpc" "main" { ... }

resource "aws\_subnet" "public" { vpc\_id = aws\_vpc.main.id # Dependencia explícita }

resource "aws\_instance" "web" { subnet\_id = aws\_subnet.public.id # Dependencia explícita }

* Terraform lee estas **dependencias explícitas** (aws\_vpc.main.id) y construye un **Grafo Acíclico Dirigido (DAG)**.
* Esto le permite paralelizar operaciones (ej. crear 5 Security Groups a la vez) y serializar otras (ej. debe crear la VPC _antes_ de la Subred, y la Subred _antes_ de la Instancia).
* El estado almacena los resultados de este grafo.

### 4. ¡PELIGRO! El Estado y la Seguridad

#### ¡ADVERTENCIA 1: Secreto en Texto Plano!

* El archivo terraform.tfstate es un archivo JSON en texto plano.
* Si usted define una contraseña en su HCL, esa contraseña se escribirá **en texto plano** en el archivo de estado.
* **Ejemplo:** resource "aws\_db\_instance" "default" { password = "MiContraseñaSecreta123" } _Su archivo .tfstate contendrá: "password": "MiContraseñaSecreta123"._

#### ¡ADVERTENCIA 2: El Estado es Frágil!

* Por defecto, el estado se almacena localmente (terraform.tfstate).
* **Problema 1 (Colaboración):** Si usted y un compañero trabajan en el mismo proyecto, cada uno tendrá una copia _local_ del estado. Si ambos ejecutan apply, ¡corromperán la infraestructura!
* **Problema 2 (Pérdida):** Si borra ese archivo o su portátil se rompe, Terraform pierde su "cerebro". No tiene idea de qué recursos gestiona, y no puede actualizarlos ni destruirlos.

### 5. La Solución (Buenas Prácticas)

#### 1. .gitignore (Mandatorio)

**NUNCA, JAMÁS, COMETA (COMMIT) ARCHIVOS DE ESTADO A GIT.**

Incluya al menos:

## Archivos de estado de Terraform

\*.tfstate \*.tfstate.backup

## Archivos de variables locales (pueden contener secretos)

\*.tfvars \*.auto.tfvars

## Directorio de plugins

.terraform/

## Archivos de plan

\*.tfplan

#### 2. Backend Remoto (Teaser del Módulo 11)

La solución profesional a los problemas de seguridad y colaboración es **no almacenar el estado localmente**.

* Terraform soporta **Backends Remotos**.
* Le diremos a Terraform que almacene el archivo de estado en un **Bucket S3** (cifrado) y que use una **Tabla de DynamoDB** para el bloqueo de estado (para evitar que dos personas ejecuten apply a la vez).
* Aprenderemos a configurar esto en el **Módulo 11**.

### 6. Material de Apoyo

#### Documentos Clave

* [**Propósito del Estado (Terraform Docs)**](https://developer.hashicorp.com/terraform/language/state/purpose)
  * **Lectura Obligatoria.** Explica por qué Terraform necesita el estado.
* [**Comandos de terraform state (CLI Docs)**](https://www.google.com/search?q=%5Bhttps://developer.hashicorp.com/terraform/cli/commands/state%5D\(https://developer.hashicorp.com/terraform/cli/commands/state\))
  * Documentación de comandos avanzados para manipular el estado (peligroso, pero útil), como terraform state list, terraform state show.

## Módulo 3.6: Lab Práctico: Creación del Primer Recurso en AWS

### 1. Objetivo del Laboratorio

Este es nuestro "Hola, Mundo" en AWS. El objetivo es combinar todo lo aprendido en los Módulos 1, 2 y 3.

* Verificar que nuestro entorno (AWS CLI, Terraform CLI) está configurado.
* Escribir una plantilla HCL básica con provider, resource, variable y output.
* Ejecutar el flujo de trabajo completo: init, plan, apply.
* Verificar el recurso en la Consola de AWS.
* Limpiar los recursos con destroy.

Vamos a crear el recurso de AWS más simple y seguro: un **Bucket S3**.

### 2. Prerrequisitos

* Haber completado el Módulo 1.8 (Creación de Usuario IAM).
* Haber completado el Módulo 1.9 (Instalación y aws configure).

### 3. Estructura de Archivos

1. Cree una nueva carpeta para este proyecto (ej. mkdir terraform-lab-s3).
2. Entre en esa carpeta (cd terraform-lab-s3).
3. Cree los siguientes archivos:

* providers.tf
* main.tf
* variables.tf
* outputs.tf

### 4. Lab: Pasos (Stepper)

{% stepper %}
{% step %}
#### Paso 1: Configurar el Provider (providers.tf)

Abra providers.tf y defina la versión de Terraform y el provider de AWS.

## providers.tf

terraform { required\_version = ">= 1.9.0"

required\_providers { aws = { source = "hashicorp/aws" version = "\~> 5.0" } } }

provider "aws" { region = "us-east-1" }
{% endstep %}

{% step %}
#### Paso 2: Declarar Variables (variables.tf)

Queremos que nuestro nombre de bucket sea único y parametrizado.

## variables.tf

variable "nombre\_del\_bucket" { description = "El nombre único global para el bucket S3." type = string }

variable "entorno" { description = "Entorno (ej. dev, test, prod)" type = string default = "dev" }
{% endstep %}

{% step %}
#### Paso 3: Definir el Recurso (main.tf)

Aquí es donde definimos _qué_ queremos construir.

## main.tf

resource "aws\_s3\_bucket" "mi\_primer\_bucket" {

## Referenciamos la variable para el nombre del bucket

bucket = var.nombre\_del\_bucket

tags = { Name = var.nombre\_del\_bucket Entorno = var.entorno } }
{% endstep %}

{% step %}
#### Paso 4: Definir las Salidas (outputs.tf)

Queremos que Terraform nos diga cuál es la URL de nuestro bucket.

## outputs.tf

output "bucket\_endpoint" { description = "El endpoint regional del bucket S3."

## Referenciamos un atributo del recurso

value = aws\_s3\_bucket.mi\_primer\_bucket.bucket\_regional\_domain\_name }
{% endstep %}

{% step %}
#### Paso 5: El Flujo de Trabajo de Terraform (La Terminal)

Abra su terminal en la carpeta terraform-lab-s3 y ejecute:

A. terraform init

$ terraform init

Salida esperada (ejemplo): Initializing the backend... Initializing provider plugins...

* Finding hashicorp/aws versions matching "\~> 5.0"...
* Installing hashicorp/aws v5.0.0...
* Installed hashicorp/aws v5.0.0 (signed by HashiCorp) Terraform has been successfully initialized!

Observe cómo ha creado una carpeta .terraform.

B. terraform plan

$ terraform plan

Terraform verá que la variable nombre\_del\_bucket es obligatoria y se la pedirá:

var.nombre\_del\_bucket El nombre único global para el bucket S3. Enter a value:

Escriba un nombre de bucket que sea globalmente único (ej: mi-bucket-juan-perez-14112025). Revise el plan (verá + para creación).

C. terraform apply

$ terraform apply

Le volverá a pedir la variable y luego la confirmación yes. Al finalizar verá:

Apply complete! Resources: 1 added, 0 changed, 0 destroyed. Outputs: bucket\_endpoint = "mi-bucket-juan-perez-14112025.s3.us-east-1.amazonaws.com"

¡Felicidades! Ha creado su primer recurso en AWS con Terraform.
{% endstep %}
{% endstepper %}

### 5. Verificación

1. Abra su Consola de AWS, vaya al servicio **S3**.
2. Verifique que su bucket existe y tiene las etiquetas correctas.
3. Abra la carpeta de su proyecto. Verá un nuevo archivo: terraform.tfstate. Ábralo en VS Code y vea cómo Terraform ha mapeado su código al recurso real.

### 6. Limpieza (terraform destroy)

Un buen arquitecto siempre limpia sus recursos de desarrollo.

$ terraform destroy

Le pedirá la variable otra vez y la confirmación yes. Ejemplo de salida al finalizar:

Destroy complete! Resources: 1 destroyed.

### 7. (Opcional) Mejora con .tfvars

¿Cansado de escribir el nombre del bucket cada vez? Use un archivo .tfvars.

1. Cree un archivo llamado terraform.tfvars.
2. Añada esto al archivo:

## terraform.tfvars

nombre\_del\_bucket = "mi-bucket-juan-perez-14112025"

3. Ahora ejecute terraform apply de nuevo. Terraform cargará automáticamente el valor desde el archivo y **no se lo preguntará**. Termine ejecutando terraform destroy.

## Módulo 3.7: Diferencias entre Módulos y Recursos

### 1. Revisión: ¿Qué es un Recurso?

Un **Recurso** (resource) es el bloque de construcción más básico de Terraform. Es un solo bloque resource que define un componente de infraestructura.

* **Analogía:** Es un **ladrillo**.
* **Ejemplo:** aws\_instance, aws\_s3\_bucket, aws\_vpc.

Ejemplo: resource "aws\_instance" "web" { ami = "..." instance\_type = "t2.micro" }

### 2. ¿Qué es un Módulo?

Un **Módulo** (module) es una colección de recursos (resource) que se agrupan para formar un componente lógico más grande.

* **Analogía:** Es una **pared prefabricada** (hecha de muchos ladrillos) que puede comprar y usar repetidamente.
* **Ejemplo:** Un módulo de "VPC" que _contiene_ los recursos para:
  * La aws\_vpc
  * 3 aws\_subnet públicas
  * 3 aws\_subnet privadas
  * Un aws\_internet\_gateway
  * Un aws\_nat\_gateway
  * Las aws\_route\_table necesarias

En lugar de escribir esos 10+ recursos cada vez, usted simplemente "llama" al módulo.

### 3. ¿Cómo se usa un Módulo?

Usted usa un bloque module para llamar a un módulo.

## main.tf

## Llamamos a un módulo "vpc" (podría ser uno que usted escribió o uno de Internet)

module "mi\_vpc\_de\_produccion" {

## "source" le dice a Terraform dónde encontrar el código del módulo

source = "./modulos/vpc" # Un directorio local

## Pasamos VARIABLES al módulo

nombre = "vpc-produccion" cidr\_block = "10.0.0.0/16" subredes\_publicas = \["10.0.1.0/24", "10.0.2.0/24"] }

## Usamos un OUTPUT del módulo para crear un servidor

resource "aws\_instance" "web" { ami = "..."

## Pedimos al módulo una de sus subredes públicas

subnet\_id = module.mi\_vpc\_de\_produccion.id\_subred\_publica\[0] }

### 4. Beneficios de Usar Módulos

1. **Organización:** Agrupan código complejo en "cajas negras" lógicas.
2. **Reusabilidad (DRY):** Defina su VPC "perfecta" una vez y reutilícela para los entornos de dev, test y prod.
3. **Abstracción:** El usuario del módulo no necesita saber _cómo_ construir una VPC.
4. **Consistencia:** Asegura estándares (seguridad, etiquetado) en la organización.

### 5. Material de Apoyo

#### Documentos Clave

* [**Introducción a Módulos (Terraform Docs)**](https://developer.hashicorp.com/terraform/language/modules)
  * La guía conceptual oficial.
* [**Registro de Módulos de Terraform**](https://registry.terraform.io/)
  * Explore el registro público. Busque el módulo oficial de "VPC" de AWS (terraform-aws-modules/vpc/aws) para ver un ejemplo de un módulo de nivel de producción.

## Módulo 3.8: Buenas Prácticas Iniciales en el Uso de Terraform

### 1. Escribir Código Limpio y Mantenible

Terraform es un lenguaje de código. Como tal, debemos aplicar las mismas disciplinas de ingeniería de software que usamos con Python o Java.

#### 1. Formateo: terraform fmt

* **¿Qué es?** Es un comando integrado que escanea su código HCL y lo reformatea automáticamente para seguir el estilo canónico de Terraform (sangría, espaciado, alineación).
* **¿Por qué?** Asegura que todo el código del equipo tenga el mismo estilo.
* **Buena Práctica:** Ejecute terraform fmt -recursive antes de cada git commit.

#### 2. Validación: terraform validate

* **¿Qué es?** Es un comando integrado que comprueba la sintaxis de su código.
* **¿Qué hace?** Comprueba que el HCL está bien escrito, que las referencias a variables son correctas, etc. **No** comprueba la lógica de AWS (ej. si su AMI existe).
* **Buena Práctica:** Ejecútelo en su pipeline de CI/CD (Módulo 12) como el primer paso.

#### 3. Nomenclatura (Naming Conventions)

* **Recursos y Variables:** Usar snake\_case (guion bajo).
  * Mal: variable "miVariable"
  * Bien: variable "mi\_variable"
  * Mal: resource "aws\_instance" "servidorWeb"
  * Bien: resource "aws\_instance" "servidor\_web"
* **Recursos (Nombre Local):** Sea descriptivo pero genérico. Use main o this si solo hay un recurso de ese tipo.
  * Bien: resource "aws\_vpc" "this" { ... }
* **Etiquetas (Tags) en AWS:** Usar PascalCase o kebab-case (depende del estándar del equipo). PascalCase (Name, Entorno) es muy común.

#### 4. Estructura de Archivos

* providers.tf (o versions.tf): Para terraform { ... } y provider { ... }.
* variables.tf: Para _todas_ las declaraciones de variable.
* outputs.tf: Para _todas_ las declaraciones de output.
* main.tf: Para los recursos (resource) y locales (locals).

#### 5. Fijar Versiones (Versioning)

* **El Problema:** Un día, el provider de AWS lanza una nueva versión que _rompe_ un argumento que usted usaba.
* **La Solución:** **Siempre** fije (pin) las versiones en su bloque terraform.
* **Operador \~> (Pessimistic):** Es el más recomendado.

terraform { required\_providers { aws = { source = "hashicorp/aws" # "~~> 5.0" significa: # Permitir v5.1, v5.2, v5.x (minor/patch) # NO permitir v6.0 (major) version = "~~> 5.0" } } }

#### 6. .gitignore (MANDATORIO)

Como se dijo en el Módulo 3.5, esto es una práctica de seguridad.

Incluya al menos:

## Archivos de estado de Terraform

\*.tfstate \*.tfstate.backup

## Archivos de variables locales (pueden contener secretos)

\*.tfvars \*.auto.tfvars

## Directorio de plugins y módulos

.terraform/

## Archivos de plan

\*.tfplan

## Logs de error

crash.log

### 2. Material de Apoyo



* [**Guía de Estilo de HashiCorp (Convenciones)**](https://developer.hashicorp.com/terraform/language/style)
  * Una guía de opinión sobre cómo nombrar y formatear.
