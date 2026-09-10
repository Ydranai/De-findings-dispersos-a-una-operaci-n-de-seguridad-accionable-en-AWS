#  AWS Security Operations Dashboard

## De findings dispersos a una operación de seguridad accionable en AWS

Este proyecto presenta una arquitectura de referencia para construir un **Dashboard Centralizado de Operaciones de Seguridad en AWS**, integrando información de **AWS Security Hub, Amazon GuardDuty e AWS IAM**.

El objetivo es transformar señales de seguridad distribuidas en diferentes servicios en una vista operacional que permita responder rápidamente:

-  ¿Qué está ocurriendo?
-  ¿Qué recurso o identidad está afectado?
-  ¿Cuál es la severidad?
-  ¿Existen credenciales que requieren rotación?
-  ¿Qué usuarios requieren atención?
-  ¿Qué debe priorizar el equipo para mitigación?

La solución utiliza servicios serverless y administrados de AWS para recolectar, procesar, visualizar y notificar eventos relevantes.

---

## Arquitectura

La solución centraliza las señales generadas por **AWS Security Hub, Amazon GuardDuty e IAM** y utiliza **Amazon EventBridge, AWS Lambda, Amazon CloudWatch y Amazon SNS** para procesarlas, visualizarlas y generar notificaciones.

> La imagen de arquitectura funcional del proyecto se encuentra disponible en la documentación del repositorio.

---

## Componentes

### AWS Security Hub

Security Hub proporciona una vista centralizada de findings relacionados con postura de seguridad, configuración y cumplimiento.

El dashboard permite visualizar findings clasificados por:

- CRITICAL
- HIGH
- MEDIUM
- LOW

Además de los indicadores agregados, se mantiene una vista operativa de findings prioritarios para facilitar su investigación y remediación.

###  Amazon GuardDuty

GuardDuty proporciona detección administrada de amenazas.

Los findings relevantes pueden integrarse en el flujo operacional para identificar comportamientos sospechosos o amenazas sobre recursos AWS.

La arquitectura permite priorizar detecciones de severidad alta y crítica.

### AWS IAM

La solución incorpora indicadores de postura de identidades y credenciales.

Se analizan:

- Usuarios IAM.
- Acceso a consola.
- Estado de MFA.
- Access Keys activas.
- Antigüedad de Access Keys.
- Último uso de credenciales.
- Credenciales sin utilización prolongada.
- Access Keys asociadas a root.
- Estado de MFA de root.

---

##  Control de Access Keys

Uno de los objetivos del dashboard es proporcionar visibilidad sobre la higiene de credenciales.

| Estado | Criterio |
|---|---|
| 🟢 OK | Menor a 75 días |
| 🟡 Warning | Entre 75 y 89 días |
| 🔴 Critical | 90 días o más |
| 🟠 Unused | Sin utilización durante 90 días o más |

> La antigüedad y el último uso se analizan de manera independiente. Una credencial antigua puede continuar siendo utilizada activamente y requerir un proceso controlado de rotación.

---

## Dashboard operacional

Amazon CloudWatch centraliza la visualización de la postura de seguridad.

### Security posture

- Findings activos.
- Findings por severidad.
- Findings de GuardDuty.
- Controles de seguridad fallidos.
- Findings HIGH/CRITICAL.

### Identity & Access

- Access Keys activas.
- Access Keys próximas a obsolescencia.
- Access Keys con antigüedad superior al umbral.
- Access Keys sin uso.
- Usuarios con acceso a consola sin MFA.
- Estado MFA de root.
- Access Keys de root.

### Tablas operativas

Además de los KPIs, se utilizan **CloudWatch Logs Insights** para mostrar información accionable como:

- Usuario.
- Access Key.
- Estado.
- Edad de la credencial.
- Último uso.
- Días sin uso.
- Servicio utilizado.
- Región.
- Estado de MFA.

Esto permite pasar de saber que existen credenciales que requieren atención a identificar **cuáles son, quién las utiliza, cuándo fueron utilizadas y sobre qué servicios AWS**.

---

## 📈 Métricas personalizadas

La solución publica métricas personalizadas en CloudWatch, entre ellas:

```text
ActiveFindingsTotal
ActiveGuardDutyFindings
FailedSecurityControls

IAMActiveAccessKeys
IAMAccessKeysWarningAge
IAMAccessKeysCriticalAge
IAMAccessKeysUnused

IAMConsoleUsersWithoutMFA
IAMRootAccessKeys
IAMRootMFAMissing
```

Los nombres pueden modificarse de acuerdo con los estándares de cada organización.

---

## Procesamiento de eventos

Amazon EventBridge permite procesar findings prioritarios y ejecutar actualizaciones programadas de la información de seguridad.

