# AGENTS.md — ERP Multicanal Sync Core

Backend transaccional que centraliza el inventario por SKU único y lo sincroniza en tiempo casi real entre Mercado Libre y otros canales (Shopify, WooCommerce, tienda local) para evitar sobreventas. Para uso interno del equipo de operaciones del negocio.

> Fuente de verdad funcional: `SPEC.md`. Si algo de este archivo contradice a `SPEC.md`, pregunta antes de actuar.

## Stack y estructura
- Node.js 22 LTS + TypeScript (modo `strict`) + Express 5. **[CONFIRMAR]**
- PostgreSQL 16 (única fuente de verdad del stock). Acceso con `pg` + SQL explícito o Drizzle. **[CONFIRMAR]**
- Cola de trabajos para webhooks y sincronización: BullMQ + Redis, o tabla `jobs` en Postgres si se quiere evitar Redis. **[CONFIRMAR]**
- Validación con Zod; tests con Vitest + Supertest.
- **Prohibido:** FastAPI y cualquier componente RAG.

```
src/
  api/          # Rutas HTTP y controladores (solo validan y delegan)
  core/         # Lógica de negocio: reglas de stock, reservas, SKU
  db/           # Migraciones, repositorios y transacciones
  connectors/   # Un subdirectorio por canal (mercadolibre/, shopify/...)
    mercadolibre/  # Cliente HTTP, OAuth y mapeo de payloads. NUNCA toca la BD
  sync/         # Workers, cola, reintentos, idempotencia
  config/       # Variables de entorno validadas
specs/          # Specs por funcionalidad (ver SPEC.md)
SPEC.md  AGENTS.md  MEMORY.md
```

## Comandos
```bash
npm install            # instalar dependencias
npm run dev            # servidor en desarrollo con recarga
npm run build          # compilar TypeScript
npm run lint           # ESLint + Prettier (comprobación)
npm run typecheck      # tsc --noEmit
npm test               # Vitest (unitarios + integración)
npm run db:migrate     # aplicar migraciones
```
> Ajusta esta sección en cuanto se cree `package.json`. Los comandos deben ser exactos y copiables.

## Convenciones
- Código, nombres de variables y commits en **inglés**; comentarios, documentación y textos de error para el usuario en **español**.
- Un conector por canal, todos implementando la misma interfaz `ChannelConnector` (archivo de referencia: `src/connectors/channel.ts`, cuando exista).
- Los controladores no contienen lógica de negocio ni SQL.
- Dinero y cantidades: enteros. Fechas: UTC en ISO 8601.
- Errores tipados (`AppError`) con código estable; nada de `catch` vacíos.

## Reglas de dominio / trampas conocidas
- **El stock nunca se modifica con "leer, calcular, escribir".** Usar siempre operaciones atómicas dentro de una transacción (`UPDATE ... SET stock = stock - $n WHERE stock >= $n`) para evitar condiciones de carrera con ventas simultáneas.
- **Webhooks de Mercado Libre:** responder `200` de inmediato y procesar en segundo plano; ML espera respuesta rápida y reintenta si no la recibe. El payload es solo una notificación (topic + resource): hay que consultar el recurso a la API para obtener el detalle.
- **Idempotencia:** la misma notificación puede llegar más de una vez. Registrar el ID de evento y descartar duplicados antes de tocar el stock.
- **Tokens OAuth de ML:** el access token caduca (~6 h) y el refresh token es de un solo uso: al renovar hay que guardar el **nuevo** refresh token de forma atómica, o se pierde el acceso. Serializar las renovaciones (un solo refresh concurrente por cuenta). Verificar detalles vigentes en la documentación oficial.
- `globalStock` no es la suma de `channelStock`: es el stock físico central; cada canal recibe una asignación. Definición exacta en `SPEC.md` §3.
- `externalId` es obligatorio salvo en el canal `local`.
- **Dry-run:** toda escritura hacia un canal externo pasa por el conector y debe respetar `DRY_RUN`. Nunca llamar directamente a una API de escritura de un canal. En dry-run la BD se revierte (`rollback`) y las acciones se registran en el informe (ver `SPEC.md` §8).
- **`sku` = código de producto existente** (del Excel actual); no inventar ni regenerar SKUs.
- La importación desde Excel está **diferida** (`SPEC.md` §10): no implementarla ni inventar el formato del archivo hasta que exista una muestra en `docs/samples/`.

## Forma de trabajar
- **Planifica antes de codificar** cuando el cambio toque más de 2 archivos, el modelo de datos, la lógica de stock, OAuth o un conector. Usa el formato de plan de `PLAN.md`.
- Cambios pequeños y revisables: una funcionalidad o corrección por vez.
- Primero la spec (`specs/<feature>.md`), luego el plan, luego el código y los tests.
- Al terminar, explica en pocas líneas: qué cambió, qué verificaste y qué queda pendiente.

## Límites
- ✅ Siempre: escribir o actualizar tests de cualquier cambio en lógica de stock.
- ✅ Siempre: usar transacciones para cualquier escritura que afecte stock o tokens.
- ✅ Siempre: validar con Zod toda entrada externa (HTTP y webhooks).
- ✅ Siempre: probar con `DRY_RUN=true` antes de cualquier ejecución contra una cuenta real.
- ✅ Siempre: actualizar `MEMORY.md` al terminar cada tarea.
- ⚠️ Pregunta antes: añadir dependencias, crear archivos fuera de la estructura anterior, cambiar el esquema de la BD o el contrato de la API, o añadir un canal nuevo.
- 🚫 Nunca: desactivar `DRY_RUN` en un entorno compartido o de producción sin que lo pida el dueño del proyecto; usar FastAPI o RAG; guardar claves o tokens en el repositorio o en logs; editar migraciones ya aplicadas; hacer llamadas a APIs reales de producción desde los tests; modificar `.env` real.

## Verificación
Antes de dar una tarea por terminada, ejecutar y reportar el resultado de:
1. `npm run typecheck && npm run lint`
2. `npm test` (incluido un test de concurrencia si se tocó stock)
3. Para webhooks/conectores: probar con payloads de ejemplo (fixtures), nunca contra producción. Verificar con un test que en dry-run no sale ninguna escritura externa y la BD no cambia.
4. Confirmar que los criterios de aceptación de la spec correspondiente se cumplen.

## Memoria
- Al empezar, lee `MEMORY.md` para conocer el estado del proyecto y las decisiones tomadas.
- Al terminar una tarea, actualízalo: estado actual, decisiones importantes (con su porqué) y errores a evitar.
- Mantenlo breve (máximo ~50 líneas): resume o elimina lo que ya no aporte.
- Si algo se convierte en una regla permanente, propón moverlo a `AGENTS.md` en lugar de dejarlo en la memoria.
- No guardes nunca datos sensibles (claves, tokens, datos personales).