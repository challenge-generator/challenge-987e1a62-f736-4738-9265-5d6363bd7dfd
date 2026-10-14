# Atributos de Calidad y Restricciones de Arquitectura

## Introducción
Este documento identifica y describe los atributos de calidad relevantes para la arquitectura del sistema, así como las restricciones técnicas y operativas que deben cumplirse para garantizar un diseño robusto y escalable. Los atributos de calidad han sido priorizados en función de los requisitos del negocio y las expectativas de los stakeholders.

## Atributos de Calidad

### 1. Latencia
**Definición:** Tiempo máximo permitido para que una solicitud sea procesada y respondida por el sistema.

**Requisitos:**
- **Aplicaciones móviles:** Máximo 200ms para operaciones críticas (ej. consultas de saldo, pagos).
- **Aplicaciones web:** Máximo 500ms para operaciones no críticas (ej. generación de reportes).

**Justificación:**
La latencia es crítica para la experiencia del usuario, especialmente en entornos móviles donde la conectividad puede ser intermitente. Una latencia alta puede resultar en abandono de transacciones y pérdida de confianza en el sistema.

**Estrategias de mitigación:**
- Uso de **caché distribuida** (Amazon ElastiCache) para reducir tiempos de acceso a datos frecuentes.
- **Optimización de consultas** en bases de datos (índices, particionamiento).
- **Edge computing** mediante Amazon CloudFront para servir contenido estático y reducir la latencia geográfica.
- **Desacoplamiento de procesos** mediante colas (Amazon SQS) para operaciones asíncronas.

---

### 2. Disponibilidad
**Definición:** Porcentaje de tiempo en el que el sistema está operativo y accesible para los usuarios.

**Requisitos:**
- **SLA:** 99.9% de disponibilidad mensual (equivalente a ~43 minutos de downtime permitido al mes).
- **Recuperación ante fallos:** Máximo 5 minutos para recuperación automática.

**Justificación:**
La disponibilidad es un requisito no negociable para sistemas financieros, donde cualquier interrupción puede resultar en pérdidas económicas y reputacionales.

**Estrategias de mitigación:**
- **Multi-AZ (Availability Zones)** para desplegar servicios críticos en múltiples zonas de disponibilidad de AWS.
- **Auto-scaling** para manejar picos de carga y garantizar capacidad suficiente.
- **Health checks** y **circuit breakers** para detectar y aislar componentes fallidos.
- **Backups automáticos** y **replicación cross-region** para bases de datos (Amazon DynamoDB Global Tables).
- **Monitoreo proactivo** con Amazon CloudWatch y alertas en tiempo real.

---

### 3. Escalabilidad
**Definición:** Capacidad del sistema para manejar un aumento en la carga de trabajo sin degradación del rendimiento.

**Requisitos:**
- **Usuarios concurrentes:** 10,000 usuarios simultáneos.
- **Transacciones por segundo:** 1,000 TPS en horas pico.
- **Crecimiento:** Escalabilidad horizontal para manejar un crecimiento del 20% anual.

**Justificación:**
El sistema debe ser capaz de manejar picos de demanda sin requerir rediseños costosos. La escalabilidad horizontal permite añadir recursos de manera incremental.

**Estrategias de mitigación:**
- **Arquitectura serverless** (AWS Lambda) para escalar automáticamente según la demanda.
- **Bases de datos escalables** (Amazon DynamoDB) con capacidad de lectura/escritura ajustable.
- **Colas de mensajes** (Amazon SQS) para desacoplar componentes y manejar carga asíncrona.
- **Balanceadores de carga** (Amazon ALB) para distribuir tráfico entre instancias.
- **Microservicios** para permitir escalar componentes individuales según su demanda.

---

### 4. Seguridad
**Definición:** Protección contra accesos no autorizados, fugas de datos y ataques maliciosos.

**Requisitos:**
- **Autenticación:** Autenticación multifactor (MFA) para usuarios internos.
- **Autorización:** Control de acceso basado en roles (RBAC) con políticas granulares.
- **Cifrado:** Cifrado en tránsito (TLS 1.2+) y en reposo (AES-256).
- **Auditoría:** Registro de todas las operaciones sensibles (logs inmutables).

