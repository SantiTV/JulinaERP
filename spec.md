# SPEC.md — ERP Multicanal Sync Core

Especificación bajo **Spec-Driven Development (SDD)**. Es la fuente de verdad funcional: el código se deriva de aquí, no al revés. Los cambios de comportamiento se hacen primero en este archivo.

Estado: **borrador v0.3** (añade modo dry-run, campos de precio y plan de importación desde Excel). Los puntos marcados **[CONFIRMAR]** son decisiones pendientes del dueño del proyecto.

---

## 1. Visión

- **Producto:** ERP Multicanal Sync Core.
- **Problema:** vender el mismo producto en varios canales sin sobrevender porque el stock está desincronizado.
- **Objetivo:** centralizar el stock por SKU único y propagarlo a Mercado Libre y otros canales de forma rápida y segura.
- **Usuarios:** operadores del negocio (consultan y corrigen inventario) y sistemas externos (webhooks).
- **Restricciones:** sin FastAPI; sin RAG; backend transaccional robusto.

### Fuera de alcance (v1)
- Facturación, contabilidad, compras a proveedores, logística/envíos.
- Interfaz web (v1 es solo API; el frontend se especifica aparte).
- Más de un canal conectado de verdad: **v1 conecta solo Mercado Libre**; los demás canales quedan en el modelo pero sin conector.
- **Importación desde Excel: diferida** (ver §10). No se implementa hasta disponer del archivo real.

## 2. Arquitectura

Tres módulos desacoplados:

| Módulo | Responsabilidad | Regla clave |
|---|---|---|
| **Core ERP** | Reglas de stock, SKU, persistencia | Única capa que escribe stock, siempre en transacción |
| **Connectors** | Hablar con APIs externas (OAuth, mapeo) | No conocen la BD; solo la interfaz `ChannelConnector` |
| **Sync Engine** | Cola, workers, reintentos, idempotencia | Procesa webhooks de forma asíncrona |

Flujo de una venta en ML:
1. ML envía notificación → `POST /webhooks/mercadolibre`.
2. La API valida, registra el evento (idempotencia), encola y responde `200` de inmediato.
3. Un worker consulta la orden a ML, descuenta stock de forma atómica y genera eventos de actualización.
4. Los conectores de los demás canales actualizan su `channelStock`.

## 3. Modelo de datos

