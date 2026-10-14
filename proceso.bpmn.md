# Proceso de Negocio: Arquitectura SOA para Sistema Financiero

## Descripción General
Este documento describe el flujo de negocio para la integración de un sistema financiero con énfasis en atributos de calidad como latencia, disponibilidad y escalabilidad. El proceso está modelado en notación BPMN 2.0 y cubre desde la solicitud del cliente hasta la confirmación de la transacción.

## Diagrama BPMN
```
@startuml
start

:Cliente inicia solicitud;

partition "Frontend Mobile/Web" {
    :Validar sesión;
    if (Sesión válida?) then (Sí)
        :Recopilar datos de solicitud;
    else (No)
        :Redirigir a autenticación;
        stop
    endif
}

partition "API Gateway" {
    :Enrutar solicitud;
    note right: Latencia < 200ms
}

partition "Servicio de Orquestación" {
    :Procesar solicitud;
    if (Datos completos?) then (Sí)
        :Invocar Servicio de Validación;
        if (Validación exitosa?) then (Sí)
            :Invocar Servicio de Transacción;
            if (Transacción exitosa?) then (Sí)
                :Registrar auditoría;
                :Generar confirmación;
            else (No)
                :Reintentar (máx. 3 intentos);
                if (¿Reintentos agotados?) then (Sí)
                    :Notificar fallo;
                    stop
                endif
            endif
        else (No)
            :Notificar error de validación;
            stop
        endif
    else (No)
        :Solicitar datos faltantes;
        stop
    endif
}

partition "Servicios Externos" {
    :Servicio de Validación;
    :Servicio de Transacción;
    :Servicio de Auditoría;
    note right: Disponibilidad 99.9%
}

:Enviar confirmación al cliente;
stop

@enduml
```

## Actores
| Actor                     | Responsabilidad                                                                 |
|---------------------------|---------------------------------------------------------------------------------|
| Cliente                   | Inicia la solicitud y recibe confirmación o notificación de error               |
| Frontend Mobile/Web       | Valida sesión y recopila datos de solicitud                                    |
| API Gateway               | Enruta la solicitud al servicio adecuado con latencia controlada (< 200ms)     |
| Servicio de Orquestación  | Coordina la lógica de negocio entre servicios internos y externos              |
| Servicio de Validación    | Valida los datos de la solicitud                                                |
| Servicio de Transacción   | Ejecuta la transacción financiera                                               |
| Servicio de Auditoría     | Registra eventos para trazabilidad y cumplimiento normativo                    |

## Puntos de Decisión
1. **Validación de sesión**: 
   - Si la sesión no es válida, el flujo redirige al cliente a autenticarse.
   - *Impacto*: Seguridad y acceso controlado.

2. **Completitud de datos**: 
   - Si los datos requeridos no están completos, se solicita al cliente que los proporcione.
   - *Impacto*: Experiencia de usuario y validación temprana.

3. **Validación exitosa**: 
   - Si la validación falla, se notifica al cliente y se detiene el flujo.
   - *Impacto*: Prevención de transacciones inválidas.

4. **Transacción exitosa**: 
   - Si la transacción falla, se reintenta hasta 3 veces antes de notificar al cliente.
   - *Impacto*: Resiliencia y manejo de errores.

## Atributos de Calidad Priorizados
| Atributo          | Métrica                          | Mecanismo de Garantía                                                                 |
|-------------------|----------------------------------|---------------------------------------------------------------------------------------|
| Latencia          | < 200ms                          | - Uso de API Gateway con enrutamiento optimizado
                                   | 
                                   |                                  | - Caché de respuestas frecuentes (Redis)
                                   | 
                                   |                                  | - Servicios desacoplados y asíncronos
                                   |
| Disponibilidad     | SLA 99.9%                        | - Balanceo de carga con AWS ALB
                                   | 
                                   |                                  | - Circuit Breaker (AWS Lambda + Step Functions)
                                   | 
                                   |                                  | - Réplicas multi-zona (AWS RDS/Aurora)
                                   |
