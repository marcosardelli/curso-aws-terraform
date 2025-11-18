# Módulo 1 - Introducción a AWS y Terraform

## Módulo 1.1: ¿Qué es AWS y su relevancia en la Nube Pública?

### Introducción

Bienvenido al inicio de tu viaje en la nube. En este primer módulo, estableceremos las bases de todo lo que vendrá. Antes de poder _automatizar_ la infraestructura, debemos entender qué es esta infraestructura, por qué es tan fundamental en la tecnología moderna y qué problemas vino a resolver.

Este documento cubre el concepto más básico pero más importante: **¿Qué es Amazon Web Services (AWS)?**

### 1. ¿Qué es Cloud Computing?

Antes de definir AWS, debemos definir "Cloud Computing" (Computación en la Nube).

En esencia, **Cloud Computing es la entrega bajo demanda de recursos de TI a través de Internet con precios de pago por uso.**

En lugar de comprar, poseer y mantener sus propios servidores físicos y centros de datos, las organizaciones pueden acceder a servicios tecnológicos, como potencia de cómputo, almacenamiento y bases de datos, según los necesiten, a través de un proveedor de nube como AWS.

#### Los Modelos de Servicio: IaaS, PaaS, SaaS

Todos los servicios en la nube se dividen en tres categorías principales. Comprender esto es crucial para saber qué herramientas usar.

**Analogía típica: El Restaurante de Pizza**

* **On-Premise (Tradicional):** Haces todo tú mismo. Cultivas los tomates, haces la masa, construyes el horno, cocinas la pizza y sirves. (Máximo control, máxima responsabilidad).
* **IaaS:** El proveedor te da la cocina (horno, gas, agua). Tú traes los ingredientes (masa, queso, salsa) y cocinas la pizza.
* **PaaS:** El proveedor te da la cocina y los ingredientes. Tú solo cocinas la pizza.
* **SaaS:** Pides una pizza. Te la entregan caliente y lista para comer. (Mínimo control, mínima responsabilidad).

**IaaS (Infrastructure as a Service)**

Es la categoría más básica y fundamental. El proveedor de nube le ofrece los bloques de construcción puros de infraestructura:

* **Qué incluye:** Servidores virtuales (VMs), redes (VPC), almacenamiento (discos duros) y balanceadores de carga.
* **Qué gestiona usted:** El sistema operativo, el middleware, el runtime (ej. Java, Python), los datos y la aplicación.
* **Ejemplos de AWS:** Amazon EC2 (servidores), Amazon VPC (redes), Amazon S3 (almacenamiento).
* **Cuándo usarlo:** Cuando necesita máximo control y flexibilidad sobre su arquitectura.

**PaaS (Platform as a Service)**

Este modelo elimina la necesidad de gestionar la infraestructura subyacente (hardware y sistemas operativos), permitiéndole centrarse únicamente en el despliegue y la gestión de sus aplicaciones.

* **Qué incluye:** El proveedor gestiona los servidores, el SO, los parches de seguridad y las redes.
* **Qué gestiona usted:** Solo su código (la aplicación) y sus datos.
* **Ejemplos de AWS:** AWS Elastic Beanstalk, Amazon RDS (Bases de Datos Relacionales).
* **Cuándo usarlo:** Cuando quiere desarrollar y desplegar aplicaciones rápidamente sin preocuparse por el mantenimiento de los servidores.

**SaaS (Software as a Service)**

Es un producto completo, gestionado y alojado por el proveedor, que se consume directamente.

* **Qué incluye:** Todo. El proveedor gestiona la aplicación completa, la infraestructura y el mantenimiento.
* **Qué gestiona usted:** Simplemente su cuenta y cómo usa el software.
* **Ejemplos:** Gmail, Salesforce, Dropbox, Office 365. (En AWS, servicios como Amazon Chime).
* **Cuándo usarlo:** Para software que resuelve una necesidad de negocio directa (email, CRM) sin necesidad de desarrollo.

**Nuestro curso se centrará principalmente en IaaS y PaaS**, ya que Terraform es la herramienta para construir y gestionar estos componentes.

### 2. Los Modelos de Despliegue de la Nube

"La Nube" no es un solo lugar. Es un modelo, y puede desplegarse de varias maneras:

#### Nube Pública (Public Cloud)

