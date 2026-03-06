# 🚗 Vehicle Rental Saga

> **Proyecto de portfolio** — Persistencia distribuida con transacciones compensatorias (Patrón Saga) en Java 21 + Spring Boot 3.5

---

## 📌 Descripción general

Este proyecto demuestra el dominio de la **persistencia de datos distribuida** y el **manejo de fallos** sin transacciones distribuidas (sin JTA/XA).

Simula un escenario real donde un sistema debe escribir en **dos bases de datos relacionales independientes de forma simultánea**, gestionando los fallos parciales mediante el **Patrón Saga con transacciones compensatorias**.

---

## 🧠 El problema que resuelve

En sistemas distribuidos, no se puede envolver dos bases de datos distintas en un único `@Transactional`. Si la escritura #1 tiene éxito y la escritura #2 falla, el sistema queda en un **estado inconsistente**.

El **Patrón Saga** resuelve esto mediante:
- Transacciones locales secuenciales por base de datos
- **Acciones compensatorias** explícitas que deshacen los pasos anteriores ante un fallo
- Sin necesidad de un Transaction Manager externo

---

## ⚖️ ¿Por qué Saga y no 2PC?

El protocolo **Two-Phase Commit (2PC)** garantiza atomicidad distribuida real, pero conlleva un coste elevado que lo hace inviable en muchos sistemas modernos.

| | 2PC (Two-Phase Commit) | Saga (implementado) |
|---|---|---|
| **Coordinación** | Coordinador central bloquea todos los recursos | Ninguno — cada BD actúa de forma independiente |
| **Atomicidad** | Garantizada a nivel distribuido | No existe — consistencia eventual |
| **Rollback** | Automático y sincrónico | Manual y explícito (bloque `catch`) |
| **Implementación** | Requiere JTA/XA + Atomikos/Narayana | Solo Java puro + lógica de negocio |
| **Bloqueos** | Mantiene locks en ambas BBDs durante toda la transacción | Sin locks entre sistemas |
| **Punto de fallo** | El coordinador es un SPOF | Sin coordinador central |

**El trade-off:** el Patrón Saga sacrifica la atomicidad inmediata a cambio de mayor resiliencia y escalabilidad. Ese es el estándar en arquitecturas de microservicios donde los recursos distribuidos no pueden — ni deben — compartir una transacción global.

---

## 🏗️ Arquitectura del flujo Saga

```
POST /api/reservations
         │
         ▼
 ┌───────────────────┐
 │ ReservationSaga   │  ← Orquestador (sin JTA)
 │     Service       │
 └───────┬───────────┘
         │
   ┌─────▼──────┐
   │  PASO 1    │  Guardar → broken_db (MySQL)
   │            │  Sin restricciones → siempre tiene éxito
   └─────┬──────┘
         │
   ┌─────▼─────────────────────────────────────────────┐
   │  PASO 2                                           │
   │  Guardar → fixed_db (PostgreSQL)                  │
   │  Tiene restricción UNIQUE (anti-overbooking)      │
   └─────┬──────────────────────┬─────────────────────-┘
         │                      │
    ✅ OK                ❌ DataIntegrityViolationException
         │                      │
   HTTP 201 Created      ┌──────▼──────────────────┐
                         │  COMPENSACIÓN            │
                         │  DELETE en broken_db     │
                         │  (rollback manual)       │
                         └──────┬───────────────────┘
                                │
                         throw OverbookingException
                                │
                                ▼
                         HTTP 409 Conflict
```

---

## 🗄️ Diseño dual de bases de datos

| | `broken_db` (MySQL 8) | `fixed_db` (PostgreSQL 16) |
|---|---|---|
| **Propósito** | BD legacy / mal diseñada | BD bien diseñada |
| **Claves foráneas** | ❌ Ninguna | ✅ Aplicadas |
| **Restricciones UNIQUE** | ❌ Ninguna | ✅ En email y matrícula |
| **Protección overbooking** | ❌ Ninguna | ✅ `UNIQUE(vehicle_id, start_date, end_date)` |
| **Puerto** | `3306` | `5432` |

---

## 📦 Stack tecnológico

| Capa | Tecnología |
|---|---|
| Lenguaje | Java 21 |
| Framework | Spring Boot 3.5.11 |
| ORM | Spring Data JPA + Hibernate |
| Pool de conexiones | HikariCP |
| BD 1 | MySQL 8 |
| BD 2 | PostgreSQL 16 |
| Contenedores | Docker + Docker Compose |
| Build | Maven |
| Utilidades | Lombok |
| Observabilidad | Spring Boot Actuator |

---

## 🚀 Puesta en marcha

### Requisitos previos

- Docker Desktop en ejecución
- Java 21
- Maven 3.9+

### 1. Clonar el repositorio

```bash
git clone https://github.com/tu-usuario/vehicle-rental-saga.git
cd vehicle-rental-saga
```

### 2. Levantar las bases de datos

```bash
docker-compose up -d
```

Esperar a que ambos contenedores estén sanos:

```bash
docker ps
# Ambos deben mostrar: (healthy)
```

