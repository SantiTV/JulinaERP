
### Modelo de datos central
´´´json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "ERPProduct",
  "type": "object",
  "properties": {
    "sku": { "type": "string", "description": "Identificador único universal (SKU)" },
    "title": { "type": "string" },
    "globalStock": { "type": "integer", "minimum": 0 },
    "channels": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "channelName": { "type": "string", "enum": ["mercadolibre", "shopify", "woocommerce", "local"] },
          "externalId": { "type": "string", "description": "ID del producto en la plataforma externa (ej. MLA123456)" },
          "channelStock": { "type": "integer", "minimum": 0 },
          "status": { "type": "string", "enum": ["active", "paused", "out_of_stock"] }
        },
        "required": ["channelName", "channelStock", "status"]
      }
    }
  },
  "required": ["sku", "title", "globalStock", "channels"]
}
´´´
### Enpoints clave
GET /api/v1/inventory: Consulta el estado del inventario unificado en tiempo real de todos los canales.

POST /api/v1/inventory/sync: Fuerza una sincronización manual o programada entre el ERP y Mercado Libre.

POST /api/v1/webhooks/mercadolibre: Endpoint que recibe notificaciones de eventos (ventas, cambios de stock) directo desde Mercado Libre para disparar la actualización en cadena.

# Fase 2: Arquitectuera de componenetes del ERP
Para un ERP robusto, dividiremos la construcción en microservicios o módulos lógicos bien acotados:

1.Módulo Core (ERP Backend):Node.js / Python.Centraliza la lógica de negocio, la base de datos de productos y el motor de reglas de inventario (si se vende 1 en Mercado Libre, restar 1 en la base central y notificar a los demás canales).

2.Módulo de Adaptadores (Connectors):External APIs.Capas independientes encargadas de hablar el idioma de cada plataforma. El adaptador de Mercado Libre maneja la autenticación OAuth 2.0 y mapea sus items con el SKU del ERP.

3.Motor de Eventos y Webhooks:Redis / BullMQ.Evita bloqueos procesando las ventas y actualizaciones de stock en segundo plano de manera asíncrona mediante colas de mensajes.

4.Panel ERP (Frontend):Tailwind CSS / Dashboard.Interfaz gráfica modular con vistas de inventario global, alertas de stock crítico, historial de sincronización y registro de errores (logs).