Definición: La infraestructura es propiedad del proveedor de nube (como AWS, Google Cloud, Microsoft Azure) y es compartida (de forma segura y aislada) por múltiples clientes (arrendatarios).\
Ventajas: Máxima escalabilidad, precios de pago por uso, sin mantenimiento de hardware.\
Desventajas: Menor control sobre la soberanía de los datos (dónde residen físicamente).

#### Nube Privada (Private Cloud)

Definición: Los recursos de computación son utilizados exclusivamente por una sola empresa u organización. Puede estar ubicada físicamente en el centro de datos on-premise de la organización o ser alojada por un tercero.\
Ventajas: Máximo control sobre la seguridad, cumplimiento y soberanía de los datos.\
Desventajas: Costos iniciales elevados (CapEx), responsabilidad total del mantenimiento, menor elasticidad.

#### Nube Híbrida (Hybrid Cloud)

Definición: Es la combinación de una nube pública y una nube privada, que permanecen como entidades únicas pero están unidas por tecnología que permite compartir datos y aplicaciones entre ellas.\
Este es el modelo empresarial más común. Las empresas mantienen sus datos más sensibles en su nube privada (on-premise) pero aprovechan la potencia de la nube pública para escalar aplicaciones, realizar análisis de Big Data o para recuperación de desastres.

### 3. La Relevancia de AWS: Las 6 Ventajas Clave

¿Por qué AWS se convirtió en el líder del mercado y por qué las empresas migran masivamente a la nube? La respuesta se encuentra en las **6 Ventajas del Cloud Computing** (según AWS):

1. **Cambiar el gasto de capital (CapEx) por gasto variable (OpEx)**

* **Tradicional (CapEx):** Comprar servidores caros por adelantado, adivinando la capacidad necesaria para 3-5 años.
* **Nube (OpEx):** Pagar solo por los recursos que se consumen, como una factura de electricidad. Esto libera capital y reduce el riesgo financiero.

2. **Beneficiarse de economías de escala masivas**

* AWS es el comprador de hardware más grande del mundo. Debido a esta escala, obtienen precios mucho más bajos en servidores y redes.
* Trasladan estos ahorros a los clientes, por lo que usar los servicios de AWS es casi siempre más barato que construirlo uno mismo.

3. **Dejar de adivinar la capacidad (Elasticidad)**

* En el modelo tradicional, si se compra poca capacidad, los clientes sufren (sitio caído). Si se compra demasiada, se desperdicia dinero.
* En la nube, se puede aprovisionar la capacidad exacta necesaria.
* **Elasticidad:** Es la capacidad de escalar recursos (hacia arriba o hacia abajo) _automáticamente_ en respuesta a la demanda. Por ejemplo, añadir más servidores durante el Black Friday y eliminarlos el lunes.

4. **Aumentar la velocidad y la agilidad**

* En un centro de datos tradicional, solicitar un nuevo servidor puede llevar semanas o meses.
* En AWS, se pueden aprovisionar cientos de servidores en minutos.
* Esta agilidad permite a los equipos de desarrollo experimentar, innovar y lanzar nuevas funcionalidades al mercado mucho más rápido.

5. **Dejar de gastar dinero en "trabajo pesado indiferenciado" (Undifferentiated Heavy Lifting)**

* "Trabajo pesado indiferenciado" es todo el esfuerzo que una empresa dedica a tareas que _no_ son su producto principal: gestionar el aire acondicionado del data center, reemplazar discos duros, aplicar parches de seguridad al hipervisor.
* AWS se encarga de todo esto, permitiendo que sus ingenieros se centren en lo que genera valor para el negocio: **crear mejores aplicaciones**.

6. **Globalizarse en minutos**

* ¿Quiere expandir su aplicación a Europa o Asia?
* En lugar de construir un nuevo centro de datos en ese continente, con AWS puede desplegar su infraestructura en una nueva región geográfica (como Frankfurt o Tokio) con unos pocos clics.

### 4. Material de Apoyo y Siguientes Pasos

#### Videos Recomendados