### 3. Arrancar la aplicación

```bash
./mvnw spring-boot:run
```

La API estará disponible en `http://localhost:8080`.

> ⚠️ **El orden importa**: levanta siempre los contenedores Docker antes que Spring Boot. Si las bases de datos no están listas, HikariCP agotará los reintentos de conexión y la aplicación fallará al arrancar.

---

## 🗂️ Estructura del proyecto

```
vehicle-rental-saga/
├── docker-compose.yml
├── sql/
│   ├── mysql-init.sql            # Esquema broken_db (sin restricciones)
│   └── postgres-init.sql         # Esquema fixed_db (FK + UNIQUE)
│
└── src/main/java/com/portfolio/vehiclerentalsaga/
    │
    ├── VehicleRentalSagaApplication.java
    │
    ├── config/
    │   ├── BrokenDbConfig.java       # DataSource MySQL + EntityManagerFactory
    │   └── FixedDbConfig.java        # DataSource PostgreSQL + EntityManagerFactory
    │
    ├── broken/                       # Entidades y repositorios MySQL
    │   ├── entity/
    │   └── repository/
    │
    ├── fixed/                        # Entidades y repositorios PostgreSQL
    │   ├── entity/
    │   └── repository/
    │
    ├── reservation/
    │   ├── dto/
    │   ├── service/
    │   │   └── ReservationSagaService.java   ← Núcleo del patrón
    │   └── controller/
    │       └── ReservationController.java
    │
    └── exception/
        ├── OverbookingException.java
        ├── CompensationFailedException.java
        └── GlobalExceptionHandler.java
```

---

## 🔌 Endpoints de la API

| Método | Endpoint | Descripción |
|---|---|---|
| `POST` | `/api/reservations` | Crear reserva (activa el Saga) |
| `GET` | `/api/reservations/{id}` | Obtener reserva por ID |
| `GET` | `/api/vehicles` | Listar todos los vehículos |
| `POST` | `/api/vehicles` | Registrar un vehículo |
| `POST` | `/api/users` | Registrar un usuario |

### Ejemplo: Crear reserva

```bash
curl -X POST http://localhost:8080/api/reservations \
  -H "Content-Type: application/json" \
  -d '{
    "userId": 1,
    "vehicleId": 1,
    "startDate": "2025-08-01",
    "endDate": "2025-08-05"
  }'
```

**Éxito (201):**
```json
{
  "id": 1,
  "status": "CONFIRMED",
  "message": "Reserva creada correctamente en ambas bases de datos"
}
```

**Overbooking (409):**
```json
{
  "status": 409,
  "error": "Overbooking detectado",
  "message": "El vehículo 1 ya tiene una reserva para las fechas solicitadas. Compensación ejecutada.",
  "compensated": true
}
```

---

## 🔍 Observar el Saga en los logs

El `application.yml` configura `org.springframework.transaction: TRACE` para poder seguir cada paso en la consola:

```
DEBUG - [SAGA PASO 1] Guardando reserva en broken_db (MySQL)...
DEBUG - [SAGA PASO 1] ✅ Guardada con id=42 en broken_db
DEBUG - [SAGA PASO 2] Guardando reserva en fixed_db (PostgreSQL)...
ERROR - [SAGA PASO 2] ❌ DataIntegrityViolationException: overbooking detectado
DEBUG - [COMPENSACIÓN] Eliminando reserva id=42 de broken_db...
DEBUG - [COMPENSACIÓN] ✅ Compensación ejecutada correctamente
ERROR - OverbookingException lanzada → HTTP 409 Conflict
```

---

## ⚙️ Configuración

### `application.yml` — propiedades clave

```yaml
datasource:
  broken:
    jdbc-url: jdbc:mysql://localhost:3306/broken_db
    username: saga_user
    password: saga_pass
  fixed:
    jdbc-url: jdbc:postgresql://localhost:5432/fixed_db
    username: saga_user
    password: saga_pass
```

---

## 🧪 Decisiones de diseño

**¿Por qué dos beans `EntityManagerFactory` separados?**
Spring Data JPA no puede autoconfigurar dos datasources. Cada fuente de datos necesita su propio `EntityManagerFactory`, `TransactionManager` y escaneo de paquetes. Esto se configura manualmente en `BrokenDbConfig` y `FixedDbConfig`.

**¿Por qué la compensación es manual?**
Porque no existe ninguna transacción distribuida que envuelva ambas operaciones. El bloque `catch` en `ReservationSagaService` es la acción compensatoria explícita — esa es la esencia del patrón Saga.

**¿Por qué MySQL como `broken_db`?**
Para reflejar una situación real: sistemas legados con esquemas mal diseñados que no pueden modificarse. El Saga debe funcionar incluso cuando uno de los extremos no ofrece garantías de integridad.

---

## 📊 Health Check

```bash
curl http://localhost:8080/actuator/health
```

```json
{
  "status": "UP",
  "components": {
    "db": { "status": "UP" },
    "diskSpace": { "status": "UP" }
  }
}
```

---

## 👤 Autor

**Eduardo Ballesteros Pérez** —
