# Módulo 9 - MONITOREO Y LOGGING

## Módulo 9.1: Lab - CloudWatch Logs con Terraform

### 1. Introducción: La Observabilidad

El monitoreo (monitoring) es reactivo; le dice si algo está roto (ej. una alarma de CPU). La **Observabilidad (Observability)** es proactiva; le permite _entender_ por qué está roto.

La Observabilidad se basa en tres pilares:

1. **Métricas (Metrics):** Agregaciones numéricas (ej. CPUUtilization).
2. **Logs (Registros):** Eventos discretos de texto (ej. \[ERROR] User login failed).
3. **Trazas (Traces):** El ciclo de vida de una petición a través de múltiples servicios (Avanzado, ej. AWS X-Ray).

En este laboratorio, nos centramos en los **Logs**.

### 2. El Problema: Logs en Instancias Efímeras

En nuestro Módulo 6, creamos un ASG. Esas instancias EC2 son "ganado" (cattle): se crean y destruyen constantemente.

* Problema: Una instancia tiene un error, escribe el error en /var/log/app.log y el ASG la termina. El log se ha perdido para siempre.
* Solución: Centralizar los logs. Las instancias deben _transmitir_ (stream) sus logs a un servicio centralizado antes de morir. Ese servicio es **Amazon CloudWatch Logs**.

### 3. Componentes de CloudWatch Logs

* **Log Group (Grupo de Logs):** Contenedor principal para un conjunto de logs (ej. /aws/ec2/mi-aplicacion-web).
* **Log Stream (Flujo de Logs):** Secuencia de eventos de log de una fuente específica (ej. el log de la instancia i-123abc o el log de la instancia i-456def).

### 4. Objetivo del Laboratorio

1. Crear un Log Group (aws\_cloudwatch\_log\_group) para nuestra aplicación.
2. Crear una política de IAM que permita a nuestras instancias EC2 _escribir_ en ese Log Group.
3. Actualizar nuestro user\_data (del Módulo 6) para instalar el **Agente de CloudWatch** y configurarlo para que envíe los logs del sistema (ej. /var/log/messages) al Log Group.

### 5. Procedimiento (resumen)

{% stepper %}
{% step %}
### Paso 1: Crear el Log Group (main.tf)

Añadir en main.tf:

{% code title="main.tf (añadir)" %}
```hcl
# 1. Crear el Log Group
resource "aws_cloudwatch_log_group" "app_logs" {
  name              = "/aws/ec2/mi-aplicacion-web"
  # ¡Buena práctica! Por defecto, los logs se guardan para siempre.
  # Pongamos un límite para ahorrar costes.
  retention_in_days = 14 # Guardar logs por 14 días
  tags = {
    Name    = "log-group-app-web"
    Entorno = "dev"
  }
}
```
{% endcode %}
{% endstep %}

{% step %}
### Paso 2: Dar Permisos al Rol de EC2 (main.tf)

Adjuntar la política gestionada de AWS al Role de EC2 para que el agente tenga permisos necesarios:

{% code title="main.tf (modificar el rol de EC2)" %}
```hcl
resource "aws_iam_role" "rol_ec2_s3_readonly" {
  # ... (configuración existente)
}

# 2. Adjuntar la Política de CloudWatch Logs
resource "aws_iam_role_policy_attachment" "adjuntar_cloudwatch_al_rol_ec2" {
  role       = aws_iam_role.rol_ec2_s3_readonly.name
  # Esta política gestionada por AWS da los permisos necesarios para el Agente de CloudWatch
  policy_arn = "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy"
}
```
{% endcode %}
{% endstep %}

{% step %}
### Paso 3: Configurar el Agente (install\_nginx.sh)

Actualizar el script de arranque (user\_data) para que instale, configure e inicie el agente.

* Crear el archivo de configuración del agente (ej. /opt/aws/amazon-cloudwatch-agent/etc/config.json). Ejemplo de contenido (JSON):

{% code title="cloudwatch-agent-config.json (ejemplo)" %}
```
```
{% endcode %}