| Escalabilidad      | 10,000 usuarios concurrentes    | - Auto-scaling de servicios (ECS/EKS)
                                   | 
                                   |                                  | - Colas de mensajes (Amazon SQS)
                                   | 
                                   |                                  | - Base de datos particionada (DynamoDB/Sharding)
                                   |

## Trade-offs Arquitectónicos
1. **Orquestación vs. Coreografía**:
   - *Decisión*: Orquestación centralizada mediante un Servicio de Orquestación.
   - *Justificación*: Mayor control sobre el flujo de negocio y facilidad para implementar patrones de resiliencia como reintentos y circuit breaker.
   - *Trade-off*: Menor desacoplamiento entre servicios, lo que puede impactar la escalabilidad horizontal.

2. **Síncrono vs. Asíncrono**:
   - *Decisión*: Flujo principal síncrono para garantizar latencia < 200ms, con tareas de auditoría y notificaciones asíncronas.
   - *Justificación*: La experiencia del usuario requiere confirmación inmediata.
   - *Trade-off*: Mayor complejidad en la gestión de estados transitorios.

3. **Monolito vs. Microservicios**:
   - *Decisión*: Arquitectura basada en microservicios con contratos bien definidos (OpenAPI).
   - *Justificación*: Escalabilidad independiente de componentes y despliegue continuo.
   - *Trade-off*: Mayor overhead en gestión de infraestructura y observabilidad.

## Ejemplo de Flujo con Datos Reales
**Solicitud de Transferencia Bancaria**:
1. Cliente envía solicitud con:
   ```json
   {
     "origen": "ES9121000418450200051332",
     "destino": "ES8021000418401234567891",
     "monto": 1500.00,
     "concepto": "Pago factura",
     "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
   }
   ```
2. Frontend valida el token y envía la solicitud al API Gateway.
3. API Gateway enruta al Servicio de Orquestación.
4. Servicio de Orquestación invoca:
   - *Servicio de Validación*: Verifica saldo, formato de cuenta y límites diarios.
   - *Servicio de Transacción*: Ejecuta la transferencia.
   - *Servicio de Auditoría*: Registra el evento en el sistema de logs.
5. Cliente recibe confirmación:
   ```json
   {
     "id_transaccion": "TXN-20231015-1432-0001",
     "estado": "COMPLETADA",
     "timestamp": "2023-10-15T14:32:10Z"
   }
   ```

## Dependencias Externas
| Servicio               | Proveedor       | Métricas de Calidad                                                                 |
|-----------------------|-----------------|------------------------------------------------------------------------------------|
| Autenticación         | AWS Cognito     | Disponibilidad: 99.9%, Latencia: < 100ms                                           |
| Validación de Cuentas | Banco Externo   | Disponibilidad: 99.5%, Latencia: < 500ms (con circuit breaker)                     |
| Procesamiento         | AWS Lambda      | Escalabilidad: 1000 invocaciones concurrentes, Latencia: < 150ms                 |
| Base de Datos         | AWS Aurora      | Disponibilidad: 99.99%, Latencia de escritura: < 50ms                              |

## Modos de Falla y Mitigaciones
| Modo de Falla                          | Impacto                          | Mitigación                                                                         |
|----------------------------------------|----------------------------------|------------------------------------------------------------------------------------|
| Caída del Servicio de Validación       | Transacciones no validadas       | Circuit Breaker + reintentos + fallback con datos cacheados (máx. 5 min)          |
| Latencia alta en Servicio de Transacción | Experiencia de usuario degradada | Timeouts estrictos (< 300ms) + cola de prioridad para transacciones críticas       |
| Sobrecarga del API Gateway             | Degradación general del sistema  | Auto-scaling + rate limiting + caché de respuestas                                 |
| Fallo en la Base de Datos              | Pérdida de datos transitorios    | Réplicas multi-zona + backups automáticos cada 5 min + recuperación en < 15 min   |

## Validación del Proceso
Para validar este flujo, se recomienda:
1. **Pruebas de carga**: Simular 10,000 usuarios concurrentes con herramientas como AWS Load Testing.
2. **Pruebas de resiliencia**: Inyectar fallos en servicios externos (Chaos Engineering).
3. **Monitoreo**: Configurar dashboards en AWS CloudWatch para latencia, tasa de errores y throughput.

---