AWS Lambda realiza la consulta, normalización y clasificación de la información antes de publicarla en CloudWatch y generar las notificaciones correspondientes.

Los snapshots periódicos permiten mantener actualizados indicadores relacionados con:

- Security Hub.
- IAM.
- Access Keys.
- MFA.
- Root Account.

---

##  Notificaciones

Amazon SNS permite distribuir eventos relevantes al equipo responsable.

Entre las condiciones que pueden generar notificaciones se encuentran:

- CRITICAL Security Finding.
- HIGH Security Finding.
- Access Key ≥ 90 días.
- Access Key sin uso ≥ 90 días.
- Usuario con acceso a consola sin MFA.
- Root Account sin MFA.
- Root Account con Access Keys.

Los mecanismos de entrega pueden configurarse según las necesidades de cada organización.

---

## ⚙️ Despliegue

### Requisitos

Antes de desplegar:

- AWS CLI configurado.
- Permisos para CloudFormation.
- Security Hub habilitado.
- GuardDuty habilitado si se desean sus detecciones.
- CloudWatch disponible.
- Permisos IAM suficientes para crear los recursos del stack.

### Descargar el proyecto

```bash
git clone <REPOSITORY_URL>
cd <REPOSITORY_NAME>
```

### Desplegar CloudFormation

```bash
aws cloudformation deploy \
  --template-file aws-security-operations-dashboard-public.yaml \
  --stack-name security-operations-dashboard \
  --region us-east-1 \
  --capabilities CAPABILITY_NAMED_IAM
```

La región puede modificarse según el entorno.

### Configurar correo para notificaciones

```bash
aws cloudformation deploy \
  --template-file aws-security-operations-dashboard-public.yaml \
  --stack-name security-operations-dashboard \
  --region us-east-1 \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    AlertEmail=security-team@example.com
```

AWS enviará una solicitud de confirmación al correo configurado.

---

## 🔐 Principios de seguridad

### Least Privilege

Los permisos IAM deben limitarse únicamente a las acciones requeridas por el colector.

### No hardcoded credentials

La plantilla no requiere almacenar Access Keys, Secret Keys, passwords ni tokens.

### Parametrización

Los valores dependientes del entorno deben manejarse mediante parámetros.

### Protección de información

No se deben publicar en repositorios públicos:

- AWS Account IDs.
- Access Key IDs reales.
- ARNs internos.
- Direcciones IP privadas.
- Dominios corporativos.
- Nombres de clientes.
- Usuarios reales.
- Logs productivos.
- Identificadores reales de recursos.

---

## Evolución de la arquitectura

La arquitectura puede ampliarse incorporando:

### Amazon Inspector

Para agregar gestión de vulnerabilidades sobre EC2, ECR y Lambda.

### AWS Security Lake

Para centralizar información de seguridad y facilitar análisis posteriores.

### AWS Config

Para ampliar controles de postura y configuración.

### SIEM / SOAR

Los eventos pueden integrarse con plataformas externas de gestión y respuesta de seguridad.

### Automated Remediation

EventBridge y Lambda pueden utilizarse para implementar acciones controladas de remediación, como aislamiento de recursos, ajustes de configuración, tratamiento de credenciales o creación automática de tickets.

Estas acciones deben implementarse con controles y aprobaciones adecuados.

---

## Beneficios

La arquitectura permite:

- Centralizar la postura de seguridad.
- Reducir la navegación entre múltiples servicios.
- Priorizar findings por severidad.
- Identificar rápidamente recursos afectados.
- Mejorar la visibilidad de IAM.
- Detectar credenciales obsoletas.
- Identificar usuarios sin MFA.
- Mantener trazabilidad operacional.
- Generar notificaciones de seguridad.
- Facilitar el proceso de remediación.

---

##  Idea principal

Un dashboard de seguridad no debería limitarse a responder:

> **¿Cuántos findings tengo?**

También debería ayudar al equipo a responder:

> **¿Qué debo corregir primero y sobre qué recurso?**

El objetivo de este proyecto es acercar las señales de seguridad a una operación realmente accionable.

---

##  Disclaimer

Este repositorio contiene una **arquitectura de referencia**.

Los nombres, parámetros y configuraciones publicados son genéricos y no representan infraestructura de un cliente específico.

Antes de utilizar la solución en producción se recomienda validar:

- Políticas IAM.
- Costos.
- Retención de logs.
- Frecuencia de ejecución.
- Umbrales de seguridad.
- Integraciones de notificación.
- Requisitos regulatorios y organizacionales.

---

## Autor

**Yindra Torres**

Cloud & AWS enthusiast focused on:

- AWS Architecture
- Cloud Operations
- Security
- FinOps
- Automation
- Artificial Intelligence

⭐ Si este proyecto te resulta útil, puedes darle una **Star** al repositorio.

Contributions, suggestions and improvements are welcome.