**Justificación:**
La seguridad es crítica en sistemas financieros para cumplir con regulaciones (ej. GDPR, PCI-DSS) y proteger la información sensible de los usuarios.

**Estrategias de mitigación:**
- **AWS IAM** para gestionar permisos con el principio de mínimo privilegio.
- **Amazon Cognito** para autenticación y autorización de usuarios.
- **AWS KMS** para gestión de claves y cifrado.
- **AWS WAF** para proteger contra ataques comunes (SQL injection, XSS).
- **Logs inmutables** con Amazon CloudTrail y AWS Config.

---

### 5. Mantenibilidad
**Definición:** Facilidad para modificar, actualizar y corregir el sistema sin introducir errores.

**Requisitos:**
- **Modularidad:** Componentes desacoplados con responsabilidades claras.
- **Documentación:** Documentación actualizada de APIs, flujos y decisiones de diseño.
- **Pruebas automatizadas:** Cobertura de pruebas > 80% para componentes críticos.

**Justificación:**
Un sistema mantenible reduce el tiempo y costo de futuras modificaciones, lo que es clave para la agilidad del negocio.

**Estrategias de mitigación:**
- **Patrones de diseño** (ej. Ports & Adapters, Hexagonal) para desacoplar componentes.
- **CI/CD pipelines** (AWS CodePipeline) para automatizar pruebas y despliegues.
- **Documentación en OpenAPI 3.1** para contratos de API.
- **Monitoreo de deuda técnica** con herramientas como SonarQube.

---

### 6. Observabilidad
**Definición:** Capacidad para monitorear, diagnosticar y entender el estado del sistema en tiempo real.

**Requisitos:**
- **Logs:** Registro estructurado de todas las operaciones.
- **Métricas:** Métricas clave (latencia, error rate, throughput) disponibles en dashboards.
- **Trazabilidad:** IDs de correlación para rastrear solicitudes a través de microservicios.

**Justificación:**
La observabilidad es esencial para detectar y resolver problemas rápidamente, especialmente en sistemas distribuidos.

**Estrategias de mitigación:**
- **Amazon CloudWatch** para métricas, logs y dashboards.
- **AWS X-Ray** para trazabilidad de solicitudes.
- **Alertas proactivas** para umbrales críticos (ej. latencia > 200ms, error rate > 1%).

---

## Restricciones del Sistema

### 1. Tecnológicas
- **Lenguaje:** Implementación en lenguajes compatibles con AWS Lambda (Node.js, Python, Java).
- **Frameworks:** Uso de OpenAPI 3.1 para contratos de API.
- **Infraestructura:** Despliegue exclusivo en AWS.

### 2. Operativas
- **Horario de operación:** 24/7 con soporte en horario laboral.
- **Tiempo de respuesta a incidentes:** Máximo 15 minutos para incidentes críticos.
- **Frecuencia de despliegues:** Hasta 2 despliegues por día sin downtime.

### 3. Regulatorias
- **Cumplimiento:** PCI-DSS para manejo de datos de pago.
- **Privacidad:** GDPR para protección de datos personales.
- **Auditoría:** Registros inmutables de todas las operaciones sensibles.

---

## Priorización de Atributos
| Atributo          | Prioridad | Impacto en el Negocio                          |
|-------------------|-----------|-----------------------------------------------|
| Disponibilidad    | Alta      | Pérdida económica y reputacional              |
| Latencia          | Alta      | Experiencia de usuario pobre                  |
| Seguridad         | Alta      | Riesgo legal y pérdida de confianza           |
| Escalabilidad     | Media     | Limitaciones para crecer                       |
| Mantenibilidad    | Media     | Costos elevados de evolución                   |
| Observabilidad    | Media     | Dificultad para diagnosticar problemas         |

---

## Conclusión
Los atributos de calidad identificados son críticos para el éxito del sistema. Las estrategias de mitigación propuestas buscan equilibrar costo, complejidad y beneficios, priorizando siempre la disponibilidad, latencia y seguridad. La arquitectura propuesta debe ser evaluada y refinada en las siguientes fases para garantizar su viabilidad técnica y operativa.