# Gestión de la comunicación con el servicio externo de notificaciones en ConstanciaUV

- Status: proposed
- Date: 2026-11-03

## Context and Problem Statement

El sistema **ConstanciaUV** requiere generar y almacenar constancias digitales para estudiantes autenticados, manteniendo un historial completo de las mismas (requisitos R1 a R3). 
El problema arquitectónico radica en cómo organizar la comunicación con un servicio de correo electrónico externo (**RT1**), considerando que dicho servicio puede no estar disponible (**RT2**), que las notificaciones pendientes deben conservarse para su reintento (**RT3**), y que se debe prevenir el envío de solicitudes duplicadas de notificación para una misma constancia debido a que el proveedor externo no garantiza por sí mismo una entrega única (**RT4**).

## Decision Drivers

1. **Resiliencia ante fallos externos (Disponibilidad y Tolerancia a Fallos):** Garantizar que la generación y el almacenamiento de las constancias continúen operando de manera independiente, aun si el servicio externo de correo deja de responder (derivado de RT2 y RT3).
2. **Idempotencia y consistencia en el procesamiento:** Prevenir el reenvío de notificaciones duplicadas para una misma constancia ante reintentos o faltas de confirmación del proveedor externo (derivado de RT4).

## Considered Options

- **Alternativa A:** Comunicación directa con recuperación de fallos (La aplicación intenta el envío inmediato tras el almacenamiento y registra los fallos para reintentar posteriormente).
- **Alternativa B:** Procesamiento asíncrono mediante eventos persistidos (La aplicación registra una notificación pendiente y un componente independiente procesa asíncronamente el envío al proveedor).

## Decision Outcome

Option chosen: **Alternativa B (Procesamiento asíncrono mediante eventos persistidos)**, porque desacopla el dominio principal (generación y almacenamiento de constancias) de una dependencia externa inherentemente volátil. Esto satisface directamente la restricción **RT2** (mantener la generación funcionando si falla el correo), facilita la cola confiable para **RT3** y centraliza la lógica de control para prevenir notificaciones duplicadas (**RT4**).

### Consequences

- Good, porque las operaciones centrales (generación y almacenamiento de constancias) nunca se bloquean ante la disponibilidad o latencia del servicio de correo externo.
- Good, porque la gestión de reintentos y la prevención de duplicados se aíslan en un trabajador asíncrono o componente de colas dedicado.
- Bad, porque introduce la complejidad asociada al procesamiento asíncrono (consistencia eventual, trabajadores en segundo plano o bucles de reintentos programados).
- Bad, porque la depuración de fallos en las notificaciones requiere rastrear registros asíncronos en lugar de códigos de respuesta HTTP directos y sincrónicos.

## Pros and Cons of the Options

### Alternativa A: Comunicación directa con recuperación de fallos
- Good, porque es más simple de implementar inicialmente sin requerir trabajadores en segundo plano o colas.
- Bad, porque los fallos temporales de red o las caídas del proveedor de correo bloquean o ralentizan directamente el flujo de transacciones de la aplicación, incrementando el acoplamiento.

### Alternativa B: Procesamiento asíncrono mediante eventos persistidos
- Good, porque aísla por completo al sistema principal de las interrupciones del servicio externo, cumpliendo de manera robusta con **RT2** y **RT3**.
- Bad, porque introduce complejidad arquitectónica al requerir estados de eventos/mensajes persistentes y lógica de ejecución en segundo plano.

## Confirmation

La correcta implementación y cumplimiento de esta decisión se comprobará mediante:
- Pruebas de integración simulando la caída total del proveedor externo de correo (mediante simulación o *mocking* de errores HTTP 500 o tiempos de espera agotados) para verificar que la constancia se almacena de forma correcta y se registra el evento pendiente.
- Pruebas de idempotencia que envíen múltiples veces el mismo evento de notificación hacia el componente procesador, validando que el proveedor externo reciba únicamente una petición válida por constancia.

## More Information
### Revisión 1. Cambio en el proveedor de notificaciones

* **Cambio recibido:** El proveedor externo modifica su interfaz de comunicación y cambia la forma en que reporta los errores de envío.
* **Impacto sobre la alternativa seleccionada:** El cambio se localiza exclusivamente en el adaptador o capa de integración del componente independiente. La lógica central de eventos persistidos y almacenamiento de constancias permanece intacta.
* **Mantenimiento de la decisión:** La decisión se mantiene vigente gracias al desacoplamiento provisto por la Alternativa B.
* **Consecuencia o comprobación actualizada:** Se deben actualizar las pruebas unitarias y de integración del adaptador del proveedor para reflejar el nuevo contrato de API.
