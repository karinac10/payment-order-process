# payment-order-process

Capa de proceso del sistema de transferencias interbancarias. Gestiona la idempotencia de las órdenes de pago, orquesta la llamada al core bancario y persiste el estado de cada transferencia en ObjectStore.

---

## Endpoints

| Método | Ruta | Descripción |
|--------|------|-------------|
| `POST` | `/payment-order/PaymentOrder/Initiate` | Procesa una orden de pago nueva o responde con el resultado cacheado (replay). |
| `GET`  | `/payment-order/PaymentOrder/{id}/Retrieve` | Recupera el estado de una orden por su ID. |

Puerto local: **8082**

---

## Arquitectura de tres capas

```
transfers-experience-api  :8081
     │  BIAN PaymentOrderProcedure
     ▼
payment-order-process     :8082   ← este servicio
     │  JSON interno
     ▼
core-transfers-system     :8083
```

---

## Decisiones de diseño

**Idempotencia con SHA-1 del cuerpo.** Antes de procesar, se calcula el hash SHA-1 del payload y se compara con el hash almacenado para esa `X-Idempotency-Key`:
- Misma key + mismo hash → **replay**: devuelve la respuesta cacheada sin llamar al core.
- Misma key + hash diferente → **409** `IDEMPOTENCIA_CONFLICTO`.
- Key nueva → procesamiento normal.

El registro de idempotencia y la orden se almacenan en ObjectStore usando dos claves distintas: `idempotencyKey` para el control de replay y `orderId` para el Retrieve.

**Process no conoce el modelo móvil.** Recibe y devuelve `PaymentOrderProcedure` de BIAN. La traducción móvil ↔ BIAN ocurre exclusivamente en Experience. Esto permite que Process sea reutilizable por cualquier canal.

**System devuelve siempre 200.** Los códigos del core (`00` aprobado, `51` saldo insuficiente, `62` cuenta bloqueada) vienen en el body como `codigoCore`. Solo el código `91` (core no disponible) provoca un error HTTP que Process propaga como `HTTP:CONNECTIVITY`. Process interpreta el `codigoCore` y construye el `PaymentInstructionStatusType` (`Completed` / `Rejected`) de la respuesta BIAN.

**Respuesta BIAN con headers de referencia.** La respuesta al POST incluye los headers `X-Payment-Order-Id` (UUID de la orden) y `Location` apuntando al endpoint Retrieve. Experience los usa para construir la respuesta al cliente móvil.

---

## Cómo ejecutar localmente

**Requisitos:** Mule 4.12.2 EE, Java 17, Maven 3.9.x. Iniciar primero `core-transfers-system` (:8083).

---

## Pipeline CI/CD

Mismo esquema que las otras capas: build único → GitHub Release inmutable → dev → qa → prod con aprobación manual. Cobertura MUnit ≥ 80 % requerida.

Cada capa tiene su propio repositorio y pipeline. Un cambio en Process no recompila ni redespliegue Experience ni System.
