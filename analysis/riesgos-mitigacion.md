# Análisis de Riesgos y Estrategias de Mitigación

## Introducción

Este documento identifica los modos de falla potenciales en la integración SOA propuesta, evalúa su impacto en los atributos de calidad (latencia, disponibilidad, escalabilidad) y define estrategias de mitigación accionables para asegurar resiliencia y cumplimiento de los SLAs.

## Metodología

Los riesgos se evalúan utilizando la siguiente matriz:

| **Probabilidad** | **Impacto**       | **Nivel de Riesgo** |
|------------------|-------------------|----------------------|
| Baja (1)         | Bajo (1)          | Bajo                 |
| Baja (1)         | Alto (3)          | Medio                |
| Alta (3)         | Bajo (1)          | Medio                |
| Alta (3)         | Alto (3)          | Alto                 |

- **Probabilidad**: Frecuencia estimada de ocurrencia (1: Baja, 2: Media, 3: Alta).
- **Impacto**: Gravedad del efecto en los atributos de calidad (1: Bajo, 2: Medio, 3: Alto).
- **Nivel de Riesgo**: Producto de probabilidad e impacto.

## Riesgos Identificados

### 1. **Fallo en el API Gateway**
- **Descripción**: Caída o saturación del API Gateway, dejando inaccesibles todos los servicios.
- **Causas**:
  - Ataque DDoS.
  - Configuración incorrecta.
  - Límites de capacidad excedidos.
- **Impacto**:
  - **Disponibilidad**: SLA 99.9% no cumplido.
  - **Latencia**: Aplicaciones móviles inutilizables.
- **Probabilidad**: 2 (Media).
- **Impacto**: 3 (Alto).
- **Nivel de Riesgo**: Alto.

#### Estrategias de Mitigación
1. **Failover Activo-Pasivo**:
   - Configurar un **segundo API Gateway** en otra zona de disponibilidad.
   - Usar **AWS Route 53** con health checks para redirigir tráfico en caso de fallo.
   - **Responsable**: Equipo de operaciones.
   - **Plazo**: 2 semanas.

2. **Rate Limiting y WAF**:
   - Implementar **AWS WAF** para proteger contra DDoS.
   - Configurar **rate limiting** por IP y por servicio.
   - **Responsable**: Equipo de seguridad.
   - **Plazo**: 1 semana.

3. **Caching de Respuestas**:
   - Cachear respuestas frecuentes en **Amazon ElastiCache (Redis)**.
   - Reducir carga en el API Gateway durante picos.
   - **Responsable**: Equipo de desarrollo.
   - **Plazo**: 3 semanas.


### 2. **Fallo en el Servicio de Autenticación**
- **Descripción**: El servicio de autenticación (Cognito/Keycloak) falla, impidiendo el acceso de usuarios.
- **Causas**:
  - Sobrecarga de solicitudes.
  - Configuración incorrecta de tokens.
  - Fallo en la base de datos de identidades.
- **Impacto**:
  - **Disponibilidad**: Bloqueo de todos los usuarios.
  - **Seguridad**: Riesgo de accesos no autorizados.
- **Probabilidad**: 2 (Media).
- **Impacto**: 3 (Alto).
- **Nivel de Riesgo**: Alto.

#### Estrategias de Mitigación
1. **Multi-Zona y Autoescalado**:
   - Desplegar el servicio de autenticación en **múltiples zonas de disponibilidad**.
   - Configurar **autoescalado** basado en métricas de CPU y memoria.
   - **Responsable**: Equipo de operaciones.
   - **Plazo**: 2 semanas.

2. **Circuit Breaker**:
   - Implementar **circuit breaker** para fallos transitorios.
   - Usar **AWS Step Functions** para manejar fallos.
   - **Responsable**: Equipo de desarrollo.
   - **Plazo**: 1 semana.

3. **Cacheo de Tokens**:
   - Cachear tokens válidos en **Redis** para reducir carga en el servicio.
   - Configurar TTL adecuado para evitar tokens obsoletos.
   - **Responsable**: Equipo de desarrollo.
   - **Plazo**: 2 semanas.


### 3. **Fallo en la Base de Datos de Transacciones**
- **Descripción**: Caída o corrupción de la base de datos Aurora, perdiendo datos de transacciones.
- **Causas**:
  - Fallo de hardware.
  - Error humano en configuración.
  - Ataque de inyección SQL.
- **Impacto**:
  - **Disponibilidad**: Pérdida de datos críticos.
  - **Consistencia**: Inconsistencias en saldos y transacciones.
- **Probabilidad**: 1 (Baja).
- **Impacto**: 3 (Alto).
- **Nivel de Riesgo**: Medio.

#### Estrategias de Mitigación
1. **Multi-AZ y Réplicas**:
   - Configurar **Aurora Multi-AZ** para alta disponibilidad.
   - Usar **réplicas de lectura** para distribuir carga.
   - **Responsable**: Equipo de operaciones.
   - **Plazo**: 1 semana.

