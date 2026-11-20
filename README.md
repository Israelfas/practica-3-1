# Práctica 3.1: comunicación asíncrona con RabbitMQ

Backend de Correos – Productos Dañados. Cuando un paquete llega dañado hay que avisarle al remitente. Ese aviso tarda, así que la API ya no lo hace mientras el cliente espera: lo deja en una cola de RabbitMQ y un consumidor aparte lo procesa.

```
Cliente → API Laravel (productor) → RabbitMQ: correos.notificaciones.danio → Consumidor → tabla notificaciones
             ↳ responde 202 al instante                     ↳ si falla: correos.notificaciones.danio.fallidas
```

## Piezas

| Pieza | Dónde |
| --- | --- |
| Broker | `docker-compose.yml` (RabbitMQ 3.13 con panel de administración) |
| Productor | `POST /api/notificaciones` en `NotificacionController@store` y `app/Mensajeria/RabbitMqPublicador.php` |
| Colas | `correos.notificaciones.danio` (durable) y `correos.notificaciones.danio.fallidas`, en `ConexionRabbitMq.php` |
| Consumidor | `php artisan notificaciones:consumir` (`ConsumirNotificaciones.php` y `ProcesadorNotificacion.php`) |
| Versión síncrona para comparar | `POST /api/notificaciones/sincrona` |
| Seguimiento | `GET /api/notificaciones?correlation_id=...` y `GET /api/notificaciones/{message_id}` |
| Pruebas | `tests/Feature/NotificacionesTest.php` |

## Mensaje

```json
{
  "message_id": "6062cc29-7242-415d-9072-7effa3fe3e61",
  "correlation_id": "danio-001",
  "tipo": "notificacion_producto_danado",
  "ocurrido_en": "2026-11-20T07:46:18Z",
  "payload": { "product_id": 1, "sku": "EC100000001EC", "canal": "email" }
}
```

`message_id` identifica cada mensaje y sirve para no procesarlo dos veces. `correlation_id` sigue toda la operación: viene en la cabecera `X-Correlation-ID` o se genera, y aparece en la respuesta, en el mensaje, en los logs y en la tabla.

## Cómo ejecutarlo

Se necesita PHP 8.2+ con la extensión `sockets`, Composer y Docker.

```bash
composer install
cp .env.example .env          # completar RABBITMQ_PASSWORD y DEMO_PASSWORD con valores de prueba
php artisan key:generate
php artisan migrate --seed
docker compose up -d          # levanta RabbitMQ
php artisan serve             # terminal 1: API (productor)
php artisan notificaciones:consumir   # terminal 2: consumidor
```

Para ver que la cola existe:

```bash
docker exec correos-rabbitmq rabbitmqctl list_queues name messages_ready consumers
```

## Uso

1. `POST /api/login` para obtener el token.
2. `POST /api/notificaciones` con `Authorization: Bearer <token>`, `X-Correlation-ID: danio-001` y el cuerpo `{"product_id": 1, "canal": "email"}`. Responde **202**.
3. El consumidor muestra `Recibido` y `Procesado` con el mismo `correlation_id`, y la notificación pasa a `enviada`.

Si el consumidor está apagado, el mensaje queda en la cola hasta que se vuelve a encender. Si falla al procesarlo, no se confirma (`nack`) y pasa a la cola de fallidos.

```bash
php artisan test --filter=NotificacionesTest
```
