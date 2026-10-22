# Nicho del Proyecto

## Nicho elegido

Barbería / salón de belleza.

## Justificación de la elección

- El tenant es el salón. Entidades por tenant: servicios, citas, clientes.

- Dominio acotado a 3 entidades con CRUD completo, sin pagos reales ni integraciones externas obligatorias.

- Entidades claras y mapeables 1:1 a tablas por schema: servicios, citas, clientes.

- Idempotencia natural: crear cita, crear servicio y crear cliente son operaciones reales donde un doble clic o reintento duplicaría datos. Casos de uso forzados evitados.

- Rate limiting con sentido: login y endpoints POST de creación. No parece artificial en sustentación.

- Multitenancy demostrable: un salón no puede ver servicios, citas ni clientes de otro salón. Aislamiento evidente sin ambigüedad.

- Bajo coste de dominio: permite invertir tiempo en multitenancy, idempotencia y rate limiting, que concentran la mayor parte de la rúbrica (25 + 15 + 15 = 55 puntos).

- Sustentación defendible: el flujo de creación de schema, resolución de tenant por JWT y operaciones idempotentes se explica en vivo sin depender de reglas de negocio frágiles.

- Puntos opcionales fáciles de justificar: IA para resumen de citas disponibles o recomendación de horarios; n8n para notificar nuevas citas o cambios de servicio.
