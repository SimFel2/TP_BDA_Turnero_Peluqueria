# TPI Backend de Aplicaciones 2026 — Dominio de Peluquería y Fidelización

Sistema backend basado en microservicios con **Java 21** y **Spring Boot 3.3**, diseñado para gestionar la reserva de turnos bajo demanda sincronizada en tiempo real con **Google Calendar API** y administrar un programa de fidelización por puntos.

---

## 1. Arquitectura de Microservicios y Componentes (`ARQ-01`, `INF-01`)

El proyecto está organizado como un **proyecto Maven multi-módulo** con separación estricta de bases de datos (*Database per Service*):

| Módulo / Contenedor | Puerto | Responsabilidad en el Dominio |
| :--- | :---: | :--- |
| **`peluqueria-commons`** | *(Librería `.jar`)* | Contratos DTO inter-servicio (`CanjePuntos*`, `AcreditacionPuntos*`, `ReintegroPuntos*`), estructura estándar `ErrorResponseDTO`, convertidor de roles JWT de Keycloak y `EmailNotificationService` no bloqueante (`RN-06`). **No comparte entidades JPA ni repositorios.** |
| **`api-gateway`** | `8080` | Punto de entrada único (Spring Cloud Gateway). Valida firma y expiración del JWT contra Keycloak y enruta `/api/turnos/**`, `/api/peluqueros/**`, `/api/servicios/**` hacia `ms-turnos` y `/api/premios/**` hacia `ms-premios`. |
| **`ms-turnos`** | `8081` | Gestiona `Peluquero`, `Servicio` y `Turno` en `db-turnos` (`5432`). Aplica reglas de no solapamiento (`RN-01`), consulta y agenda eventos en Google Calendar (`RN-02`), y coordina canjes, acreditaciones y reintegros con `ms-premios`. |
| **`ms-premios`** | `8082` | Gestiona `Premio`, `CuentaPuntos` y `MovimientoPuntos` en `db-premios` (`5433`). Procesa canjes (`RN-03`), acreditaciones únicas por turno completado (`RN-04`) y reintegros por cancelación (`RN-05`). |
| **`keycloak`** | `8085` | Proveedor de identidad OpenID Connect con el realm `peluqueria-realm` pre-importado (`ROLE_CLIENTE` y `ROLE_ADMIN`). |
| **`mailpit`** | `1025` / `8025` | Servidor SMTP de pruebas (`1025`) con bandeja web en `http://localhost:8025` para verificar los correos enviados (`RN-06`). |

---

## 2. Matriz de Seguridad y Control de Acceso (`SEG-01`)

| Microservicio | Operación / Endpoint | `ROLE_CLIENTE` | `ROLE_ADMIN` | Sin Token |
| :--- | :--- | :---: | :---: | :---: |
| `ms-turnos` | `GET /api/peluqueros`, `GET /api/servicios` | Permitido (`200`) | Permitido (`200`) | Denegado (`401`) |
| `ms-turnos` | `POST / PUT /api/peluqueros`, `/api/servicios` | Denegado (`403`) | Permitido (`201`/`200`) | Denegado (`401`) |
| `ms-turnos` | `POST /api/turnos`, `PATCH /api/turnos/{id}/cancelar` *(propio)* | Permitido (`201`/`200`) | Permitido (`201`/`200`) | Denegado (`401`) |
| `ms-turnos` | `PATCH /api/turnos/{id}/completar` | Denegado (`403`) | Permitido (`200`) | Denegado (`401`) |
| `ms-premios` | `GET /api/premios`, `GET /api/premios/mi-cuenta` | Permitido (`200`) | Permitido (`200`) | Denegado (`401`) |
| `ms-premios` | `POST / PUT /api/premios` | Denegado (`403`) | Permitido (`201`/`200`) | Denegado (`401`) |

---

## 3. Reglas de Negocio y Procesamiento (`NEG-02`)

