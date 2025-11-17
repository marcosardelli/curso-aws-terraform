# Módulo 2 - Fundamentos de AWS

## Módulo 2.1: Regiones y Zonas de Disponibilidad

### 1. ¿Qué es la Infraestructura Global de AWS?

Amazon Web Services opera una infraestructura de nube masiva y distribuida globalmente. Para el arquitecto de soluciones, esta infraestructura no es una "nube" abstracta, sino una red física de centros de datos.

La infraestructura global de AWS se compone de:

* **Regiones (Regions):** El concepto de más alto nivel.
* **Zonas de Disponibilidad (Availability Zones - AZs):** Los componentes de una Región.
* **Ubicaciones de Borde (Edge Locations):** Puntos de presencia para la entrega de contenido (se verán en el Módulo 12 con CloudFront).



### 2. Regiones (Regions)

* **¿Qué son?** Una Región es un área geográfica física e independiente en el mundo (ej. "Norte de Virginia", "Irlanda", "São Paulo", "Sídney").
* **¿De qué se componen?** Cada Región está compuesta por _múltiples_ Zonas de Disponibilidad (AZs), aisladas y físicamente separadas.
* **Características Clave:**
  * **Aislamiento:** Las Regiones están completamente aisladas entre sí. Una falla en la Región de Irlanda no afectará a la Región de Ohio.
  * **Soberanía de Datos:** Los datos que almacena en una Región (ej. eu-west-1 en Irlanda) _no_ salen de esa Región a menos que usted los mueva explícitamente. Fundamental para cumplir leyes como el GDPR.
  * **Latencia:** Usted elige la Región para minimizar latencia hacia sus usuarios.

### 3. Zonas de Disponibilidad (Availability Zones - AZs)

* **¿Qué son?** Una Zona de Disponibilidad (AZ) es uno o más centros de datos (data centers) discretos dentro de una Región.
* **Características Clave:**
  * **Aislamiento Físico:** Separadas por kilómetros, pero con baja latencia (<10ms).
  * **Recursos Independientes:** Energía, refrigeración y redes independientes.
  * **Tolerancia a Fallos:** Diseñadas para que un desastre que afecte a una AZ no afecte a las otras AZs de la misma Región.

### 4. El Diseño de Alta Disponibilidad (HA)

La arquitectura fundamental de AWS se basa en **no diseñar para el fallo, sino esperar el fallo.**

* **Anti-Patrón:** Desplegar una aplicación en una sola instancia EC2 en una sola AZ. Si esa instancia (o esa AZ) falla, su aplicación se cae.
* **Buena Práctica (HA):** Desplegar su aplicación en _múltiples_ instancias EC2, distribuidas en _al menos dos_ Zonas de Disponibilidad, con un Balanceador de Carga (ELB) al frente.

{% stepper %}
{% step %}
### Proceso de Alta Disponibilidad (resumen)

1. El usuario se conecta al Balanceador de Carga.
2. El ELB distribuye el tráfico a la Instancia 1 (en AZ-A) y a la Instancia 2 (en AZ-B).
3. **Falla la AZ-A:** El ELB lo detecta (mediante _health checks_) y deja de enviar tráfico a la Instancia 1.
4. **Resultado:** El 100% del tráfico se redirige automáticamente a la Instancia 2 (en AZ-B). Los usuarios pueden experimentar una ligera lentitud, pero **la aplicación sigue funcionando.**
{% endstep %}
{% endstepper %}

### 5. Material de Apoyo

#### Videos Recomendados