2. **Backups Automáticos**:
   - Configurar **AWS Backup** para snapshots diarios.
   - Implementar **point-in-time recovery** para recuperar datos específicos.
   - **Responsable**: Equipo de operaciones.
   - **Plazo**: 1 semana.

3. **Validación de Datos**:
   - Implementar **validación de esquemas** en el servicio de transacciones.
   - Usar **constraints** en la base de datos para evitar corrupción.
   - **Responsable**: Equipo de desarrollo.
   - **Plazo**: 2 semanas.


### 4. **Fallo en Servicios Externos (Fraude, Pagos)**
- **Descripción**: Los servicios externos de fraude o pagos fallan, bloqueando transacciones.
- **Causas**:
  - Caída del proveedor externo.
  - Latencia alta en llamadas.
  - Timeouts configurados incorrectamente.
- **Impacto**:
  - **Latencia**: Transacciones rechazadas o retrasadas.
  - **Disponibilidad**: Bloqueo de funcionalidades críticas.
- **Probabilidad**: 3 (Alta).
- **Impacto**: 2 (Medio).
- **Nivel de Riesgo**: Alto.

#### Estrategias de Mitigación
1. **Circuit Breaker y Retries**:
   - Implementar **circuit breaker** con AWS Step Functions.
   - Configurar **reintentos exponenciales** para fallos transitorios.
   - **Responsable**: Equipo de desarrollo.
   - **Plazo**: 1 semana.

2. **Fallback Local**:
   - Implementar **lógica de fallback** para aprobar transacciones con bajo riesgo.
   - Usar **reglas de negocio** para evaluar riesgo sin depender del servicio externo.
   - **Responsable**: Equipo de desarrollo.
   - **Plazo**: 3 semanas.

3. **Caching de Respuestas**:
   - Cachear respuestas de servicios externos para reducir latencia.
   - Configurar TTL corto (ejemplo: 5 minutos).
   - **Responsable**: Equipo de desarrollo.
   - **Plazo**: 2 semanas.


### 5. **Inconsistencia en Datos Distribuidos (Consentimientos vs Transacciones)**
- **Descripción**: Inconsistencias entre consentimientos y transacciones debido a fallos en la comunicación entre servicios.
- **Causas**:
  - Fallos en colas de mensajes (SQS).
  - Errores en la lógica de compensación.
  - Retrasos en la propagación de eventos.
- **Impacto**:
  - **Consistencia**: Transacciones procesadas sin consentimiento válido.
  - **Cumplimiento**: Riesgo de incumplimiento regulatorio (PSD2).
- **Probabilidad**: 2 (Media).
- **Impacto**: 3 (Alto).
- **Nivel de Riesgo**: Alto.

#### Estrategias de Mitigación
1. **Sagas con Compensación**:
   - Implementar **sagas** para transacciones distribuidas.
   - Definir **pasos de compensación** para revertir cambios en caso de fallo.
   - **Responsable**: Equipo de desarrollo.
   - **Plazo**: 4 semanas.

2. **Event Sourcing**:
   - Usar **event sourcing** para registrar todos los cambios de estado.
   - Permitir reconstrucción de estados consistentes.
   - **Responsable**: Equipo de desarrollo.
   - **Plazo**: 5 semanas.

3. **Validación en Tiempo Real**:
   - Implementar **validación síncrona** de consentimientos antes de procesar transacciones.
   - Usar **DynamoDB Streams** para propagar cambios en tiempo real.
   - **Responsable**: Equipo de desarrollo.
   - **Plazo**: 3 semanas.


### 6. **Ataque de Seguridad (OAuth2/FAPI)**
- **Descripción**: Explotación de vulnerabilidades en OAuth2/FAPI, permitiendo accesos no autorizados.
- **Causas**:
  - Configuración incorrecta de tokens.
  - Reutilización de tokens.
  - Falta de validación de scopes.
- **Impacto**:
  - **Seguridad**: Acceso no autorizado a datos sensibles.
  - **Cumplimiento**: Violación de regulaciones (GDPR, PSD2).
- **Probabilidad**: 2 (Media).
- **Impacto**: 3 (Alto).
- **Nivel de Riesgo**: Alto.

#### Estrategias de Mitigación
1. **Validación Estricta de Tokens**:
   - Implementar **validación de scopes** en cada servicio.
   - Usar **short-lived tokens** (ejemplo: 15 minutos).
   - **Responsable**: Equipo de seguridad.
   - **Plazo**: 1 semana.

2. **Monitoreo de Actividad Sospechosa**:
   - Configurar **AWS GuardDuty** para detectar accesos anómalos.
   - Implementar **alertas en tiempo real**.
   - **Responsable**: Equipo de seguridad.
   - **Plazo**: 1 semana.