| Código | Regla de Negocio | Entradas | Condición de Control / Procesamiento | Resultado Esperado (Éxito vs. Rechazo) |
| :--- | :--- | :--- | :--- | :--- |
| **`RN-01`** | Horario laboral y no solapamiento interno | `peluqueroId`, `fechaHoraInicio`, `duracionMinutos` | El rango `[inicio, fin]` debe ser futuro, estar dentro de la franja del salón (`09:00` a `20:00` hs) y no solaparse con ningún turno `RESERVADO` del mismo peluquero en `db-turnos`. | **Éxito:** Avanza a validación externa.<br>**Rechazo:** `400 Bad Request` (fuera de horario) o `409 Conflict` (solapamiento). |
| **`RN-02`** | Disponibilidad en Google Calendar | `emailCalendar`, `fechaHoraInicio`, `fechaHoraFin` | La consulta de eventos del peluquero en Google Calendar para `[inicio, fin]` debe devolver disponibilidad libre. | **Éxito:** Horario validado y agendado.<br>**Rechazo:** `409 Conflict` indicando compromiso externo en Google Calendar. |
| **`RN-03`** | Suficiencia de puntos para canje | `clienteId`, `premioId` | El premio debe estar `activo` y `CuentaPuntos.saldoPuntos >= Premio.puntosRequeridos`. Se calcula `precioFinal = precioBase * (1 - porcentajeDescuento / 100)`. | **Éxito:** Descuenta puntos y aplica el descuento.<br>**Rechazo:** `422 Unprocessable Entity` por saldo insuficiente, abortando la reserva. |
| **`RN-04`** | Transición válida y acreditación única | `turnoId`, `estadoActual` | Solo un turno en estado `RESERVADO` puede pasar a `COMPLETADO` o `CANCELADO`. Un turno ya `COMPLETADO` o `CANCELADO` es inmutable. | **Éxito:** Cambia a `COMPLETADO` y suma puntos.<br>**Rechazo:** `409 Conflict` impidiendo acreditar puntos dos veces. |
| **`RN-05`** | Reintegro por cancelación temprana | `turnoId`, `premioCanjeadoId` | Si un turno pasa de `RESERVADO` a `CANCELADO` y posee `premioCanjeadoId != null`, se restituyen los puntos descontados en `ms-premios` y se libera Google Calendar. | **Éxito:** Turno `CANCELADO` y saldo de puntos restaurado en `ms-premios`. |
| **`RN-06`** | Notificaciones por email no bloqueantes | `emailCliente`, datos del evento | Al reservar, cancelar, sumar puntos o canjear un premio se envía correo vía SMTP. Si el servidor SMTP falla, se captura la excepción y se registra en el log (`WARN`). | **Éxito:** Correo enviado (visible en Mailpit `:8025`).<br>**Fallo SMTP:** La transacción principal se confirma normalmente sin arrojar `500`. |

---

## 4. Instrucciones de Compilación, Despliegue y Pruebas (`INF-01`, `API-01`)

### Paso 1: Ejecutar pruebas unitarias y empaquetar con Maven
```bash
mvn clean package
```

### Paso 2: Levantar toda la infraestructura con Docker Compose
```bash
docker compose up --build -d
```

### Paso 3: Ejecutar la Colección de Postman (`API-01`)
En el directorio `postman/` se incluyen los archivos listos para importar en **Postman** o **Bruno**:
1. Importar la colección: `postman/TPI_Peluqueria_2026.postman_collection.json`
2. Importar el entorno: `postman/TPI_Peluqueria_Local.postman_environment.json`
3. Ejecutar en orden las 4 carpetas:
   * **Carpeta 1 (Autenticación):** Ejecuta `1.1 Login Cliente` y `1.2 Login Administrador`. Los scripts guardan automáticamente `{{token_cliente}}` y `{{token_admin}}`.
   * **Carpeta 2 (`SEG-01`):** Verifica rechazos `401 Unauthorized` y `403 Forbidden`.
   * **Carpeta 3 (`NEG-01` / `INT-01`):** Recorre de punta a punta el flujo central (reserva, completitud con acreditación de +100 puntos, reserva con canje de descuento del 10% y cancelación con reintegro automático).
   * **Carpeta 4 (`NEG-02`):** Comprueba los rechazos de reglas de negocio (`400`, `409` y `422`).
4. Abrir `http://localhost:8025` (**Mailpit**) para visualizar todos los correos electrónicos generados durante el flujo (`RN-06`).