```json
{
  "agent": { "run_as_user": "root" },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/messages",
            "log_group_name": "/aws/ec2/mi-aplicacion-web",
            "log_stream_name": "{instance_id}/syslog"
          },
          {
            "file_path": "/var/log/nginx/access.log",
            "log_group_name": "/aws/ec2/mi-aplicacion-web",
            "log_stream_name": "{instance_id}/nginx-access"
          },
          {
            "file_path": "/var/log/nginx/error.log",
            "log_group_name": "/aws/ec2/mi-aplicacion-web",
            "log_stream_name": "{instance_id}/nginx-error"
          }
        ]
      }
    }
  }
}
```

* Nota: **{instance\_id}** es una variable mágica que el agente reemplazará por el ID real de la instancia.

Actualizar install\_nginx.sh para crear ese archivo y arrancar el agente. Ejemplo (fragmento):

{% code title="install_nginx.sh (fragmento)" %}
```bash
#!/bin/bash
# install_nginx.sh (MODIFICADO)
LOG_GROUP_NAME="/aws/ec2/mi-aplicacion-web"

# 1. Instalar Nginx y Agente de CloudWatch
yum update -yy
yum install -y nginx amazon-cloudwatch-agent

# 2. Crear el archivo de configuración del Agente
cat > /opt/aws/amazon-cloudwatch-agent/etc/config.json <<EOF
{
  "agent": { "run_as_user": "root" },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/messages",
            "log_group_name": "${LOG_GROUP_NAME}",
            "log_stream_name": "{instance_id}/syslog"
          },
          {
            "file_path": "/var/log/nginx/access.log",
            "log_group_name": "${LOG_GROUP_NAME}",
            "log_stream_name": "{instance_id}/nginx-access"
          }
        ]
      }
    }
  }
}
EOF

# 3. Iniciar el agente de CloudWatch
/opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/etc/config.json -s

# 4. Iniciar Nginx
systemctl start nginx
systemctl enable nginx
```
{% endcode %}

Importante: user\_data debe ser único; la forma correcta es que el user\_data cree el archivo de config en la instancia.
{% endstep %}

{% step %}
### Paso 4: Ejecutar y Verificar

1. Ejecutar: terraform apply (escriba yes). El ASG reemplazará las instancias con la nueva configuración.
2. Verificación en la consola de AWS:
   * CloudWatch -> Log Groups -> verá /aws/ec2/mi-aplicacion-web.
   * Dentro del grupo verá Log Streams con IDs de instancias (ej. i-0123abc.../syslog).
   * Abrir un stream para ver los logs transmitidos.
3. terraform destroy al finalizar.
{% endstep %}
{% endstepper %}

{% hint style="info" %}
Punto clave: El **Agente de CloudWatch** es la herramienta estándar para enviar logs (y métricas personalizadas) desde EC2 a CloudWatch. La configuración del agente es un JSON; lo más robusto es mantener el config.json en el repositorio y usar templatefile() para inyectar variables y user\_data para colocar el archivo en la instancia e iniciar el agente.
{% endhint %}

***

## Módulo 9.2: Lab - Creación de Métricas Personalizadas y Alarmas

### 1. Introducción

CloudWatch ofrece métricas estándar (CPUUtilization, NetworkIn), pero para métricas de negocio usamos **Métricas Personalizadas (Custom Metrics)**. El Agente de CloudWatch puede enviar métricas (p. ej. uso de memoria).

### 2. Objetivo

1. Ampliar la configuración del Agente para que envíe métricas de uso de memoria (RAM).
2. Crear una Alarma de CloudWatch (aws\_cloudwatch\_metric\_alarm) que vigile la métrica personalizada.

### 3. Procedimiento

{% stepper %}
{% step %}
### Paso 1: Actualizar la Configuración del Agente

Añadir la sección metrics al config.json. Ejemplo (fragmento de install\_nginx.sh que crea el config):

{% code title="install_nginx.sh (config con métricas)" %}
```bash
cat > /opt/aws/amazon-cloudwatch-agent/etc/config.json <<EOF
{
  "agent": { "run_as_user": "root" },
  "metrics": {
    "namespace": "MiAplicacion",
    "metrics_collected": {
      "mem": {
        "measurement": [ "mem_used_percent" ],
        "metrics_collection_interval": 60
      }
    },
    "append_dimensions": {
      "InstanceId": "\${aws:InstanceId}"
    }
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/nginx/access.log",
            "log_group_name": "${LOG_GROUP_NAME}",
            "log_stream_name": "{instance_id}/nginx-access"
          }
        ]
      }
    }
  }
}
EOF
```
{% endcode %}

