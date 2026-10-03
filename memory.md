# MEMORY.md — Estado del proyecto

> Máximo ~50 líneas. Resume o elimina lo que ya no aporte. Sin datos sensibles.
> Reglas permanentes → proponer moverlas a `AGENTS.md`.

## Estado actual
- Fase: **pre-implementación**. Solo existen `SPEC.md`, `AGENTS.md` y `MEMORY.md`.
- Último paso hecho: spec v0.3 (añade modo dry-run, campos de precio e importación diferida).
- Siguiente paso: resolver D1–D8 de `SPEC.md` §9 y ejecutar `/init`; luego, tarea 1 del plan de entrega (esqueleto + BD).

## Decisiones tomadas (con su porqué)
- **Sin FastAPI ni RAG**: decisión del dueño; el foco es un backend transaccional robusto.
- **Node.js + TypeScript + Express + PostgreSQL** (propuesto): la consistencia del stock requiere transacciones ACID. Pendiente de confirmar.
- **v1 conecta solo Mercado Libre**: reducir alcance; los demás canales quedan en el modelo.
- **Libro de movimientos (`stock_movements`) inmutable**: permite auditar y reconciliar discrepancias.
- **Webhooks asíncronos**: responder rápido y procesar en cola para no perder notificaciones.
- **Dry-run como requisito de v1** (`SPEC.md` §8): decorator sobre el conector + rollback en BD; permite probar con datos reales sin riesgo.
- **`sku` = código de producto que ya existe en el Excel**; la cantidad en stock también viene del Excel.

## Errores a evitar
- Calcular stock en la aplicación y luego escribirlo (condición de carrera). Usar `UPDATE` atómico.
- Perder el refresh token de ML al renovar: guardar el nuevo de forma atómica.
- Procesar un webhook duplicado dos veces: comprobar idempotencia primero.

## Pendiente / abierto
- D1 cola · D2 asignación por canal · D3 autenticación API · D4 sobreventa · D5 ORM · D6 verificación del webhook · D7 moneda · D8 `DRY_RUN` por defecto en producción.
- **Acción diferida:** importar el Excel (nombre, precio, link, detalles, cantidad, código). No hay archivo disponible aún; ver `SPEC.md` §10.3 para las tareas cuando se tenga.