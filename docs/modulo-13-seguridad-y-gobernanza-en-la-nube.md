# Módulo 13 - SEGURIDAD Y GOBERNANZA EN LA NUBE

## Módulo 13.1: Gobernanza en la Nube y AWS Organizations

### 1. ¿Qué es la Gobernanza en la Nube?

La **Gobernanza** es la respuesta a la pregunta: "¿Cómo nos aseguramos de que nuestros equipos están usando la nube de forma correcta?".

Implica establecer "barreras de seguridad" (guardrails) para controlar:

* **Costes:** ¿Quién puede lanzar instancias x2large?
* **Seguridad:** ¿Están todos nuestros buckets S3 cifrados?
* **Cumplimiento:** ¿Cumplimos con la normativa RGPD?
* **Operaciones:** ¿Están todos los recursos etiquetados correctamente?

### 2. El Pilar de la Gobernanza: AWS Organizations

Como vimos en el Módulo 2.2, **AWS Organizations** es el servicio fundamental para gestionar múltiples cuentas de AWS. Es la herramienta principal para implementar la gobernanza.

Recordatorio del modelo:

* **Cuenta Maestra (Management Account):** La cuenta raíz que paga la factura y aplica las políticas.
* **Unidades Organizativas (OUs):** Carpetas para agrupar cuentas (ej. OU-Produccion, OU-Desarrollo, OU-Sandbox).
* **Cuentas Miembro:** Las cuentas individuales de AWS donde se despliegan los recursos.

### 3. La Herramienta de Gobernanza: Políticas de Control de Servicio (SCP)

Las **SCPs (Service Control Policies)** son la herramienta de gobernanza más potente de AWS Organizations.

* **¿Qué son?** Políticas JSON (similares a IAM) que se aplican a una OU o a una cuenta.
* **¿Qué hacen?** Actúan como un "filtro" o "barrera de seguridad" que restringe las acciones que los usuarios (incluido el administrador de la cuenta) pueden realizar.
* **Deny vs. Allow:** Las SCPs **NO** conceden permisos. Solo los filtran. La política de IAM de un usuario debe seguir permitiendo la acción.
* **Efecto:** Permiso Final = Política de IAM (Usuario) ∩ Política de SCP (OU)

Ejemplo:

* Política de IAM (Usuario): Allow: ec2:\* (Permite todo en EC2).
* Política de SCP (OU): Deny: ec2:RunInstances si la instancia _no_ es t3.micro.
* Resultado: El usuario solo podrá lanzar instancias t3.micro. Cualquier otra acción (como ec2:DescribeInstances) funcionará, pero RunInstances para una m5.large será **denegado**.

### 4. Terraform y AWS Organizations (aws\_organizations\_...)

Terraform puede gestionar toda la estructura de AWS Organizations:

* aws\_organization: Para gestionar la propia organización.
* aws\_organizations\_account: Para crear nuevas cuentas de AWS.
* aws\_organizations\_organizational\_unit: Para crear OUs.
* aws\_organizations\_policy: Para crear una SCP.
* aws\_organizations\_policy\_attachment: Para adjuntar la SCP a una OU.

Flujo de Trabajo (Bootstrap)

{% stepper %}
{% step %}
### Habilitar Organizations y preparar IAM

1. Habilite AWS Organizations manualmente en la cuenta raíz (Maestra).
2. Cree un Rol de IAM en la cuenta Maestra con permisos organizations:\*.
{% endstep %}

{% step %}
### Ejecutar Terraform desde un proyecto que asume el rol

1. Un proyecto de Terraform (ej. terraform-org-setup) asume el rol creado en la cuenta Maestra.
2. El proyecto usa los recursos aws\_organizations\_\* para construir la jerarquía de OUs y aplicar las SCPs.
{% endstep %}
{% endstepper %}

{% hint style="info" %}
Las SCPs son la "barrera de seguridad" definitiva. Son la única forma de imponer una regla a un administrador de cuenta.
{% endhint %}

Material de apoyo