Explicación breve:

* "namespace": "MiAplicacion" → namespace personalizado para la métrica.
* "mem\_used\_percent" → captura el porcentaje de RAM usada.
* "append\_dimensions" → agrega dimensiones como InstanceId.
{% endstep %}

{% step %}
### Paso 2: Crear la Alarma de CloudWatch (main.tf)

Ejemplo de recurso Terraform:

{% code title="main.tf (alarma de memoria)" %}
```
```
{% endcode %}

```hcl
resource "aws_cloudwatch_metric_alarm" "alarma_memoria" {
  alarm_name          = "alarma-alta-memoria-ram"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "mem_used_percent"
  namespace           = "MiAplicacion"
  period              = "300"
  statistic           = "Average"
  threshold           = "85"
  alarm_description   = "Alarma cuando la RAM de la aplicación supera el 85%"
  alarm_actions       = [aws_sns_topic.alertas_db.arn]
  ok_actions          = [aws_sns_topic.alertas_db.arn]
}
```

Nota: el nombre de la métrica y el namespace deben coincidir exactamente con el config.json.
{% endstep %}

{% step %}
### Paso 3: Ejecutar y Verificar

1. terraform apply (sí).
2. Verificación en consola:
   * CloudWatch -> Métricas -> Todas las Métricas -> verá el namespace "MiAplicacion".
   * Gráficar mem\_used\_percent por InstanceId.
   * CloudWatch -> Alarmas -> verá alarma-alta-memoria-ram (probablemente "Insufficient Data" mientras recopila).
3. terraform destroy cuando termine.
{% endstep %}
{% endstepper %}

{% hint style="info" %}
Punto clave: Enviar métricas directamente desde el agente es más eficiente y barato que extraer métricas de logs. El agente puede integrarse con StatsD, collectd o endpoints Prometheus.
{% endhint %}

***

## Módulo 9.3: Lab - Integración con SNS para Notificaciones

### 1. Introducción

SNS (Simple Notification Service) permite que las alarmas publiquen mensajes a un tópico y que suscriptores (ej. email) los reciban. Flujo: CloudWatch (Alarma) -> Tópico SNS -> Suscripción (Email) -> Bandeja de entrada.

### 2. Objetivo

Crear el Tópico SNS y la Suscripción por email para que nuestras alarmas realmente nos notifiquen.

### 3. Procedimiento

{% stepper %}
{% step %}
### Paso 1: Crear el Tópico SNS (main.tf)

Ejemplo:

{% code title="main.tf (sns topic)" %}
```hcl
resource "aws_sns_topic" "alertas_generales" {
  name = "alertas-app-produccion"
  tags = {
    Name = "sns-alertas-produccion"
  }
}
```
{% endcode %}
{% endstep %}

{% step %}
### Paso 2: Crear la Suscripción por Email (variables.tf + main.tf)

Añadir variable para el email (cambiar el default):

{% code title="variables.tf" %}
```hcl
variable "email_de_alerta" {
  description = "El email que recibirá las alertas de producción"
  type        = string
  default     = "su-email@ejemplo.com" # ¡Cambiar esto!
}
```
{% endcode %}

Crear la suscripción:

{% code title="main.tf (suscripción email)" %}
```hcl
resource "aws_sns_topic_subscription" "suscripcion_email" {
  topic_arn = aws_sns_topic.alertas_generales.arn
  protocol  = "email"
  endpoint  = var.email_de_alerta
}
```
{% endcode %}
{% endstep %}

{% step %}
### Paso 3: Actualizar las Alarmas para usar el Tópico

Modificar alarm\_actions / ok\_actions en las alarmas para que apunten a: aws\_sns\_topic.alertas\_generales.arn

Ejemplo (alarma\_memoria):

{% code title="main.tf (alarma actualizada)" %}
```hcl
alarm_actions = [aws_sns_topic.alertas_generales.arn]
ok_actions    = [aws_sns_topic.alertas_generales.arn]
```
{% endcode %}
{% endstep %}

{% step %}
### Paso 4: Ejecutar y Verificar (acción manual requerida)

