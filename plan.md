Quiero implementar el descuento de stock cuando llega una venta de Mercado Libre:
al procesar el evento, reducir globalStock y registrar el movimiento.

Antes de escribir código, prepárame un plan con:
1. Cómo vas a hacer el descuento de forma atómica y idempotente, respetando SPEC.md §3 y
   las reglas de dominio de AGENTS.md (sin leer-calcular-escribir, sin descontar dos veces
   el mismo evento).
2. Qué archivos vas a crear o modificar (core/, db/, sync/) y qué cambia en cada uno.
3. Los casos límite y las dudas que debo decidir yo antes de empezar: ventas simultáneas,
   stock insuficiente (sobreventa, D4), evento duplicado, SKU no mapeado a un externalId,
   orden cancelada después de descontar.
4. Qué tests escribirías (incluido uno de concurrencia) y cómo verificarás los criterios
   de aceptación 1, 2 y 5 de SPEC.md §7.
5. Qué actualizarías en SPEC.md, AGENTS.md y MEMORY.md.

No toques ningún archivo hasta que apruebe el plan.