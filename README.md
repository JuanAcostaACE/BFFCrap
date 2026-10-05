# SmartCane AI — Backend

Sistema de tecnología asistiva IoT para personas con discapacidad visual.

---

## Stack

| Capa | Tecnología |
|---|---|
| Backend | Java 21, Spring Boot 4.1.1 |
| Base de datos | PostgreSQL (Railway) / H2 (local) |
| IA | Google Cloud Vision API |
| Firmware | ESP32 (C++ / Arduino) |
| Simulación | Wokwi |
| Frontend | HTML + CSS + Vanilla JS |

---

## Patrones de diseño implementados

| Patrón | Clases | Descripción |
|---|---|---|
| **Strategy** | `ModoProcesamientoStrategy`, `PasivoStrategy`, `ActivoStrategy` | Selección dinámica del modo de procesamiento |
| **Factory Method** | `CreadorStrategy`, `CreadorPasivoStrategy`, `CreadorActivoStrategy`, `StrategyFactory` | Creación desacoplada de strategies |
| **Facade / Adapter** | `VisionFacade`, `VisionAdapter`, `VisionException` | Aislamiento de Google Cloud Vision |
| **Observer** | `EventoObserver`, `EventoRegistradoEvent`, `LogEventoObserver` | Notificación sin acoplamiento |
| **State** | `EstadoSistema`, `EstadoInactivo`, `EstadoPasivo`, `EstadoActivo`, `ContextoSistema` | Gestión de estados del sistema |
| **Command** | `Comando<T>`, `ComandoGuardarEvento`, `ComandoConsultarIA`, `EjecutorComando` | Encapsulamiento de operaciones |

---

## Flujo de procesamiento (todos los patrones en acción)

```
ESP32 POST /api/v1/eventos
    │
    ▼
EventoController
    │
    ▼
EventoService
    ├─ [State]   ContextoSistema.procesarEvento()   → transición de estado
    ├─ [Factory] StrategyFactory.obtenerEstrategia() → crea strategy
    └─ [Strategy] estrategia.procesar()
            │
            ├─ PASIVO: [Command] ComandoGuardarEvento → [Observer] EventoRegistradoEvent
            └─ ACTIVO: [Command] ComandoConsultarIA  → [Facade] VisionAdapter
                                                     → [Command] ComandoGuardarEvento
                                                     → [Observer] EventoRegistradoEvent
```

---

## Correr localmente (sin Railway ni Google Vision)

```bash
# 1. Requiere Java 21 en JAVA_HOME
set JAVA_HOME=C:\Program Files\Java\jdk-21.0.11

# 2. Arrancar (usa H2 en memoria automáticamente)
.\mvnw.cmd spring-boot:run

# 3. API disponible en:
http://localhost:8080/api/v1/eventos        # GET lista, POST nuevo evento
http://localhost:8080/api/v1/eventos/{id}   # GET por ID
http://localhost:8080/api/v1/eventos/estado # GET estado actual del sistema (State pattern)
http://localhost:8080/h2-console            # Consola H2 (solo local)
http://localhost:8080                       # Frontend
```

---

## Prueba rápida (sin hardware)

```bash
# Modo PASIVO — Obstáculo detectado
curl -X POST http://localhost:8080/api/v1/eventos \
  -H "Content-Type: application/json" \
  -d '{"modo":"PASIVO","distanciaCm":28.5}'

# Modo ACTIVO — Análisis solicitado (sin imagen → guarda SIN_IMAGEN)
curl -X POST http://localhost:8080/api/v1/eventos \
  -H "Content-Type: application/json" \
  -d '{"modo":"ACTIVO","distanciaCm":28.5}'

# Ver todos los eventos
curl http://localhost:8080/api/v1/eventos

# Ver estado del sistema (State pattern)
curl http://localhost:8080/api/v1/eventos/estado
```

---

## Avance 1 (Wokwi + Ngrok + Railway) — 9 Oct

### Paso 1 — Ngrok
```bash
# Descargar ngrok desde https://ngrok.com/download
ngrok http 8080
# Copia la URL HTTPS: ej. https://abc123.ngrok-free.app
```

### Paso 2 — Wokwi
1. Ir a https://wokwi.com/projects/new/esp32
2. Reemplazar `diagram.json` con el contenido de `hardware/wokwi/diagram.json`
3. Reemplazar `sketch.ino` con el contenido de `hardware/wokwi/sketch.ino`
4. En `sketch.ino` reemplazar `REEMPLAZAR_CON_TU_URL_NGROK` con la URL de Ngrok
5. Ejecutar la simulación
6. Acercar el sensor (slider de distancia) → POST PASIVO automático
7. Presionar el botón verde → POST ACTIVO

### Paso 3 — Railway PostgreSQL
1. Ir a https://railway.app → New Project → Add a Database → PostgreSQL
2. En Variables, Railway expone: `DATABASE_URL`, `PGUSER`, `PGPASSWORD`, `PGHOST`, `PGPORT`
3. Agregar tu servicio Spring Boot → Variables de entorno:
   - `DB_URL` = `jdbc:postgresql://{PGHOST}:{PGPORT}/{PGDATABASE}`
   - `DB_USERNAME` = valor de `PGUSER`
   - `DB_PASSWORD` = valor de `PGPASSWORD`
   - `SPRING_PROFILES_ACTIVE` = `prod`
4. Ejecutar `schema-postgresql.sql` y `data-postgresql.sql` en la consola de Railway

---

## Variables de entorno requeridas (producción)

| Variable | Descripción | Ejemplo |
|---|---|---|
| `DB_URL` | JDBC URL de PostgreSQL | `jdbc:postgresql://host:5432/railway` |
| `DB_USERNAME` | Usuario de la DB | `postgres` |
| `DB_PASSWORD` | Contraseña | `···` |
| `SPRING_PROFILES_ACTIVE` | Perfil Spring | `prod` |
| `GOOGLE_APPLICATION_CREDENTIALS` | Path al JSON de Service Account | `/app/gcp-key.json` |
| `PORT` | Puerto HTTP (Railway lo inyecta solo) | `8080` |

---

## Estructura del proyecto

```
api/
├── src/main/java/com/smartcane/api/
│   ├── controller/     EventoController, GlobalExceptionHandler
│   ├── command/        Comando, ComandoGuardarEvento, ComandoConsultarIA, EjecutorComando
│   ├── facade/         VisionFacade, VisionAdapter, VisionException
│   ├── factory/        CreadorStrategy, CreadorPasivoStrategy, CreadorActivoStrategy, StrategyFactory
│   ├── model/          RegistroEvento, TipoObstaculo
│   ├── observer/       EventoObserver, EventoRegistradoEvent, LogEventoObserver
│   ├── repository/     RegistroEventoRepository, TipoObstaculoRepository
│   ├── service/        EventoService
│   ├── state/          EstadoSistema, EstadoInactivo, EstadoPasivo, EstadoActivo, ContextoSistema
│   └── strategy/       ModoProcesamientoStrategy, PasivoStrategy, ActivoStrategy
├── src/main/resources/
│   ├── static/         index.html, style.css, app.js
│   ├── application.yaml
│   ├── schema-postgresql.sql
│   └── data-postgresql.sql
├── hardware/wokwi/
│   ├── diagram.json    Circuito ESP32 simulado
│   ├── sketch.ino      Firmware ESP32
│   └── wokwi.toml      Librerías
├── railway.toml        Configuración de despliegue
└── pom.xml
```