1. terraform apply (sí).
2. En la bandeja de entrada del email configurado, recibirá un correo de "AWS Notification - Subscription Confirmation". Debe hacer clic en "Confirm subscription". Si no confirma, la suscripción quedará en estado "Pending".
3. Consola: SNS -> Topics -> alertas-app-produccion -> pestaña Suscripciones -> estado "Confirmed".
4. terraform destroy para limpiar.
{% endstep %}
{% endstepper %}

***

## Módulo 9.4: AWS CloudTrail y Exportación de Logs a S3

> Resumen conceptual (ya tratado en detalle en Módulo 4.7).

### 1. Introducción: ¿Logs de App vs. Logs de Auditoría?

* CloudWatch Logs: logs de aplicación y sistema (¿Qué hace mi Nginx?).
* CloudTrail: logs de auditoría de API de AWS (¿Quién eliminó mi EC2?).

### 2. aws\_cloudtrail

El recurso aws\_cloudtrail crea un Trail que captura acciones de la API y las envía a un destino (habitualmente S3).

### 3. ¿Por qué Exportar Logs a S3?

CloudTrail por defecto guarda \~90 días; para cumplimiento es insuficiente. Buenas prácticas:

1. Retención a largo plazo con lifecycle rules y Glacier Deep Archive.
2. Inmutabilidad con S3 Object Lock.
3. Análisis con Amazon Athena (consultas SQL sobre JSON).

### 4. Patrón Terraform (resumen)

1. aws\_s3\_bucket para logs.
2. data "aws\_iam\_policy\_document" para política que permita s3:PutObject por cloudtrail.amazonaws.com.
3. aws\_s3\_bucket\_policy adjunta.
4. aws\_cloudtrail apuntando a s3\_bucket\_name.

### 5. Material de Apoyo

* https://docs.aws.amazon.com/athena/latest/ug/cloudtrail-logs.html

***

## Módulo 9.5: Lab - Configuración de Dashboards en CloudWatch

### 1. Introducción

Un Dashboard de CloudWatch unifica métricas, logs y alarmas en una sola pantalla.

### 2. Objetivo

Crear aws\_cloudwatch\_dashboard que muestre:

* CPUUtilization del ASG.
* DatabaseConnections del RDS.
* Estado de alarma\_memoria (opcional).

### 3. Procedimiento

{% stepper %}
{% step %}
### Paso 1: Crear el cuerpo del Dashboard (JSON)

Recomiendo crear el dashboard en la consola, arrastrar widgets y copiar el JSON.

Ejemplo de archivo dashboard-body.json (usar templatefile() para inyectar variables):

{% code title="dashboard-body.json (ejemplo)" %}
```json
{
  "widgets": [
    {
      "type": "metric",
      "x": 0,
      "y": 0,
      "width": 12,
      "height": 6,
      "properties": {
        "metrics": [
          [ "AWS/EC2", "CPUUtilization", "AutoScalingGroupName", "${asg_name}" ]
        ],
        "period": 300,
        "stat": "Average",
        "region": "${aws_region}",
        "title": "CPU del ASG de la Aplicación"
      }
    },
    {
      "type": "metric",
      "x": 0,
      "y": 7,
      "width": 12,
      "height": 6,
      "properties": {
        "metrics": [
          [ "AWS/RDS", "DatabaseConnections", "DBInstanceIdentifier", "${db_instance_id}" ]
        ],
        "period": 300,
        "stat": "Average",
        "region": "${aws_region}",
        "title": "Conexiones a la Base de Datos"
      }
    }
    /* Puede añadirse un widget para métricas personalizadas si lo desea */
  ]
}
```
{% endcode %}
{% endstep %}

{% step %}
### Paso 2: Crear el recurso en Terraform

Ejemplo:

{% code title="main.tf (dashboard)" %}
```hcl
resource "aws_cloudwatch_dashboard" "dashboard_principal" {
  dashboard_name = "Dashboard-Principal-${var.entorno}"
  dashboard_body = templatefile("dashboard-body.json", {
    aws_region    = var.aws_region
    asg_name      = aws_autoscaling_group.asg_app_web.name
    db_instance_id = aws_db_instance.db_principal.id
  })
}
```
{% endcode %}