* Documentos Clave
  * [Documentación de AWS Organizations (Terraform)](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/organizations_organization)
  * [Políticas de Control de Servicio (SCPs) (AWS Docs)](https://docs.aws.amazon.com/es_es/organizations/latest/userguide/orgs_manage_policies_scps.html)

## Módulo 13.2: Lab - Políticas de Etiquetado (Tagging) Obligatorias

### 1. Introducción

El problema: FinOps no sabe qué equipo es dueño de qué instancia. Solución: forzar etiquetas Proyecto y Entorno en cada recurso. Herramientas posibles: Tag Policies (Organizations) o AWS Config. En este lab usaremos una SCP (más estricta) para forzar etiquetado.

### 2. Objetivo del Laboratorio

Crear una SCP que **deniegue** la creación de cualquier instancia EC2 si _no_ tiene la etiqueta Proyecto.

### 3. Definir la Política de SCP (main.tf)

(Asumimos que estamos en un proyecto de Terraform que gestiona la Organización, como se describió en 13.1).

{% code title="main.tf" %}
```terraform
# 1. Definir la Política de SCP
data "aws_iam_policy_document" "scp_forzar_etiqueta_proyecto" {
  statement {
    sid    = "DenegarEC2SinEtiquetaProyecto"
    effect = "Deny"
    actions = [
      # La acción de crear la instancia
      "ec2:RunInstances"
    ]
    resources = [
      # Se aplica a la creación de instancias
      "arn:aws:ec2:*:*:instance/*"
    ]
    condition {
      test     = "StringNotLike"
      variable = "aws:RequestTag/Proyecto"
      # Denegar si la etiqueta "Proyecto" NO es "algo"
      # (es decir, si no existe o está vacía)
      values = ["?*"]
    }
  }
}

# 2. Crear el recurso de Política en Organizations
resource "aws_organizations_policy" "scp_etiqueta_proyecto" {
  name        = "ForzarEtiquetaProyecto"
  description = "Deniega RunInstances si falta la etiqueta Proyecto"
  content     = data.aws_iam_policy_document.scp_forzar_etiqueta_proyecto.json
  type        = "SERVICE_CONTROL_POLICY"
}

# 3. (Opcional) Adjuntar la Política a una OU
# Necesitaríamos el ID de la OU (ej. 'ou-dev')
# resource "aws_organizations_policy_attachment" "attach_scp" {
#   policy_id = aws_organizations_policy.scp_etiqueta_proyecto.id
#   target_id = "ou-1234-abcdef" # ID de la OU de Desarrollo
# }
```
{% endcode %}

Notas sobre la condición:

* aws:RequestTag/Proyecto: clave de condición que comprueba el valor de la etiqueta en el momento de la solicitud.
* "?_": comodín que significa "al menos un carácter". StringNotLike "?_" = nulo o vacío.

### 4. Ejecutar y Verificar

{% stepper %}
{% step %}
### Aplicar Terraform

* Ejecutar: terraform apply
* Escriba: yes
* Resultado: Terraform creará la SCP en la cuenta Maestra.
{% endstep %}

{% step %}
### Verificación Manual

1. Consola → AWS Organizations → Políticas → Service Control Policies.
2. Debería ver la política ForzarEtiquetaProyecto.
{% endstep %}

{% step %}
### Adjuntar la Política (Manual)

1. Ir a la "Raíz" (Root) o a la OU de Desarrollo.
2. Hacer clic en "Adjuntar" y seleccionar la nueva política.
{% endstep %}

{% step %}
### Prueba (Falla)

1. Inicie sesión en una cuenta miembro afectada por la SCP.
2. EC2 → "Lanzar Instancia".
3. Complete todos los pasos pero NO añada etiquetas.
4. Intentar lanzar: la consola mostrará un error de autorización.
{% endstep %}

{% step %}
### Prueba (Éxito)

1. Repetir lanzamiento y en la página final añadir etiqueta:
   * Clave: Proyecto
   * Valor: mi-test
2. Lanzar: la instancia se creará.
{% endstep %}
{% endstepper %}

{% hint style="warning" %}
Punto clave: Las SCPs son el "martillo" de la gobernanza. Son muy eficaces para bloquear acciones. Limitación: si queremos solo auditar (no bloquear), use AWS Config.
{% endhint %}

## Módulo 13.3: Lab - AWS Config: Auditoría y Cumplimiento como Código

### 1. ¿Qué es AWS Config?

* CloudTrail = libro de registro (quién hizo qué).
* AWS Config = auditor / inventario.
* AWS Config descubre y registra continuamente la configuración de recursos y su historial.
* Provee inventario (CMDB) y auditoría de cambios.

### 2. AWS Config Rules (Reglas de Configuración)

* Permite evaluar recursos contra reglas (por ejemplo, "Buckets S3 deben tener versionado").
* Resultado: Compliant / Non-Compliant.
* Opcionalmente se pueden añadir acciones de remediación (Lambda, etc.).

### 3. Objetivo del Laboratorio

Usar Terraform para configurar AWS Config y desplegar una regla gestionada por AWS:

* Configurar AWS Config para auditar toda la cuenta.
* Desplegar la regla: s3-bucket-public-read-prohibited

### 4. Paso 1: Configurar el "Grabador" (Recorder)

Terraform: crear bucket, política, rol y activar recorder & delivery channel.

{% code title="main.tf (Config - recorder y delivery)" %}
```terraform
# 1. Bucket S3 para los datos de Config
resource "aws_s3_bucket" "bucket_config" {
  bucket = "mi-cuenta-logs-config-12345"
}

# 2. Política de Bucket para permitir a Config escribir
data "aws_iam_policy_document" "politica_bucket_config" {
  statement {
    effect = "Allow"
    actions = ["s3:PutObject"]
    resources = ["${aws_s3_bucket.bucket_config.arn}/*"]
    principals {
      type        = "Service"
      identifiers = ["config.amazonaws.com"]
    }
  }
}

resource "aws_s3_bucket_policy" "config_policy" {
  bucket = aws_s3_bucket.bucket_config.id
  policy = data.aws_iam_policy_document.politica_bucket_config.json
}

# 3. Rol de IAM para que Config lea los recursos
data "aws_iam_policy_document" "confianza_config" {
  statement {
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["config.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "rol_config" {
  name               = "rol-aws-config"
  assume_role_policy = data.aws_iam_policy_document.confianza_config.json
}

# 4. Adjuntar la política gestionada por AWS
resource "aws_iam_role_policy_attachment" "config_permisos" {
  role       = aws_iam_role.rol_config.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWS_ConfigRole"
}

# 5. Activar el Grabador de Configuración
resource "aws_config_configuration_recorder" "recorder" {
  name     = "default"
  role_arn = aws_iam_role.rol_config.arn

  recording_group {
    all_supported = true
  }
}

# 6. Configurar el Canal de Entrega
resource "aws_config_delivery_channel" "channel" {
  name          = "default"
  s3_bucket_name = aws_s3_bucket.bucket_config.name

  depends_on = [
    aws_config_configuration_recorder.recorder,
    aws_iam_role.rol_config
  ]
}
```
{% endcode %}

### 5. Paso 2: Desplegar la Regla de Cumplimiento (main.tf)

Desplegar la regla gestionada por AWS.

{% code title="main.tf (Config - regla gestionada)" %}
```terraform
# 7. Desplegar una Regla Gestionada por AWS
resource "aws_config_config_rule" "s3_public_read" {
  name = "s3-bucket-public-read-prohibited"

  source {
    owner             = "AWS"
    source_identifier = "S3_BUCKET_PUBLIC_READ_PROHIBITED"
  }

  depends_on = [aws_config_configuration_recorder.recorder]
}
```
{% endcode %}

### 6. Paso 3: Ejecutar y Verificar

{% stepper %}
{% step %}
### Aplicar Terraform

* Ejecutar: terraform apply
* Escriba: yes
{% endstep %}

{% step %}
### Verificar en la consola

1. Consola → AWS Config.
2. Verá que el grabador está "Grabando".
3. En "Reglas" aparecerá s3-bucket-public-read-prohibited.
{% endstep %}

{% step %}
### Probar la regla

1. Cree un bucket S3 público.
2. Espere unos minutos.
3. La regla pasará a "No Conforme" y detallará qué bucket viola la política.
{% endstep %}

{% step %}
### Limpiar

* Ejecutar: terraform destroy
{% endstep %}
{% endstepper %}

{% hint style="info" %}
SCP vs. Config:

* SCP: Preventivo (bloquea acciones).
* Config: Detectivo (audita y reporta). Use ambos según la necesidad.
{% endhint %}

## Módulo 13.4: Cumplimiento de Normativas (RGPD, ISO, SOC2)

### 1. Introducción

El cumplimiento es un resultado: adherirse a reglas de gobierno o industria. AWS opera bajo el Modelo de Responsabilidad Compartida:

* AWS: responsable del cumplimiento DE la nube (centros de datos, hardware, servicios).
* Usted: responsable del cumplimiento EN la nube (usar servicios de forma conforme).

Terraform no hace "conforme" por sí solo, pero ayuda a implementar y probar controles.

### 2. Herramientas de AWS para el Cumplimiento

1. AWS Artifact: portal para descargar informes de auditoría (ISO, SOC2, PCI).
2. AWS Config:
   * Audita configuraciones.
   * Conformance Packs: plantillas preconstruidas alineadas a estándares.
3. AWS Security Hub:
   * Agrega hallazgos y da puntuación de cumplimiento contra estándares.

### 3. Controles Técnicos (Terraform) para RGPD/ISO

| Requisito de Cumplimiento      | Control Técnico de AWS        | Recurso de Terraform                                                    |
| ------------------------------ | ----------------------------- | ----------------------------------------------------------------------- |
| Cifrado de Datos en Reposo     | AWS KMS, Cifrado de EBS/S3    | aws\_kms\_key, aws\_s3\_bucket\_server\_side\_encryption\_configuration |
| Cifrado de Datos en Tránsito   | Política de S3 (Forzar HTTPS) | data "aws\_iam\_policy\_document"                                       |
| Auditoría de Acceso (Logging)  | AWS CloudTrail                | aws\_cloudtrail                                                         |
| Principio de Mínimo Privilegio | Políticas de IAM              | aws\_iam\_policy, aws\_iam\_role                                        |
| Seguridad de Red (Firewall)    | Grupos de Seguridad (SGs)     | aws\_security\_group                                                    |
| Soberanía de Datos (RGPD)      | Restricción de Regiones (SCP) | aws\_organizations\_policy                                              |

{% hint style="info" %}
Punto clave: El cumplimiento no es un producto, es un proceso de aplicar controles y auditar. Terraform es ideal para implementar estos controles como código.
{% endhint %}

## Módulo 13.5: Lab - Restricción de Regiones (con SCP)

### 1. Introducción

Problema: operar solo en Europa por RGPD. Solución: SCP que deniegue todas las regiones excepto las aprobadas (ej. eu-west-1).

### 2. Objetivo del Laboratorio

Crear y adjuntar una SCP que solo permita acciones en la región eu-west-1 (y excepciones necesarias).

### 3. Definir la Política de SCP (main.tf)

(En el proyecto terraform-org-setup de la cuenta Maestra)

{% code title="main.tf" %}
```terraform
data "aws_iam_policy_document" "scp_restringir_regiones" {
  statement {
    sid    = "PermitirSoloRegionUE"
    effect = "Deny"
    # Denegar cualquier acción excepto not_actions (excepciones)
    not_actions = [
      # Servicios globales que deben permitirse siempre
      "iam:*",
      "organizations:*",
      "route53:*",
      "s3:GetAccountPublicAccessBlock",
      "cloudfront:*",
      "sts:*"
    ]
    resources = ["*"]
    condition {
      test     = "StringNotEquals"
      variable = "aws:RequestedRegion"
      # Denegar si la región NO ES una de estas
      values = ["eu-west-1", "us-east-1"] # us-east-1 necesaria para IAM global
    }
  }
}

resource "aws_organizations_policy" "scp_restringir_regiones" {
  name        = "Restringir-Regiones-No-UE"
  description = "Solo permite operaciones en regiones de la UE (y us-east-1)"
  content     = data.aws_iam_policy_document.scp_restringir_regiones.json
  type        = "SERVICE_CONTROL_POLICY"
}

resource "aws_organizations_policy_attachment" "attach_scp" {
  policy_id = aws_organizations_policy.scp_restringir_regiones.id
  target_id = "ou-1234-abcdef" # Reemplace con su OU
}
```
{% endcode %}

### 4. Ejecutar y Verificar

{% stepper %}
{% step %}
### Aplicar Terraform

* Ejecutar: terraform apply
* Escriba: yes
{% endstep %}

{% step %}
### Verificación práctica

1. Inicie sesión en una cuenta miembro de la OU afectada.
2. Prueba 1 (Éxito): Región eu-west-1 → EC2 Dashboard debería cargar.
3. Prueba 2 (Fallo): Cambie a us-east-2 → EC2 Dashboard dará error de autorización.
{% endstep %}
{% endstepper %}

{% hint style="warning" %}
Las excepciones (not\_actions) son críticas. Servicios globales como IAM deben excluirse, o bloqueará la capacidad de iniciar sesión o gestionar identidades.
{% endhint %}

## Módulo 13.6: Monitoreo de Cumplimiento con AWS Security Hub

### 1. Introducción

Tenemos múltiples paneles (SCPs, Config, GuardDuty, etc.). Security Hub agrega, organiza y prioriza hallazgos de múltiples servicios, dando una puntuación de cumplimiento.

* Security Hub integra: AWS Config, GuardDuty, Inspector, IAM Access Analyzer, y terceros.
* Proporciona puntuación frente a estándares (CIS, PCI-DSS, AWS Foundational...).

### 2. Lab: Habilitar Security Hub con Terraform

{% code title="main.tf (Security Hub)" %}
```terraform
# 1. Habilitar Security Hub en la cuenta
resource "aws_securityhub_account" "this" {}

# 2. Suscribirse al estándar de AWS (Buenas Prácticas)
resource "aws_securityhub_standards_subscription" "aws_foundational" {
  standards_arn = "arn:aws:securityhub:::ruleset/aws-foundational-security-best-practices/v/1.0.0"
}

# 3. (Opcional) Suscribirse al estándar PCI
resource "aws_securityhub_standards_subscription" "pci" {
  standards_arn = "arn:aws:securityhub:us-east-1::standards/pci-dss/v/3.2.1"
  depends_on    = [aws_securityhub_account.this]
}
```
{% endcode %}

### 3. Ejecutar y Verificar

{% stepper %}
{% step %}
### Aplicar Terraform

* Ejecutar: terraform apply
* Escriba: yes
{% endstep %}

{% step %}
### Verificar en consola

1. Consola → Security Hub.
2. Esperar 15–30 minutos para que se pueble la información.
3. Verá un resumen con la puntuación de seguridad y controles listados.
{% endstep %}
{% endstepper %}

{% hint style="info" %}
Security Hub no reemplaza a Config; lo consume. En organizaciones, habilítelo en la cuenta Maestra (o cuenta de seguridad delegada) y agregue hallazgos de las cuentas miembro.
{% endhint %}

## Módulo 13.7: Gestión de Riesgos y Buenas Prácticas de Gobernanza (Resumen)

Tres capas de Gobernanza en la Nube:

1. Capa 1: Prevención (El Martillo)
   * Herramienta: AWS Organizations (SCPs).
   * Acción: Deny.
   * Uso: reglas absolutas (ej. impedir región, impedir lanzar sin etiqueta).
2. Capa 2: Detección (El Auditor)
   * Herramienta: AWS Config.
   * Acción: Audit → Non-Compliant.
   * Uso: políticas complejas que se auditan y eventualmente remedian.
3. Capa 3: Agregación (El Panel de Control)
   * Herramienta: AWS Security Hub.
   * Acción: Agregar y Priorizar.
   * Uso: ofrecer un único panel y puntuación de cumplimiento.

Terraform: el habilitador

* Gobernanza como Código (GaC): definir prevención, detección y agregación como código.
* Beneficios: auditable (SCPs en Git), repetible (reglas Config en nuevas cuentas), consistente (Security Hub habilitado igual en todas partes).

Fin del módulo.