3. **Autenticación Multifactor (MFA)**:
   - Requerir **MFA** para operaciones sensibles.
   - Usar **AWS Cognito** para gestionar MFA.
   - **Responsable**: Equipo de desarrollo.
   - **Plazo**: 2 semanas.


### 7. **Sobrecarga en Colas de Mensajes (SQS)**
- **Descripción**: La cola de mensajes se satura, retrasando el procesamiento de notificaciones.
- **Causas**:
  - Pico de transacciones.
  - Consumidores lentos.
  - Configuración incorrecta de visibilidad.
- **Impacto**:
  - **Latencia**: Notificaciones retrasadas.
  - **Disponibilidad**: Pérdida de mensajes si la cola se llena.
- **Probabilidad**: 2 (Media).
- **Impacto**: 2 (Medio).
- **Nivel de Riesgo**: Medio.

#### Estrategias de Mitigación
1. **Autoescalado de Consumidores**:
   - Configurar **autoescalado** para consumidores de SQS.
   - Usar **AWS Lambda** para procesamiento paralelo.
   - **Responsable**: Equipo de operaciones.
   - **Plazo**: 1 semana.

2. **Dead Letter Queue (DLQ)**:
   - Configurar **DLQ** para mensajes que fallan repetidamente.
   - Implementar **reprocesamiento automático**.
   - **Responsable**: Equipo de desarrollo.
   - **Plazo**: 1 semana.

3. **Priorización de Mensajes**:
   - Implementar **colas de prioridad** para mensajes críticos.
   - Usar **SQS con FIFO** para garantizar orden.
   - **Responsable**: Equipo de desarrollo.
   - **Plazo**: 2 semanas.

## Resumen de Riesgos y Mitigaciones

| **Riesgo**                          | **Probabilidad** | **Impacto** | **Nivel** | **Mitigaciones Clave**                          |
|-------------------------------------|------------------|-------------|-----------|-----------------------------------------------|
| Fallo en el API Gateway             | 2                | 3           | Alto      | Failover, WAF, caching                          |
| Fallo en el Servicio de Autenticación| 2                | 3           | Alto      | Multi-Zona, circuit breaker, cacheo de tokens   |
| Fallo en Base de Datos de Transacciones | 1              | 3           | Medio     | Multi-AZ, backups, validación de datos          |
| Fallo en Servicios Externos         | 3                | 2           | Alto      | Circuit breaker, fallback, caching              |
| Inconsistencia en Datos Distribuidos | 2              | 3           | Alto      | Sagas, event sourcing, validación en tiempo real |
| Ataque de Seguridad (OAuth2/FAPI)    | 2                | 3           | Alto      | Validación de tokens, MFA, monitoreo            |
| Sobrecarga en Colas de Mensajes     | 2                | 2           | Medio     | Autoescalado, DLQ, priorización                 |

## Plan de Acción

| **Prioridad** | **Riesgo**                          | **Acciones**                                                                 | **Responsable**       | **Plazo**   |
|---------------|-------------------------------------|-----------------------------------------------------------------------------|-----------------------|-------------|
| Alta          | Fallo en el API Gateway             | Configurar failover activo-pasivo y WAF                                    | Operaciones           | 2 semanas   |
| Alta          | Fallo en el Servicio de Autenticación| Multi-Zona y circuit breaker                                               | Operaciones           | 2 semanas   |
| Alta          | Fallo en Servicios Externos         | Circuit breaker y fallback local                                           | Desarrollo            | 1 semana    |
| Alta          | Inconsistencia en Datos Distribuidos | Implementar sagas y validación en tiempo real                              | Desarrollo            | 4 semanas   |
| Alta          | Ataque de Seguridad (OAuth2/FAPI)    | Validación estricta de tokens y MFA                                        | Seguridad             | 2 semanas   |
| Media         | Fallo en Base de Datos de Transacciones | Configurar Multi-AZ y backups automáticos                               | Operaciones           | 1 semana    |
| Media         | Sobrecarga en Colas de Mensajes      | Autoescalado de consumidores y DLQ                                         | Operaciones           | 1 semana    |

## Conclusión

El análisis de riesgos revela que los principales puntos de falla están relacionados con la disponibilidad de componentes críticos (API Gateway, autenticación, servicios externos) y la consistencia de datos distribuidos. Las estrategias de mitigación propuestas abordan estos riesgos mediante redundancia, resiliencia y observabilidad.

Implementar estas mitigaciones garantizará que la arquitectura cumpla con los SLAs de latencia (< 200ms), disponibilidad (99.9%) y escalabilidad (10,000 usuarios concurrentes), además de proteger contra amenazas de seguridad y fallos catastróficos.

El siguiente paso es priorizar la implementación de las mitigaciones según el plan de acción y monitorear su efectividad mediante métricas clave.