Nota: Si no puede rellenar el widget de métricas personalizadas dinámicamente, puede eliminarlo del JSON.
{% endstep %}

{% step %}
### Paso 3: Ejecutar y Verificar

1. terraform init (por templatefile).
2. terraform apply (sí).
3. Console: CloudWatch -> Dashboards -> verá Dashboard-Principal-dev con los gráficos.
4. terraform destroy para limpiar.
{% endstep %}
{% endstepper %}

***

## Módulo 9.6: Lab - Monitoreo de Costes con AWS Budgets

### 1. Introducción

AWS Budgets permite alertas sobre costes; útil para controlar facturas.

### 2. Objetivo

Crear aws\_budgets\_budget que envíe una alerta a un Tópico SNS si el coste mensual supera un umbral.

### 3. Procedimiento

{% stepper %}
{% step %}
### Paso 1: Configurar el Budget (main.tf)

Ejemplo:

{% code title="main.tf (budget)" %}
```hcl
resource "aws_budgets_budget" "presupuesto_mensual" {
  name        = "presupuesto-mensual-general"
  budget_type = "COST"
  limit_amount = "20.0"
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  notification {
    notification_type = "ACTUAL"
    comparison_operator = "GREATER_THAN"
    threshold = 80.0
    threshold_type = "PERCENTAGE"
    subscriber_sns_topic_arns = [aws_sns_topic.alertas_generales.arn]
  }
}
```
{% endcode %}
{% endstep %}

{% step %}
### Paso 2: Permisos para SNS (opcional pero recomendado)

Los budgets necesitan permiso para publicar en el tópico. Añadir una política de tópico:

{% code title="main.tf (sns policy para budgets)" %}
```hcl
data "aws_iam_policy_document" "politica_sns_budgets" {
  statement {
    effect = "Allow"
    actions = ["SNS:Publish"]
    principals {
      type = "Service"
      identifiers = ["budgets.amazonaws.com"]
    }
    resources = [aws_sns_topic.alertas_generales.arn]
  }
}

resource "aws_sns_topic_policy" "sns_budgets_policy" {
  arn    = aws_sns_topic.alertas_generales.arn
  policy = data.aws_iam_policy_document.politica_sns_budgets.json
}
```
{% endcode %}
{% endstep %}

{% step %}
### Paso 3: Ejecutar y Verificar

1. terraform apply (sí).
2. Console: Billing -> Budgets -> verá presupuesto-mensual-general.
3. terraform destroy para limpiar.
{% endstep %}
{% endstepper %}

{% hint style="info" %}
Punto clave: Los budgets aplicados como código son una red de seguridad crítica contra costes descontrolados; aplíquelos desde el primer día en cada cuenta.
{% endhint %}

***

## Módulo 9.7: Automatización del Monitoreo y Buenas Prácticas

### 1. Día 0 vs. Día 2

* Día 0: terraform apply para crear recursos.
* Día 2: gestionar, monitorear y mantener. Terraform es crítico para el Día 2.

### 2. Terraform como herramienta de Día 2 (resumen)

* Logs: aws\_cloudwatch\_log\_group + user\_data instalan automáticamente el agente.
* Métricas: Agente de CloudWatch envía métricas personalizadas uniformes.
* Alarmas: aws\_cloudwatch\_metric\_alarm crea alarmas junto con recursos.
* Dashboards: aws\_cloudwatch\_dashboard versionado en Git.
* Costes: aws\_budgets\_budget aplicado como código.

### 3. Buenas Prácticas de Observabilidad

* Registrar todo y configurar lifecycle rules en S3 para retención económica.
* Alertar sobre síntomas (latencia) no solo causas (CPU).
* Usar namespaces y dimensiones para organizar métricas.
* Proteger logs de auditoría: bucket S3 de CloudTrail debe ser seguro (Object Lock si procede).

### 4. Material de Apoyo

* Pilar de Excelencia Operativa (AWS Well-Architected): [https://docs.aws.amazon.com/wellarchitected/latest/operational-excellence-pillar/welcome.html](https://docs.aws.amazon.com/wellarchitected/latest/operational-excellence-pillar/welcome.html)
* Amazon CloudWatch - Página de Producto: \
  [https://aws.amazon.com/es/cloudwatch/](https://aws.amazon.com/es/cloudwatch/)

***