* [**Infraestructura Global de AWS (Video Oficial)**](https://www.youtube.com/watch?v=UuRX2gK0IYw)

#### Documentos Clave

* [**Infraestructura Global de AWS (Página Oficial)**](https://aws.amazon.com/es/about-aws/global-infrastructure/)
* [**Regiones y Zonas de Disponibilidad (Documentación)**](https://docs.aws.amazon.com/es_es/AWSEC2/latest/UserGuide/using-regions-availability-zones.html)

***

## Módulo 2.2: Cuentas y AWS Organizations

### 1. ¿Qué es una Cuenta de AWS?

Una **Cuenta de AWS** es el concepto fundamental de aislamiento.

* **¿Qué es?** Una cuenta es un "contenedor" para sus recursos de AWS. Cuando se registra en AWS, crea una cuenta.
* **Límite de Seguridad:** La cuenta es el límite de seguridad más fuerte que existe en AWS. Por defecto, ningún recurso o usuario de la Cuenta A puede ver o acceder a nada en la Cuenta B.
* **Límite de Facturación:** Cada cuenta genera su propia factura.

#### El Usuario Raíz (Root User)

* Cada cuenta de AWS tiene un **Usuario Raíz (Root User)** (email + contraseña usados al registrar la cuenta).
* **Tiene poder absoluto:** Permisos ilimitados (AdministratorAccess) que no se pueden restringir.
* **Buena Práctica:** **NO use el usuario Raíz para tareas diarias.** Su único propósito es:

{% stepper %}
{% step %}
### Configurar la cuenta raíz — pasos recomendados

1. Configurar la facturación.
2. Crear su primer usuario Administrador (un usuario IAM).
3. Configurar AWS Organizations.
{% endstep %}
{% endstepper %}

* **Acción de Seguridad Inmediata:** Siempre [habilite la autenticación multifactor (MFA)](https://www.google.com/search?q=https://docs.aws.amazon.com/es_es/IAM/latest/UserGuide/id_root-user.html%23root_user_mfa) en el usuario Raíz.

### 2. El Problema: Estrategia de Múltiples Cuentas

Historicamente, muchas empresas pusieron todo en una única cuenta (Dev, Test, Prod). Resultado: caos en seguridad, facturación y límites por cuenta.

### 3. La Solución: AWS Organizations

* **¿Qué es?** Servicio de gestión para **consolidar y gestionar múltiples cuentas de AWS** bajo una jerarquía centralizada.
* **¿Qué problema resuelve?** Gestión, seguridad, facturación y gobernanza entre múltiples cuentas.

{% stepper %}
{% step %}
### Características clave (resumen)

1. **Facturación Consolidada:**
   * Cuentas miembro enlazadas a una cuenta "Maestra" (Management Account).
   * Una sola factura; descuentos por volumen se aplican a toda la organización.
{% endstep %}

{% step %}
2. **Jerarquía (Unidades Organizativas - OUs):**
   * Agrupar cuentas en OUs anidadas que reflejen la estructura de la empresa.
   * Ejemplo:
     * Raíz
       * OU: Producción
         * Cuenta: App-Web-Prod
         * Cuenta: BaseDatos-Prod
       * OU: Desarrollo
         * Cuenta: App-Web-Dev
         * Cuenta: Sandbox-Juan
         * Cuenta: Sandbox-Maria
{% endstep %}

{% step %}
3. **Políticas de Control de Servicio (Service Control Policies - SCPs):**
   * Restricciones aplicadas a OUs o cuentas, que limitan servicios/acciones incluso si el usuario es admin en la cuenta.
   * Ejemplos: prohibir el uso de una Región, denegar creación de tipos de instancia no permitidos, etc.
{% endstep %}
{% endstepper %}

### 4. Material de Apoyo

#### Documentos Clave

* [**Mejores Prácticas de Seguridad para el Usuario Raíz (Documentación)**](https://docs.aws.amazon.com/es_es/IAM/latest/UserGuide/id_root-user.html)
* [**¿Qué es AWS Organizations? (Documentación)**](https://docs.aws.amazon.com/es_es/organizations/latest/userguide/orgs_introduction.html)
* [**Tutorial: Creación de una Jerarquía de OUs (AWS Well-Architected Labs)**](https://docs.aws.amazon.com/es_es/organizations/latest/userguide/orgs_tutorials_basic.html)

***

## Módulo 2.3: IAM: Usuarios, Roles y Políticas

### 1. ¿Qué es IAM (Identity and Access Management)?

IAM es el corazón de la seguridad en AWS. Servicio global que permite gestionar acceso: ¿Quién (Principal) puede hacer qué (Action) en qué recursos (Resource) y bajo qué condiciones (Condition)?

### 2. Los Componentes Clave de IAM

#### Principals (El "Quién")

* **Usuarios (Users):** Entidad que representa a una persona o aplicación con acceso a largo plazo. Credenciales: contraseña (consola) y Access Key/Secret Key (programático).
  * Buena práctica: evitar usuarios IAM para humanos si puede usar IAM Identity Center (SSO).
* **Roles:** Identidad sin credenciales de larga duración. Permisos que pueden ser "asumidos" temporalmente por un principal.
  * Ventaja: credenciales temporales (STS), evita almacenar claves.
  * Casos de uso (resumen en stepper):

{% stepper %}
{% step %}
1. Servicio -> Servicio: EC2 asume un Rol para escribir en S3 (mejor que poner claves en la instancia).
{% endstep %}

{% step %}
2. Cross-Account: Cuenta A asume un Rol en la Cuenta B para auditar recursos.
{% endstep %}

{% step %}
3. Federación (SSO): Usuario corporativo asume Rol en AWS para obtener acceso temporal.
{% endstep %}
{% endstepper %}

#### Políticas (Policies) — El "Qué"

* Documentos JSON que definen permisos. Ejemplo de estructura (se mantiene tal cual):

{ "Version": "2012-10-17", "Statement": \[ { "Sid": "AllowS3ReadAccess", "Effect": "Allow", "Action": \[ "s3:GetObject", "s3:ListBucket" ], "Resource": \[ "arn:aws:s3:::mi-bucket-de-ejemplo", "arn:aws:s3:::mi-bucket-de-ejemplo/\*" ] } ]}

* Componentes: Effect (Allow/Deny — Deny explícito siempre gana), Action, Resource (ARN), Condition (opcional).

### 3. Principio de Mínimo Privilegio

Otorgue solo los permisos mínimos necesarios. Anti-patrón: dar AdministratorAccess por defecto. Buena práctica: políticas personalizadas con permisos justos.

### 4. Material de Apoyo

* [**Documentación de IAM: ¿Cómo funciona?**](https://docs.aws.amazon.com/es_es/IAM/latest/UserGuide/introduction.html)
* [**Referencia de Acciones, Recursos y Condiciones de IAM**](https://docs.aws.amazon.com/es_es/service-authorization/latest/reference/service-authorization.pdf)

***

## Módulo 2.4: Cómputo: EC2, Auto Scaling y ELB

### 1. El Trío de Cómputo

* Amazon EC2 (servidores), Amazon ELB (balanceador de carga) y Auto Scaling Groups (gestor de capacidad) forman un sistema integrado.

### 2. Amazon EC2

* VMs en la nube. Conceptos: AMI, Instance Type (familias: t, m, c, r...), Security Group (firewall stateful), User Data (script que corre una vez al iniciar).

### 3. Amazon ELB

* Distribuye tráfico entre destinos (EC2). Alta disponibilidad y health checks. Si una instancia falla el health check, ELB deja de enviarle tráfico.

### 4. Auto Scaling Groups (ASG)

* Mantiene una colección de instancias EC2, asegura capacidad deseada y escala según políticas (dinámico o programado). Auto-reparación: reemplaza instancias fallidas.

### 5. Arquitectura Integrada (El Trío en Acción)

{% stepper %}
{% step %}
1. Un usuario accede a su sitio web. La petición DNS apunta al **ELB**.
{% endstep %}

{% step %}
2. El **ELB** (distribuido en AZ-A y AZ-B) recibe la petición.
{% endstep %}

{% step %}
3. El ELB realiza un chequeo de salud y ve que tiene 2 instancias saludables en el **ASG**.
{% endstep %}

{% step %}
4. El ELB envía la petición a una de las instancias EC2 (ej. en la AZ-A).
{% endstep %}

{% step %}
5. Llega el Black Friday. El tráfico se dispara.
{% endstep %}

{% step %}
6. CloudWatch detecta que la CPU promedio del **ASG** es del 85%.
{% endstep %}

{% step %}
7. CloudWatch dispara una alarma, que le dice al **ASG** que "escale horizontalmente" (Scale-Out).
{% endstep %}

{% step %}
8. El **ASG** lanza 2 nuevas instancias EC2 (aumentando la capacidad de 2 a 4).
{% endstep %}

{% step %}
9. El **ASG** le dice al **ELB**: "He lanzado 2 nuevas instancias, aquí están".
{% endstep %}

{% step %}
10. El **ELB** añade las nuevas instancias a su pool y empieza a enviarles tráfico.
{% endstep %}

{% step %}
11. **Resultado:** La aplicación maneja el pico de tráfico y el rendimiento se estabiliza.
{% endstep %}
{% endstepper %}

### 6. Material de Apoyo

* [**Tutorial: Configurar una aplicación escalada y con balanceo de carga (Documentación)**](https://docs.aws.amazon.com/es_es/autoscaling/ec2/userguide/autoscaling-load-balancer.html)
* [**Tipos de Instancia EC2 (Página Oficial)**](https://aws.amazon.com/es/ec2/instance-types/)

***

## Módulo 2.5: Almacenamiento: S3, EBS y EFS

### 1. El Problema

No todo el almacenamiento es igual: elegir mal = costo o mal rendimiento.

### 2. Almacenamiento de Bloques: Amazon EBS

* Volúmenes tipo disco (HDD/SSD), vinculados a 1 EC2 en la misma AZ. Persistentes; snapshots a S3. Uso: disco de sistema, bases de datos en una sola máquina.

### 3. Almacenamiento de Objetos: Amazon S3

* Buckets (nombres globales únicos), objetos (key). Acceso por HTTP/HTTPS. Durabilidad 99.999999999%. Uso: archivos estáticos, backups, data lakes, estado de Terraform.

### 4. Almacenamiento de Archivos: Amazon EFS

* Sistema de ficheros NFS compartido, escalable y multi-AZ. Múltiples EC2 pueden montarlo simultáneamente (Linux). Uso: uploads compartidos en apps en ASG, home dirs.

### 5. Tabla Comparativa

| Característica | Amazon EBS (Bloque)      | Amazon S3 (Objeto)                | Amazon EFS (Archivo)            |
| -------------- | ------------------------ | --------------------------------- | ------------------------------- |
| Analogía       | Disco duro virtual       | Almacenamiento infinito (objetos) | Disco de red (NAS)              |
| Unidad         | Volumen                  | Objeto (archivo)                  | Sistema de archivos             |
| Acceso         | Montado como disco local | API / HTTP                        | Montado via NFS                 |
| Conectividad   | 1 EC2 (misma AZ)         | Ilimitado                         | Miles de EC2 (Multi-AZ)         |
| Uso típico     | SO, DB de alto IOPS      | Backups, estáticos, Data Lake     | Contenido compartido, home dirs |

### 6. Material de Apoyo

* [**Almacenamiento en la Nube de AWS (Página de Servicios)**](https://aws.amazon.com/es/products/storage/)
* [**Comparativa Detallada: S3 vs EBS vs EFS (Artículo de terceros)**](https://www.logicata.com/blog/aws-storage-services/)

***

## Módulo 2.6: Redes con VPC, Subnets, Route Tables y Security Groups

### 1. ¿Qué es Amazon VPC?

* Su red lógicamente aislada en AWS. Control completo: rangos IP, subredes y seguridad.

### 2. Bloques CIDR

* Ejemplo: 10.0.0.0/16 (VPC). Subredes deben estar dentro del bloque CIDR de la VPC.

### 3. Componentes Clave

* **Subredes:** Cada subred reside en una AZ.
* **Internet Gateway (IGW):** Permite comunicación VPC ↔ Internet.
* **Route Tables:** Reglas para enrutar tráfico. **Subred pública** = tabla con ruta 0.0.0.0/0 → IGW.
* **NAT Gateway:** Permite que instancias privadas inicien conexiones a Internet sin exponerlas públicamente.

### 4. Firewalls: Security Groups vs. NACLs

* **Security Groups (SG):** A nivel de instancia. Stateful. Solo reglas Allow. Uso: firewall principal y granular.
* **Network ACLs (NACL):** A nivel de subred. Stateless. Allow y Deny. Evaluadas por orden numérico. Uso: opcional, listas negras o segunda capa.

Comparativa: la mayoría de arquitecturas se construyen solo con Security Groups; NACLs por defecto normalmente se dejan allowing all.

### 5. Material de Apoyo

* [**Security Groups vs. NACLs (video)**](https://www.google.com/search?q=https://www.youtube.com/watch?v%3DR-lA-GgPEXA)
* [**Documentación del Usuario de VPC**](https://docs.aws.amazon.com/es_es/vpc/latest/userguide/what-is-amazon-vpc.html)
* [**Ejemplo: VPC con Subredes Públicas y Privadas (Documentación)**](https://docs.aws.amazon.com/es_es/vpc/latest/userguide/VPC_Scenario2.html)

***

## Módulo 2.7: Monitoreo y Observabilidad con CloudWatch

### 1. ¿Qué es Amazon CloudWatch?

Servicio de monitoreo y observabilidad nativo de AWS: métricas, logs, alarmas y dashboards.

### 2. Pilares de CloudWatch

* **Metrics:** métricas estándar (AWS) y custom metrics. Retención 15 meses (con granularidad variable).
* **Logs:** centraliza registros con agentes; Log Groups, Log Streams, Log Insights (lenguaje de consulta).
* **Alarms:** Umbral + acción (ej. escalar ASG, publicar en SNS). Estados: OK / ALARM / INSUFFICIENT\_DATA.
* **Dashboards:** paneles personalizables.

### 3. Integración

* CloudWatch → ASG (alarma desencadena scale-out).
* CloudWatch → SNS para notificaciones (email, SMS, Slack).
* CloudTrail puede enviar logs a CloudWatch Logs.

### 4. Material de Apoyo

* [**Documentación de CloudWatch**](https://docs.aws.amazon.com/es_es/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html)
* [**Publicar Métricas Personalizadas**](https://docs.aws.amazon.com/es_es/AmazonCloudWatch/latest/monitoring/publishingMetrics.html)
* [**CloudWatch Log Insights**](https://docs.aws.amazon.com/es_es/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html)

***

## Módulo 2.8: Gestión de Costos con AWS Cost Explorer

### 1. Modelo de Precios

Pagar por uso: cómputo, almacenamiento, transferencia de datos. Riesgo: recursos olvidados o mal configurados.

### 2. ¿Qué es AWS Cost Explorer?

Herramienta para visualizar, entender y gestionar costos. Gratis, tarda 24h en poblar datos.

* Filtrado y agrupación por servicio, región, tipo de instancia, cuenta (Organizations), etiquetas (tags).
* Pronósticos (ML).
* Guardar informes.

### 3. Herramientas Relacionadas

* **AWS Budgets:** Alertas cuando se supera un presupuesto (recomendado activar).
* **Cost and Usage Report (CUR):** CSV detallado volcado a S3 para análisis avanzado.

### 4. Estrategias para Controlar Costos

1. Etiquetar todo (tags).
2. Usar AWS Budgets.
3. Revisar Cost Explorer regularmente.
4. Eliminar recursos no usados (EBS huérfanos, EIPs no asignadas).
5. Elegir servicio correcto (S3 vs EBS).
6. Auto Scaling (min size 0 para entornos dev si aplica).

### 5. Material de Apoyo

* [**AWS Cost Management**](https://aws.amazon.com/es/aws-cost-management/aws-cost-explorer/)
* [**Tutorial: Configurar un Presupuesto (AWS Budgets)**](https://docs.aws.amazon.com/es_es/cost-management/latest/userguide/budgets-create.html)

***

## Módulo 2.9: Casos de Uso Típicos de Arquitecturas en AWS

### 1. Poniendo los Bloques Juntos

Use el AWS Architecture Center para arquitecturas de referencia.

### 2. Caso de Uso 1: Patrón Canónico (Arquitectura Web de 3 Capas)

* VPC multi-AZ con subredes públicas y privadas.
* Capa web: ALB en subredes públicas (o S3 + CloudFront si estático).
* Capa app: ASG de EC2 en subredes privadas. SG que solo permite tráfico desde el ALB. NAT Gateway para salida.
* Capa datos: RDS Multi-AZ en subredes privadas; SG que permite solo tráfico desde la capa app.

### 3. Caso de Uso 2: Big Data / Data Lake

* S3 como repositorio central. Kinesis/Glue para ingesta. Athena/Redshift para análisis.

### 4. Caso de Uso 3: Serverless

* API Gateway → Lambda → DynamoDB. Paga por ejecución; no hay servidores persistentes.

### 5. Material de Apoyo

* [**AWS Architecture Center**](https://aws.amazon.com/es/architecture/)
* [**Whitepaper: Arquitectura de 3 Capas (WordPress best practices)**](https://d0.awsstatic.com/whitepapers/wordpress-best-practices-on-aws.pdf)

***

## Módulo 2.10: Buenas Prácticas Iniciales en AWS

### 1. AWS Well-Architected Framework

Metodología basada en 6 pilares: Operational Excellence, Security, Reliability, Performance Efficiency, Cost Optimization, Sustainability.

### 2. Lista de Tareas de Seguridad "Día 0"

#### Asegurar el Usuario Raíz: pasos recomendados

{% stepper %}
{% step %}
1. **Habilite MFA** en el usuario Raíz (app de autenticación o llave hardware).
{% endstep %}

{% step %}
2. **Guarde las credenciales del Root** en un gestor de contraseñas seguro.
{% endstep %}

{% step %}
3. **No use** el Root para operaciones diarias.
{% endstep %}
{% endstepper %}

#### Crear un Usuario Administrador IAM — pasos

{% stepper %}
{% step %}
1. Con el Root, vaya a IAM y cree un Usuario IAM (ej. admin-juan).
{% endstep %}

{% step %}
2. Adjunte la política gestionada **AdministratorAccess** y configure contraseña.
{% endstep %}

{% step %}
3. Habilite MFA para este usuario y cierre sesión del Root; use el usuario admin-juan.
{% endstep %}
{% endstepper %}

#### Configurar Facturación y Alertas

* Cree un presupuesto (ej. $20) en AWS Budgets y configure alertas (email) cuando supere % del presupuesto.

#### Habilitar AWS CloudTrail

* Cree un Trail que aplique a **todas las regiones** y almacene logs en un bucket S3 (retención a largo plazo).

#### (Opcional) AWS Organizations

* Para empresas, crear una Organización y usar cuentas separadas (Dev, Prod, etc.).

### 3. Material de Apoyo

* [**AWS Well-Architected Framework**](https://aws.amazon.com/es/architecture/well-architected/)
* [**Well-Architected Tool (consola)**](https://aws.amazon.com/es/well-architected-tool/)

***
