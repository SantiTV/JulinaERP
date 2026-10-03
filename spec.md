# SPEC.md — ERP Multicanal Sync Core

Especificación bajo **Spec-Driven Development (SDD)**. Es la fuente de verdad funcional: el código se deriva de aquí, no al revés. Los cambios de comportamiento se hacen primero en este archivo.

Estado: **borrador v0.2**. Los puntos marcados **[CONFIRMAR]** son decisiones pendientes del dueño del proyecto.

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
    "sku": { "type": "string", "minLength": 1 },
    "title": { "type": "string", "minLength": 1 },
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

**Cambios respecto a la idea inicial:** se añaden `lastSyncedAt` y `updatedAt` (diagnóstico de sincronización) y `minLength` en SKU/título.

### Tablas mínimas (PostgreSQL)
- `products(sku PK, title, global_stock CHECK >= 0, updated_at)`
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

## 8. Decisiones abiertas

| # | Tema | Opciones |
|---|---|---|
| D1 | Cola de trabajos | Redis+BullMQ vs tabla en Postgres |
| D2 | Política de asignación por canal | Stock completo vs porcentajes |
| D3 | Autenticación de la API interna | API key vs JWT |
| D4 | Manejo de sobreventa | Pausar + alertar vs rechazar |
| D5 | ORM / acceso a datos | SQL explícito vs Drizzle |
| D6 | Verificación de origen del webhook | Según documentación vigente de ML |

## 9. Plan de entrega sugerido

1. Esqueleto del proyecto + BD + migraciones + `GET /inventory`.
2. Lógica de stock atómica + libro de movimientos + tests de concurrencia.
3. Conector ML: OAuth y renovación de tokens.
4. Webhook + cola + idempotencia.
5. `POST /inventory/sync` + `GET /jobs/:id`.
6. Segundo canal (Shopify).