* [**¿Qué es AWS? (Video Oficial de AWS)**](https://www.youtube.com/watch?v=a9__D53WsUs) (Aprox. 2 minutos)
  * Un resumen visual y de alto nivel de qué es Amazon Web Services y su propuesta de valor.

#### Documentos Clave

* [**Las 6 Ventajas del Cloud Computing (Página oficial de AWS)**](https://aws.amazon.com/es/what-is-cloud-computing/)
  * Revise la fuente oficial de las 6 ventajas discutidas anteriormente.
* [**IaaS vs. PaaS vs. SaaS (Un artículo de Red Hat)**](https://www.redhat.com/es/topics/cloud-computing/iaas-vs-paas-vs-saas)
  * Un excelente artículo que profundiza en las diferencias entre los modelos de servicio, con ejemplos claros.
* [**AWS Cloud Adoption Framework (AWS CAF)**](https://aws.amazon.com/es/professional-services/CAF/)
  * **Lectura conceptual clave:** Este no es un servicio, sino una **metodología**. Es la guía que AWS proporciona a las empresas para ayudarles a migrar a la nube de forma estructurada.
  * El CAF organiza la migración en **6 Perspectivas** (grupos de trabajo): Negocio, Personas, Gobernanza, Plataforma, Seguridad, Operaciones.

***

## Módulo 1.2: Principales Servicios de AWS para Infraestructura

### Introducción

En el módulo anterior, definimos qué es AWS. Ahora, abriremos el capó y conoceremos los "bloques de construcción" fundamentales que utilizaremos durante todo el curso.

Piense en estos servicios como el "elenco de personajes" principal de su infraestructura. Cada uno tiene un rol especializado. En este módulo, los presentamos conceptualmente. En los módulos siguientes (4-9), aprenderá a desplegar y gestionar cada uno de estos utilizando Terraform.

### 1. Identidad (El "Quién"): IAM (Identity and Access Management)

**IAM es, posiblemente, el servicio más importante de AWS.** Es el pilar sobre el que se construye toda la seguridad.

* **¿Qué es?** IAM es un servicio global (no está atado a una región específica) que le permite gestionar el acceso a los servicios y recursos de AWS de forma segura.
* **¿Qué problema resuelve?** Por defecto, una nueva cuenta de AWS viene con un "usuario raíz" (Root User) que tiene control total. Usar este usuario para tareas diarias es extremadamente peligroso. IAM le permite crear "usuarios" y "roles" con permisos limitados (siguiendo el **Principio de Mínimo Privilegio**) para interactuar con la cuenta.

#### Componentes Clave de IAM:

* **Usuarios (Users):** Una entidad que representa a una persona o una aplicación que necesita interactuar con AWS. Los usuarios tienen credenciales (contraseña para la consola o claves de acceso para el CLI/API).
* **Grupos (Groups):** Una colección de usuarios. Es una forma de aplicar permisos a un conjunto de usuarios (ej. "Grupo de Desarrolladores", "Grupo de Finanzas").
* **Políticas (Policies):** Son el núcleo de IAM. Son documentos JSON que definen explícitamente los permisos (ej. Allow o Deny, la Action como s3:GetObject, y el Resource como arn:aws:s3:::my-bucket/\*).
* **Roles:** Una "identidad" temporal con permisos específicos que puede ser "asumida" por una entidad de confianza. Este es un concepto clave.
  * **Caso de uso 1:** Un servicio de AWS (como EC2) asume un rol para poder escribir en S3 (sin necesidad de almacenar claves de acceso en la instancia).
  * **Caso de uso 2:** Un usuario de otra cuenta de AWS asume un rol para acceder a recursos en su cuenta (acceso federado).

### 2. Cómputo (El "Cerebro"): EC2, ELB y ASG

Este es el trío de servicios que impulsa la mayoría de las aplicaciones web.

#### Amazon EC2 (Elastic Compute Cloud)

* **¿Qué es?** Es el servicio central de AWS. Proporciona capacidad de cómputo segura y de tamaño variable (servidores virtuales o Máquinas Virtuales) en la nube.
* **¿Qué problema resuelve?** Elimina la necesidad de comprar hardware físico. Le permite lanzar un servidor (Linux, Windows, o macOS) en minutos.
* **Conceptos Clave de EC2:**
  * **AMI (Amazon Machine Image):** La plantilla utilizada para lanzar una instancia. Define el sistema operativo, el software preinstalado y la configuración.
  * **Tipo de Instancia (Instance Type):** La "potencia" del servidor. Define la CPU, la memoria (RAM), el almacenamiento y la capacidad de red (ej. t2.micro, m5.large).
  * **Grupo de Seguridad (Security Group):** Un _firewall virtual_ a nivel de instancia que controla el tráfico entrante y saliente (ej. "permitir tráfico HTTP en el puerto 80").

#### Amazon ELB (Elastic Load Balancing)

* **¿Qué es?** Es un servicio que distribuye automáticamente el tráfico de red entrante (ej. peticiones web) entre múltiples instancias EC2.
* **¿Qué problema resuelve?**
  1. **Alta Disponibilidad (High Availability):** Si una instancia EC2 falla, el ELB deja de enviarle tráfico y lo redirige a las instancias saludables.
  2. **Escalabilidad (Scalability):** Permite escalar su aplicación horizontalmente (añadiendo más instancias EC2) sin que el usuario final note ningún cambio.
* **Tipos principales:**
  * **Application Load Balancer (ALB):** El más común. Opera en la Capa 7 (HTTP/HTTPS) y es "consciente" de la aplicación (puede enrutar tráfico basado en la URL, ej. /api va a un grupo de servidores y /images a otro).
  * **Network Load Balancer (NLB):** Ultra-alto rendimiento. Opera en la Capa 4 (TCP) y es ideal para picos de tráfico extremos o IPs estáticas.

#### Grupos de Auto Scaling (Auto Scaling Groups - ASG)

* **¿Qué es?** Un servicio que ajusta automáticamente el número de instancias EC2 (el "grupo") para satisfacer la demanda.
* **¿Qué problema resuelve?** La elasticidad.
  * **Scale-out (Escalado horizontal):** Lanza nuevas instancias EC2 automáticamente cuando la demanda sube (ej. la CPU promedio supera el 70%).
  * **Scale-in (Reducción):** Termina instancias EC2 automáticamente cuando la demanda baja (ej. por la noche) para ahorrar dinero.
* **Cómo funciona:** El ASG trabaja en conjunto con el ELB. Cuando el ASG lanza una nueva instancia, la registra automáticamente en el ELB para que empiece a recibir tráfico. Cuando la termina, la desregistra primero.

### 3. Almacenamiento (La "Memoria")

AWS ofrece diferentes tipos de almacenamiento para diferentes necesidades. Usar el incorrecto puede costar mucho dinero o arruinar el rendimiento.

#### Amazon S3 (Simple Storage Service)

* **¿Qué es?** Es un servicio de **almacenamiento de objetos**.
* **Piense en él como:** Un servicio de almacenamiento infinito para archivos. NO es un disco duro.
* **¿Qué problema resuelve?** Almacenar cantidades prácticamente ilimitadas de datos no estructurados (imágenes, videos, copias de seguridad, archivos de log, archivos estáticos de un sitio web).
* **Características Clave:**
  * **Objetos:** Los archivos se almacenan como "Objetos" (el archivo en sí) dentro de "Buckets" (contenedores con un nombre único global).
  * **Durabilidad Extrema:** Diseñado para una durabilidad de 99.999999999% ("once nueves"). Es casi imposible perder un archivo.
  * **Clases de Almacenamiento:** Puede mover archivos automáticamente a clases más baratas si no se accede a ellos con frecuencia (ej. S3 Standard -> S3 Glacier para archivo).

#### Amazon EBS (Elastic Block Store)

* **¿Qué es?** Es un servicio de **almacenamiento de bloques**.
* **Piense en él como:** Un disco duro (HDD o SSD) virtual de alto rendimiento que se conecta a _una única_ instancia EC2.
* **¿Qué problema resuelve?** Proporciona el volumen de arranque (disco del sistema operativo) y los discos de datos para sus instancias EC2.
* **Características Clave:**
  * **Ligado a 1 EC2:** Un volumen EBS solo puede estar conectado a una instancia EC2 a la vez (en la misma Zona de Disponibilidad).
  * **Persistente:** Los datos en un volumen EBS persisten independientemente de la vida de la instancia EC2 (si la instancia se detiene, el disco sigue allí).

#### Amazon EFS (Elastic File System)

* **¿Qué es?** Es un servicio de **almacenamiento de archivos** de red (NAS) totalmente gestionado (basado en NFS, solo para Linux).
* **Piense en él como:** Un disco de red compartido.
* **¿Qué problema resuelve?** Permitir que _múltiples_ instancias EC2 (potencialmente cientos) accedan y escriban en el mismo sistema de archivos _al mismo tiempo_.
* **Características Clave:**
  * **Compartido:** Ideal para directorios home compartidos, sistemas de gestión de contenido (CMS) o cualquier aplicación que necesite un sistema de archivos común.
  * **Elástico:** Crece y decrece automáticamente a medida que añade o elimina archivos. No es necesario aprovisionar el tamaño por adelantado.

### 4. Red : Amazon VPC (Virtual Private Cloud)

* **¿Qué es?** Es su propia sección de red lógicamente aislada dentro de la nube de AWS.
* **Piense en él como:** Su centro de datos virtual privado.
* **¿Qué problema resuelve?** Le da control total sobre su entorno de red. Es el contenedor de seguridad fundamental donde lanza todos sus recursos (como EC2 y RDS).

#### Componentes Clave de VPC:

* **Subredes (Subnets):** Son rangos de direcciones IP dentro de su VPC.
  * **Subred Pública:** Una subred que tiene una ruta a un "Internet Gateway" (IGW). Los recursos aquí (como un balanceador de carga) pueden ser accesibles desde Internet.
  * **Subred Privada:** Una subred que NO tiene una ruta directa a Internet. Los recursos aquí (como una base de datos) están seguros y aislados.
* **Tablas de Enrutamiento (Route Tables):** Un conjunto de reglas que determinan a dónde se dirige el tráfico de red.
* **NAT Gateway:** Un servicio de AWS que se coloca en la subred pública y permite que las instancias en la subred privada inicien conexiones hacia Internet (ej. para descargar parches de seguridad), pero impide que Internet inicie conexiones hacia ellas.

### 5. Monitoreo (Los "Sentidos"): Amazon CloudWatch

* **¿Qué es?** Es un servicio de monitoreo y observabilidad.
* **¿Qué problema resuelve?** Recopila datos de todos sus servicios de AWS para que pueda entender qué está sucediendo, por qué y cómo reaccionar.

#### Componentes Clave:

* **Métricas (Metrics):** La recopilación de datos de series temporales (ej. CPUUtilization de una EC2, RequestCount de un ELB).
* **Alarmas (Alarms):** Acciones automáticas que se disparan cuando una métrica cruza un umbral (ej. "Enviar una notificación si CPUUtilization > 80% durante 5 minutos").
* **Logs:** Un lugar centralizado para recopilar y analizar archivos de log de sus aplicaciones e instancias.

### 6. Material de Apoyo

#### Documentos Clave

* [**Centro de Arquitectura de AWS (AWS Architecture Center)**](https://aws.amazon.com/es/architecture/)
  * **Lectura fundamental.** Biblioteca de arquitecturas de referencia para ver cómo se combinan estos servicios en el mundo real (ej. "Arquitectura Web de 3 Capas").

***

## Módulo 1.3: Introducción al Concepto de Infrastructure as Code (IaC)

### 1. El Problema: "ClickOps" y la Deriva de Configuración

Antes de IaC, la forma de crear un servidor en la nube era a través de la consola web mediante clics. Este método, conocido despectivamente como **"ClickOps"**, funciona para infraestructuras con relativamente pocos recursos (vpc, subnets, servidores, databases), pero falla estrepitosamente a escala.

Problemas principales del enfoque manual:

* Es lento.
* Es propenso a errores.
* Es inconsistente entre entornos (dev, test, prod).
* No es escalable.
* No hay auditoría, facilitando la deriva de configuración (configuration drift).

Para ilustrar el flujo manual típico, usamos un stepper que refleja el proceso clásico de creación de una instancia:

{% stepper %}
{% step %}
#### Iniciar en la consola

1. Entrar a la consola de AWS.
2. Hacer clic en "Launch Instance".
{% endstep %}

{% step %}
#### Seleccionar configuración básica

3. Seleccionar una AMI del menú desplegable.
4. Seleccionar un tipo de instancia (haciendo clic).
{% endstep %}

{% step %}
#### Configurar red y seguridad

5. Configurar el Security Group (haciendo clic).
{% endstep %}

{% step %}
#### Lanzar

6. Hacer clic en "Launch".
{% endstep %}
{% endstepper %}

Esto demuestra por qué el método manual no escala y conduce a la deriva de la configuración.

### 2. La Solución: Infrastructure as Code (IaC)

**Infrastructure as Code (IaC)** es el proceso de gestionar y aprovisionar la infraestructura de cómputo a través de **archivos de definición legibles por máquina (código)**, en lugar de procesos manuales.

Beneficios clave:

* Repetibilidad y consistencia.
* Velocidad y automatización.
* Versionado y gestión de cambios (Git).
* Reducción de riesgos mediante planes de cambios previsibles.

### 3. Paradigmas de IaC: Declarativo vs. Imperativo

#### Imperativo (El "Cómo")

* Usted define cada paso que se debe ejecutar.
* Ejemplos: scripts Bash, Python (Boto3), Ansible en modo procedural.
* Desventaja: debe gestionar la lógica para comprobar y reconciliar estado.

#### Declarativo (El "Qué")

* Usted declara el estado final deseado; la herramienta decide cómo alcanzarlo.
* Ejemplos: **Terraform**, CloudFormation, Pulumi.
* Ventaja: las herramientas declarativas comparan el estado deseado con el estado real y realizan las acciones mínimas necesarias para reconciliar.

Terraform es una herramienta declarativa, por eso la potencia del curso se basa en ella.

### 4. Material de Apoyo y Siguientes Pasos

#### Documentos Clave

* [**¿Qué es IaC? (Documentación oficial de HashiCorp)**](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/infrastructure-as-code)

***

## Módulo 1.4: ¿Qué es Terraform y Ventajas frente a otros IaC?

### 1. ¿Qué es Terraform?

Terraform es una herramienta de Infraestructura como Código (IaC) de código abierto creada por HashiCorp. Usa HCL (HashiCorp Configuration Language) y sigue un flujo: Init → Plan → Apply ... ->Destroy.

Características principales:

* Agnóstico de proveedor (multi-cloud) mediante Providers.
* Declarativo (HCL).
* Gestión de estado (state file).

### 2. Ventajas y Comparativa de IaC

#### Terraform vs. AWS CloudFormation

* Terraform: agnóstico, HCL, state file explícito.
* CloudFormation: nativo AWS, JSON/YAML, state gestionado por AWS, soporte día 1 para nuevos servicios de AWS.

Veredicto estratégico:

* CloudFormation si está 100% en AWS y necesita soporte inmediato en servicios nuevos.
* Terraform si necesita flexibilidad multi-nube o prefiere HCL.

#### Terraform vs. Ansible

* Terraform: aprovisionamiento (Día 0), declarativo, con estado.
* Ansible: gestión de configuración (Día 1+), procedural/imperativo, sin estado por defecto.
* Patrón complementario: Terraform aprovisiona, Ansible configura el software dentro de las VMs.

#### Terraform vs. Pulumi / AWS CDK

* Terraform (HCL): DSL, enfocado a Ops.
* Pulumi/CDK: lenguajes de propósito general (TypeScript, Python, etc.), más orientado a Devs con mayor poder de abstracción.

### 3. Documentación oficial

* [**Introducción a Terraform (Página oficial de HashiCorp)**](https://www.terraform.io/intro)

***

## Módulo 1.5: Diferencia entre Terraform Open Source y Terraform Cloud/Enterprise

### 1. El Ecosistema de Terraform

#### Terraform Open Source (OSS) / CLI

* Binario terraform que ejecuta init, plan, apply, destroy.
* Usted es responsable de la ejecución, gestión de estado (backend remoto) y bloqueo de estado.

#### Contexto Estratégico: BSL y OpenTofu

* HashiCorp cambió la licencia de Terraform a BSL (agosto 2023).
* La comunidad creó **OpenTofu**, un fork bajo licencia MPL gobernado por la Linux Foundation.
* OpenTofu es un reemplazo binario "drop-in" para terraform.

Para el curso usaremos terraform, pero recuerde OpenTofu como referencia del futuro OSS.

### 2. Terraform Cloud (TFC) y Terraform Enterprise (TFE)

Características de TFC:

* Gestión de estado remota automática.
* Ejecución remota y gobernada (applies controlados).
* Registro privado de módulos.
* Policy as Code (Sentinel / OPA).
* Workspaces que conectan repositorios Git, variables y estado.

Resumen rápido:

<table><thead><tr><th>Característica</th><th width="240" align="right">Terraform OSS / OpenTofu</th><th>Terraform Cloud (TFC)</th></tr></thead><tbody><tr><td>Coste</td><td align="right">Gratuito</td><td>Freemium / Pago para funciones avanzadas</td></tr><tr><td>Gestión de Estado</td><td align="right">Manual (S3 + DynamoDB, por ejemplo)</td><td>Automática y segura</td></tr><tr><td>Ejecución</td><td align="right">Local o CI/CD propio</td><td>Remota y gobernada</td></tr><tr><td>Policy as Code</td><td align="right">No (requiere terceros)</td><td>Sí</td></tr><tr><td>Registro de Módulos</td><td align="right">No (usar Git)</td><td>Sí (privado)</td></tr></tbody></table>

### 3. Material de Apoyo

#### Documentos Clave

* [**OpenTofu (Página Oficial)**](https://opentofu.org/)
* [**Comparativa de Precios de Terraform Cloud**](https://www.hashicorp.com/products/terraform/pricing)
* [**Anuncio de la Licencia BSL de HashiCorp**](https://www.hashicorp.com/blog/hashicorp-updates-licensing-faq-based-on-community-questions)

***

## Módulo 1.6: Relación entre AWS y Terraform en Proyectos Reales

### 1. El Vínculo: El Provider de AWS

* Un Provider es un plugin que Terraform usa para gestionar recursos específicos (por ejemplo, hashicorp/aws).
* Ejemplo de declaración:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

Terraform init descarga el provider y permite al código HCL traducirse a llamadas API de AWS.

### 2. El Flujo de Trabajo en un Proyecto Real (GitOps)

Flujo estándar empresarial (resumen):

* PR: desarrollador cambia HCL en una rama y abre un **Pull Request**.
* CI/CD: ejecuta terraform init/validate/plan y publica el plan en el PR.
* Revisión humana: arquitecto revisa el plan (no solo el código).
* Merge: al fusionar, un pipeline seguro ejecuta terraform apply con los permisos necesarios aplicando el plan aprobado.

Este flujo evita ejecutar apply desde un portátil y crea trazabilidad auditada.

### 3. Casos de Uso Empresariales Reales

* Creación de Landing Zones codificadas.
* Migraciones masivas a AWS usando Terraform (ej. reducción del tiempo de configuración).
* Gestión a escala (ej. Mercado Libre con miles de instancias y despliegues diarios).

### 4. Documentación relacionada

* [**Documentación del Provider de AWS de Terraform**](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
* [**Tutorial de GitHub Actions para Terraform**](https://developer.hashicorp.com/terraform/tutorials/automation/github-actions)

***

## Módulo 1.7: Arquitectura de Terraform: Providers, Resources, State

### 1. La Arquitectura Central de Terraform

Tres componentes centrales:

1. Resources (El "Qué")
2. Providers (El "Traductor")
3. State (El "Mapa")

### 2. Resources (Recursos)

Ejemplo de sintaxis:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  tags = {
    Name = "HelloWorld"
  }
}
```

* resource: palabra clave.
* tipo\_de\_recurso: definido por el provider (ej. aws\_instance).
* nombre\_local: identificador dentro del HCL para referenciar el recurso.

### 3. Providers (Proveedores)

* Son binarios que implementan CRUD para recursos.
* Se declaran en terraform.required\_providers y se descargan con terraform init.

### 4. State (Estado)

* Archivo JSON (terraform.tfstate) que mapea el código al mundo real (IDs, atributos).
* Terraform lo usa para calcular el plan comparando el código, el estado local y la API real.
* Advertencia de seguridad: contiene datos sensibles en texto plano. Nunca hacer commit del .tfstate a Git.

Best practice: usar un Backend Remoto seguro (S3 + DynamoDB o Terraform Cloud).

### 5. Documentación relacionada

* [**Propósito del Estado de Terraform (Documentación Oficial)**](https://developer.hashicorp.com/terraform/language/state)

***

## Módulo 1.8: Configuración Inicial de Cuenta en AWS (para Terraform)

### 1. ¿Cómo se Autentica Terraform?

Terraform necesita credenciales para hablar con AWS. En el curso usaremos un **Usuario IAM con Claves de Acceso** (AdministratorAccess para lab), aunque en producción nunca se debe usar AdministratorAccess.

### 2. Creación del Usuario IAM (Lab 0)

Objetivo: crear un usuario programático y obtener Access Key ID y Secret Access Key.

Siga estos pasos como un stepper:

{% stepper %}
{% step %}
#### Acceder a IAM

1. Inicie sesión en la Consola de AWS con un usuario Root o con privilegios de administrador.
2. Vaya a IAM (busque "IAM").
{% endstep %}

{% step %}
#### Crear usuario

3. En el panel izquierdo haga clic en "Users".
4. Haga clic en "Create user".
{% endstep %}

{% step %}
#### Detalles del usuario

5. User name: p. ej. terraform-student.\
   IMPORTANT: NO marque "Provide user access to the AWS Management Console" (usuario programático).
{% endstep %}

{% step %}
#### Permisos

6. Seleccione "Attach policies directly".\
   Busque y marque **AdministratorAccess**. (Para el lab).
{% endstep %}

{% step %}
#### Revisar y crear

7. Revise y haga clic en "Create user".
{% endstep %}
{% endstepper %}

### 3. Guardar las Credenciales (Paso Crítico)

* Tras crear el usuario, abra su detalle y vaya a "Security credentials" → "Access keys".
* Cree una nueva Access Key para el caso de uso "CLI".
* Se mostrará **una sola vez** el Access Key ID y Secret Access Key. Copie ambos y guárdelos en un lugar seguro.

### 4. Métodos de Autenticación (Contexto Avanzado)

* IAM Identity Center (SSO) para humanos.
* Roles OIDC (CI/CD) para máquinas, evitando claves estáticas.

### 5. Documentación relacionada

* [**Creación de un Usuario IAM (Documentación de AWS)**](https://docs.aws.amazon.com/es_es/IAM/latest/UserGuide/id_users_create.html)
* [**Gestión de Claves de Acceso (Documentación de AWS)**](https://docs.aws.amazon.com/es_es/IAM/latest/UserGuide/id_credentials_access-keys.html)

***

## Módulo 1.9: Instalación y Configuración de Terraform y AWS CLI

### 1. El Software Requerido

Necesitamos:

1. AWS CLI (v2+)
2. Terraform CLI (v1.9+)
3. Un editor de código (Visual Studio Code recomendado) y la extensión oficial de Terraform.

### 2. Instalación (Paso a Paso)

#### 2.1. Instalar el AWS CLI (v2+)

* Windows: instalar MSI desde la guía oficial.
* macOS: brew install awscli
* Linux: seguir la guía oficial.\
  Verificación: aws --version

#### 2.2. Instalar el Terraform CLI (v1.9+)

* Descargar el ZIP desde la página oficial de Terraform, descomprimir y mover el binario a una carpeta en PATH (ej. /usr/local/bin o una carpeta en Windows en PATH).\
  Verificación: terraform version

#### 2.3. Instalar VS Code y la Extensión

* Instalar Visual Studio Code y la extensión oficial "HashiCorp Terraform".

### 3. Configuración: El "Puente" entre Terraform y AWS

Usaremos aws configure (AWS CLI) para guardar credenciales en \~/.aws/credentials, que el provider de AWS de Terraform buscará automáticamente.

Ejecute:

aws configure

Responda:

* AWS Access Key ID: (pegue su Access Key ID)
* AWS Secret Access Key: (pegue su Secret Access Key)
* Default region name: ej. us-east-1
* Default output format: json

### 4. Validación Final (El "Hola, Mundo" de la Configuración)

#### Validación 1: AWS CLI

Ejecute:

aws sts get-caller-identity

Debería recibir un JSON con UserId, Account y Arn.

#### Validación 2: Terraform

1. Cree una carpeta (ej. terraform-test) y un archivo main.tf con el siguiente contenido (lee datos, no crea recursos):

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

data "aws_caller_identity" "current" {}

output "mi_id_de_cuenta" {
  value = data.aws_caller_identity.current.account_id
}
```

2. En la carpeta, ejecute:

terraform init

3. Luego:

terraform apply -auto-approve

Resultado esperado:

Apply complete! Resources: 0 added, 0 changed, 0 destroyed.\
Outputs: mi\_id\_de\_cuenta = "123456789012"

Si ve esto, su entorno está configurado y listo.

### 5. Documentación relacionada

* [**Instalar el AWS CLI (Guía Oficial)**](https://docs.aws.amazon.com/es_es/cli/latest/userguide/getting-started-install.html)
* [**Instalar Terraform (Guía Oficial)**](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli)
* [**Configuración del AWS CLI (Guía Oficial)**](https://docs.aws.amazon.com/es_es/cli/latest/userguide/getting-started-quickstart.html)
