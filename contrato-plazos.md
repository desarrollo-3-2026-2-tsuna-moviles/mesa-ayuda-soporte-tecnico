# Contrato entre tiquetes-service y plazos-service

> Estado: **BORRADOR** — el equipo lo revisa y lo congela en la semana 1.
> Cualquier cambio después se registra en la bitácora.

## 1. Llamada síncrona (HTTP)

Se usa al **crear** un reporte (RF-03).

```
GET http://plazos-service:8082/api/compromisos?categoria={CATEGORIA}&prioridad={PRIORIDAD}
```

**Respuesta 200**

```json
{
  "categoria": "HARDWARE",
  "prioridad": "ALTA",
  "horasRespuesta": 8
}
```

**Respuesta 404**: no existe compromiso para esa combinación.

plazos-service solo devuelve las horas del compromiso. La fecha de vencimiento la
calcula tiquetes-service: `vencimiento = fechaCreacion + horasRespuesta`.

### Valores permitidos (propuesta)

| Categoría | Prioridades |
|-----------|-------------|
| HARDWARE, SOFTWARE, RED, ACCESOS, OTRO | BAJA, MEDIA, ALTA, CRITICA |

## 2. Qué hace tiquetes-service si plazos-service no responde (RQ-01)

1. La llamada tiene **timeout de 1 s de conexión y 1 s de lectura**, declarado en `application.yml`.
2. Si falla, el reporte **se guarda igual** y devuelve su número.
   - `vencimiento = null`
   - `estadoPlazo = PENDIENTE`
3. Una tarea programada reintenta cada 10 s los reportes con `estadoPlazo = PENDIENTE`.
4. Cuando plazos-service responde, se completa el vencimiento y queda `estadoPlazo = CALCULADO`.

Con esto la respuesta al usuario no supera los 2 s y el plazo se completa antes de 60 s después de que plazos-service vuelva.

## 3. Evento asíncrono (RF-06)

Publicado por tiquetes-service en cada cambio de estado y consumido por plazos-service.

- Exchange: `tiquetes.eventos` (topic)
- Routing key: `tiquete.cambio-estado`
- Cuerpo:

```json
{
  "tiqueteId": 15,
  "estadoAnterior": "EN_PROGRESO",
  "estadoNuevo": "RESUELTO",
  "categoria": "HARDWARE",
  "prioridad": "ALTA",
  "vencimiento": "2026-10-20T14:00:00",
  "fechaCambio": "2026-10-19T16:30:00"
}
```

plazos-service saca del tablero los reportes en estado `RESUELTO` o `CERRADO`.