Contrato lógico (el JSON Schema de la idea inicial se conserva, con las mejoras marcadas):

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "ERPProduct",
  "type": "object",
  "properties": {
    "sku": { "type": "string", "minLength": 1, "description": "Código del producto (el mismo que ya se usa en el Excel actual)" },
    "title": { "type": "string", "minLength": 1 },
    "price": { "type": "integer", "minimum": 0, "description": "Precio base en la unidad mínima de la moneda" },
    "currency": { "type": "string", "pattern": "^[A-Z]{3}$", "description": "ISO 4217, p. ej. COP" },
    "details": { "type": "string", "description": "Texto libre de detalles (viene del Excel)" },
    "sourceUrl": { "type": "string", "format": "uri", "description": "Link original del producto (viene del Excel)" },
    "globalStock": { "type": "integer", "minimum": 0 },
    "channels": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "channelName": { "enum": ["mercadolibre", "shopify", "woocommerce", "local"] },
          "externalId": { "type": "string" },
          "channelStock": { "type": "integer", "minimum": 0 },
          "status": { "enum": ["active", "paused", "out_of_stock"] },
          "lastSyncedAt": { "type": "string", "format": "date-time" }
        },
        "required": ["channelName", "channelStock", "status"]
      }
    },
    "updatedAt": { "type": "string", "format": "date-time" }
  },
  "required": ["sku", "title", "globalStock", "channels"]
}
```

**Cambios respecto a la idea inicial:** se añaden `lastSyncedAt` y `updatedAt` (diagnóstico de sincronización), `minLength` en SKU/título y los campos `price`, `currency`, `details` y `sourceUrl`, que provienen del inventario actual en Excel. El `sku` es el **código de producto que ya existe**: no se genera uno nuevo.

### Tablas mínimas (PostgreSQL)
- `products(sku PK, title, price, currency, details, source_url, global_stock CHECK >= 0, updated_at)`
- `channel_listings(sku FK, channel_name, external_id, channel_stock CHECK >= 0, status, last_synced_at)` con `UNIQUE(channel_name, external_id)`
- `stock_movements(id, sku, delta, reason, source_event_id, created_at)`: **libro de movimientos inmutable**; auditoría y base para reconciliar.
- `webhook_events(event_id UNIQUE, channel, payload, status, received_at)`: idempotencia.
- `channel_accounts(channel, account_id, access_token_enc, refresh_token_enc, expires_at)`: tokens cifrados.

### Reglas de stock
1. `globalStock` es el stock físico central. `channelStock` es la cantidad que se **publica** en cada canal.
2. Política de asignación por defecto: **[CONFIRMAR]** (a) publicar el `globalStock` completo en todos los canales, o (b) repartir por porcentajes por canal.
3. Una venta descuenta `globalStock` y genera un `stock_movement`; luego se recalcula `channelStock` y se propaga.
4. `globalStock` nunca queda negativo. Si llega una venta sin stock suficiente: registrar movimiento marcado `oversold`, poner a 0, pausar publicaciones y alertar. **[CONFIRMAR]**

## 4. API (`/api/v1`)

Autenticación de los endpoints internos: API key o JWT. **[CONFIRMAR]** (la idea inicial no lo definía y es necesario).

### 4.1 `GET /inventory`
Lista de productos según el schema. Parámetros opcionales: `page`, `pageSize`, `sku`, `channel`.
- `200`: `{ "data": [ERPProduct], "page": 1, "pageSize": 50, "total": 120 }`

### 4.2 `POST /inventory/sync`
Fuerza reconciliación con un canal.
```json
{ "sku": "PROD-001", "targetChannel": "mercadolibre" }
```
- `sku` opcional: si se omite, sincroniza todo el catálogo (trabajo en segundo plano).
- Acepta `?dryRun=true` (ver §8).
- `202 Accepted` con `{ "jobId": "..." }`; el estado se consulta en `GET /jobs/:id`.
- `400` payload inválido · `404` SKU inexistente · `409` ya hay una sincronización en curso para ese SKU/canal.

### 4.3 `POST /webhooks/mercadolibre`
Endpoint público. Recibe notificaciones (topic, resource, user_id...).
- Responde `200` rápido, sin procesar en línea.
- Ignora topics no soportados (responde `200` igualmente).
- Descarta duplicados por `event_id`/`_id`.
- Hay que verificar el origen de las notificaciones (IP/firma) según la documentación vigente de ML. **[CONFIRMAR método]**

### 4.4 `GET /jobs/:id` (nuevo)
Estado de una sincronización (`queued | running | done | failed`) con resumen de errores.

### 4.5 `GET /health` (nuevo)
Estado de BD, cola y validez del token de ML.

## 5. Mercado Libre: autenticación

- OAuth 2.0 (authorization code). Almacenar `access_token` y `refresh_token` cifrados.
- Renovar de forma proactiva antes de expirar y reactiva ante `401`.
- El refresh token es de un solo uso: persistir el nuevo atómicamente y serializar renovaciones.
- Fallo de renovación → marcar la cuenta como `needs_reauth` y alertar.
- Verificar límites y detalles vigentes en la documentación oficial de ML antes de implementar.

## 6. Requisitos no funcionales

- **Consistencia:** ninguna venta simultánea puede producir stock negativo ni pérdida de descuentos (test de concurrencia obligatorio).
- **Latencia:** el webhook responde en < 500 ms; propagación a canales objetivo < 30 s en condiciones normales. **[CONFIRMAR]**
- **Resiliencia:** reintentos con backoff exponencial; cola de errores (dead-letter) tras N intentos.
- **Observabilidad:** logs estructurados con `sku`, `eventId`, `channel`; nunca registrar tokens.
- **Seguridad:** secretos solo por variables de entorno; tokens cifrados en reposo.

## 7. Criterios de aceptación (v1)

1. Dado un SKU con `globalStock = 5`, cuando llegan 2 webhooks de venta de 3 unidades al mismo tiempo, entonces el stock nunca es negativo y la segunda venta se trata según la regla de sobreventa.
2. Dado el mismo webhook enviado dos veces, el stock se descuenta una sola vez.
3. Dado un access token caducado, la siguiente llamada a ML renueva el token y reintenta sin intervención humana.
4. `POST /inventory/sync` con un SKU inexistente devuelve `404`.
5. Cada cambio de `globalStock` genera exactamente un `stock_movement` trazable al evento de origen.
6. El webhook responde `200` aunque el procesamiento posterior falle (y el fallo queda registrado y reintentable).
7. Con `DRY_RUN=true`, ninguna llamada de escritura llega a un canal externo y la BD queda idéntica antes y después (verificable en test).
8. Una operación en dry-run devuelve un informe de acciones con `executed: false` para cada escritura simulada.
9. `?dryRun=true` activa el modo en una petición, pero `?dryRun=false` no desactiva un `DRY_RUN` global.

## 8. Modo dry-run

Permite ejecutar el sistema completo **sin ninguna escritura externa ni persistente**, y obtener un informe de lo que habría hecho.

### 8.1 Comportamiento
| Operación | En dry-run |
|---|---|
| Lecturas a APIs de canales (consultar ítem, orden) | Se ejecutan de verdad |
| Escrituras a canales (stock, precio, pausar/activar) | **No se ejecutan**; se registran como acción simulada |
| Escrituras en BD (stock, movimientos, eventos) | **No se confirman**: se ejecutan dentro de una transacción con `ROLLBACK` al final |
| Renovación de tokens OAuth | Se permite (es necesaria para leer); es la única escritura excepcionalmente permitida |
| Colas / reintentos | Los trabajos se procesan en línea y no se encolan |

### 8.2 Activación
- Variable de entorno `DRY_RUN=true|false` (global).
- Parámetro `?dryRun=true` por petición (solo puede **activar**, nunca desactivar un `DRY_RUN` global).
- **Valor por defecto:** `true` en desarrollo y en cualquier entorno donde `DRY_RUN` no esté definido. En producción hay que ponerlo en `false` de forma explícita. **[CONFIRMAR]**
- Al arrancar, el sistema registra en el log y en `GET /health` si está en dry-run.

### 8.3 Implementación (requisitos)
1. Un único punto de control: el patrón *decorator* `DryRunConnector` implementa `ChannelConnector`, delega las lecturas al conector real y registra las escrituras sin ejecutarlas.
2. Ningún código fuera de los conectores llama directamente a APIs externas de escritura.
3. El core ejecuta su lógica de stock con una unidad de trabajo (`unitOfWork`) que hace `rollback` cuando `dryRun` es verdadero.
4. Cada operación produce un **informe de acciones**:

```json
{
  "dryRun": true,
  "actions": [
    {
      "type": "channel.updateStock",
      "sku": "PROD-001",
      "channel": "mercadolibre",
      "externalId": "MLA123456789",
      "from": 10,
      "to": 7,
      "executed": false
    }
  ],
  "warnings": [],
  "errors": []
}
```

5. Las respuestas HTTP de operaciones en dry-run incluyen `"dryRun": true` y el informe.

### 8.4 Usos previstos
- Validar la importación del Excel antes de escribir nada.
- Primera conexión con la cuenta real de Mercado Libre.
- **Modo sombra:** recibir webhooks reales, calcular el resultado y compararlo con lo ocurrido, sin enviar cambios.

## 9. Decisiones abiertas

| # | Tema | Opciones |
|---|---|---|
| D1 | Cola de trabajos | Redis+BullMQ vs tabla en Postgres |
| D2 | Política de asignación por canal | Stock completo vs porcentajes |
| D3 | Autenticación de la API interna | API key vs JWT |
| D4 | Manejo de sobreventa | Pausar + alertar vs rechazar |
| D5 | ORM / acceso a datos | SQL explícito vs Drizzle |
| D6 | Verificación de origen del webhook | Según documentación vigente de ML |
| D7 | Moneda y manejo de precio por canal | Una moneda global vs precio por canal |
| D8 | `DRY_RUN` por defecto en producción | `true` (seguro) vs `false` |

## 10. Importación desde Excel (ACCIÓN DIFERIDA)

> **Estado: pendiente.** Se implementará cuando se disponga del archivo real. Hasta entonces, el sistema arranca con la BD vacía y se pueden cargar productos de prueba con fixtures.

**Situación actual:** el inventario vive en un Excel con las columnas: nombre, precio, link, detalles, **cantidad en stock** y **código de producto**.

### 10.1 Mapeo previsto (se confirma al ver el archivo)
| Columna Excel | Campo del ERP |
|---|---|
| Código de producto | `sku` |
| Nombre | `title` |
| Precio | `price` (+ `currency`) |
| Cantidad en stock | `globalStock` |
| Detalles | `details` |
| Link | `sourceUrl` y, si es un link de ML, `channels[].externalId` |

### 10.2 Requisitos
1. Comando `npm run import:excel -- <archivo>` (o `POST /inventory/import`), **idempotente**: upsert por `sku`.
2. **Siempre se ejecuta primero en dry-run** y produce un informe de validación (sin escribir). Solo se aplica con confirmación explícita.
3. Validaciones: SKU vacío o duplicado, precio no numérico o negativo, cantidad negativa o no entera, link ilegible, filas completamente vacías.
4. Las filas inválidas **no bloquean** las válidas: se informan y se omiten.
5. La importación crea un `stock_movement` inicial por producto (`reason = "initial_import"`).
6. Tras el import, **la base de datos es la fuente de verdad**; el Excel deja de editarse como inventario.
7. Lectura de `.xlsx` con `exceljs` o exportando antes a CSV. **[CONFIRMAR]**

### 10.3 Tareas pendientes cuando haya Excel
- Revisar los nombres reales de columnas y el formato de números y precios.
- Guardar 5–10 filas sin datos sensibles en `docs/samples/`.
- Verificar el patrón de los links de ML para extraer el `externalId`.
- Decidir qué hacer con "detalles" si contiene variaciones (talla, color).
- Registrar una moneda por defecto (D7).

## 11. Plan de entrega sugerido

1. Esqueleto del proyecto + BD + migraciones + `GET /inventory`.
2. Lógica de stock atómica + libro de movimientos + tests de concurrencia.
3. Conector ML: OAuth y renovación de tokens.
4. Webhook + cola + idempotencia.
5. `POST /inventory/sync` + `GET /jobs/:id`.
6. Modo dry-run transversal (`DryRunConnector` + unidad de trabajo con rollback), idealmente desde el paso 2.
7. Importación desde Excel (cuando haya archivo) y primera conexión real en dry-run.
8. Segundo canal (Shopify o WooCommerce).