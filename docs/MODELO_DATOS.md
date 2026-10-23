# Modelo de Datos

## Entidades Globales (Schema `public`)

Estas entidades viven en el schema compartido y son accesibles por todos los tenants.

### Tenants

Representa cada organización (salón de belleza) en el sistema.

| Campo | Tipo | Restricciones | Propósito |
|---|---|---|---|
| id | UUID | PRIMARY KEY | Identificador único del tenant |
| name | text | NOT NULL | Nombre del salón |
| slug | text | NOT NULL, UNIQUE | URL-friendly identifier del salón |
| plan | text | NOT NULL, DEFAULT 'free' | Tipo de plan (free, premium, enterprise) |
| status | text | NOT NULL, DEFAULT 'active' | Estado (active, suspended, deleted) |
| created_at | timestamp | NOT NULL, DEFAULT NOW() | Fecha de creación del tenant |
| updated_at | timestamp | NOT NULL, DEFAULT NOW() | Fecha de última actualización |

**Índices:** UNIQUE(slug), INDEX(status)

---

### Users

Usuarios que acceden al sistema. Cada usuario pertenece a un único tenant.

| Campo | Tipo | Restricciones | Propósito |
|---|---|---|---|
| id | UUID | PRIMARY KEY | Identificador único del usuario |
| tenant_id | UUID | NOT NULL, FOREIGN KEY → Tenants(id) | Tenant al que pertenece |
| email | text | NOT NULL, UNIQUE | Email del usuario (único globalmente) |
| password_hash | text | NOT NULL | Hash argon2 de la contraseña |
| role | text | NOT NULL, DEFAULT 'staff', CHECK IN ('admin', 'staff', 'barbero') | Rol del usuario en el tenant |
| created_at | timestamp | NOT NULL, DEFAULT NOW() | Fecha de creación |
| updated_at | timestamp | NOT NULL, DEFAULT NOW() | Fecha de última actualización |

**Índices:** UNIQUE(email), INDEX(tenant_id), INDEX(tenant_id, role)

**Constraints:** Un usuario solo puede tener un email. El email es único globalmente para evitar duplicados.

---

### IdempotencyKeys

Almacena claves de idempotencia para operaciones de escritura. Previene duplicación si una petición se reintenta.

| Campo | Tipo | Restricciones | Propósito |
|---|---|---|---|
| key | text | PRIMARY KEY | Identificador único de la clave (UUID o similar) |
| tenant_id | UUID | NOT NULL | Tenant asociado (evita collisiones) |
| endpoint | text | NOT NULL | Ruta del endpoint (ej: `/api/citas`) |
| request_hash | text | NOT NULL | Hash del request (método + payload) |
| response_status | integer | NOT NULL | Código HTTP de la respuesta (201, 400, etc.) |
| response_body | jsonb | Respuesta completa serializada en JSON | |
| created_at | timestamp | NOT NULL, DEFAULT NOW() | Fecha de creación |

**Índices:** INDEX(tenant_id, key), INDEX(created_at)

**TTL:** Los registros con antiguedad > 24h se eliminan automáticamente (opcional: trigger o job externo).

**Propósito:** Si un cliente envía la misma petición POST dos veces con la misma `key`, se devuelve el `response_body` original sin duplicar la operación.

---

## Entidades del Tenant (Replicadas en cada Schema Privado)

Estas entidades existen en un schema privado por tenant. Cada tenant tiene su propia copia de estas tablas, aisladas de otros tenants.

### Clientes

Clientes del salón. Información de los usuarios que compran servicios.

| Campo | Tipo | Restricciones | Propósito |
|---|---|---|---|
| id | UUID | PRIMARY KEY | Identificador único del cliente |
| nombre | text | NOT NULL | Nombre del cliente |
| telefono | text | NOT NULL, UNIQUE PER TENANT | Teléfono (único dentro del tenant) |
| email | text | | Email del cliente (opcional) |
| notas | text | | Notas internas del salón sobre el cliente |
| created_at | timestamp | NOT NULL, DEFAULT NOW() | Fecha de creación |
| updated_at | timestamp | NOT NULL, DEFAULT NOW() | Fecha de última actualización |

**Índices:** UNIQUE(telefono), INDEX(nombre)

**Constraint de unicidad:** El `telefono` es único solo dentro del schema del tenant. Dos tenants pueden tener clientes con el mismo número.

---

### Servicios

Servicios que ofrece el salón (cortes, tinturas, etc.).

| Campo | Tipo | Restricciones | Propósito |
|---|---|---|---|
| id | UUID | PRIMARY KEY | Identificador único del servicio |
| nombre | text | NOT NULL | Nombre del servicio (ej: "Corte clásico") |
| precio | decimal(10, 2) | NOT NULL | Precio en moneda local |
| duracion_minutos | integer | NOT NULL | Duración estimada en minutos |
| activo | boolean | NOT NULL, DEFAULT true | Si el servicio está disponible |
| created_at | timestamp | NOT NULL, DEFAULT NOW() | Fecha de creación |
| updated_at | timestamp | NOT NULL, DEFAULT NOW() | Fecha de última actualización |

**Índices:** INDEX(activo), INDEX(nombre)

---

### Citas

Citas de clientes en el salón.

| Campo | Tipo | Restricciones | Propósito |
|---|---|---|---|
| id | UUID | PRIMARY KEY | Identificador único de la cita |
| cliente_id | UUID | NOT NULL, FOREIGN KEY → Clientes(id) | Cliente que tiene la cita |
| barbero_id | UUID | NOT NULL, FOREIGN KEY → Users(id) | Barbero/empleado asignado |
| fecha_hora_inicio | timestamp | NOT NULL | Inicio de la cita |
| fecha_hora_fin | timestamp | NOT NULL | Fin de la cita (calculado o ingresado) |
| estado | text | NOT NULL, DEFAULT 'pendiente', CHECK IN ('pendiente', 'completada', 'cancelada') | Estado de la cita |
| created_at | timestamp | NOT NULL, DEFAULT NOW() | Fecha de creación |
| updated_at | timestamp | NOT NULL, DEFAULT NOW() | Fecha de última actualización |

**Índices:** INDEX(cliente_id), INDEX(barbero_id), INDEX(fecha_hora_inicio), INDEX(estado)

**Constraint:** `fecha_hora_fin > fecha_hora_inicio`

**Propósito:** Rastrear citas de clientes. Puede tener uno o varios servicios asociados en `CitasDetalle`.

---

### CitasDetalle

Detalles de servicios incluidos en una cita. Una cita puede incluir múltiples servicios.

| Campo | Tipo | Restricciones | Propósito |
|---|---|---|---|
| id | UUID | PRIMARY KEY | Identificador único del detalle |
| cita_id | UUID | NOT NULL, FOREIGN KEY → Citas(id) ON DELETE CASCADE | Cita a la que pertenece |
| servicio_id | UUID | NOT NULL, FOREIGN KEY → Servicios(id) | Servicio incluido en la cita |
| precio_historico | decimal(10, 2) | NOT NULL | Precio del servicio al momento de la cita (para historial) |

**Índices:** INDEX(cita_id), INDEX(servicio_id)

**Propósito:** Permite que una cita incluya múltiples servicios. El `precio_historico` preserva el precio en el momento de la cita, para auditoría e historial de precios.

---

## Relaciones Entre Entidades

```
public.Tenants
    ├── 1:N → public.Users (tenant_id)
    └── 1:N → public.IdempotencyKeys (tenant_id)

[Cada schema tenant_XXX contiene:]

Clientes ←─── 1:N ←─── Citas
                            ├── N:1 ──→ Users (barbero_id)
                            └── 1:N ──→ CitasDetalle
                                        ├── N:1 ──→ Servicios
                                        └── precio_historico (snapshot)
```

---

## Particionamiento por Schema

### Schema `public`
Contiene: `Tenants`, `Users`, `IdempotencyKeys`

Accesible por todas las aplicaciones. Datos globales del sistema.

### Schema `tenant_<UUID>`
Contiene: `Clientes`, `Servicios`, `Citas`, `CitasDetalle`

Un schema por cada tenant activo. Completamente aislado de otros tenants.

**Ejemplo:**
- Salón "FitZone" con UUID `abc123` tiene schema `tenant_abc123`
- Salón "Beauty Plus" con UUID `def456` tiene schema `tenant_def456`
- Una query en `tenant_abc123.clientes` nunca puede acceder a `tenant_def456.clientes`

---

## Validaciones y Reglas de Integridad

### A Nivel de Base de Datos

| Regla | Implementación | Propósito |
|---|---|---|
| Email único globalmente | UNIQUE(email) en Users | Evitar duplicados de usuarios |
| Teléfono único por tenant | UNIQUE(telefono) en Clientes (dentro del schema) | Un cliente no se registra dos veces |
| Estado de cita válido | CHECK en Citas | Solo estados permitidos |
| Barbero válido | FOREIGN KEY barbero_id → Users | Un barbero debe existir en el tenant |
| Horario de cita válido | CHECK fecha_hora_fin > fecha_hora_inicio | Cita con duración válida |
| Casada en cascada | ON DELETE CASCADE en CitasDetalle | Si se elimina una cita, sus detalles se eliminan |

### A Nivel de Aplicación (Core)

| Regla | Propósito |
|---|---|
| Cliente activo y verificado | Validar cliente antes de crear cita |
| Servicio activo | Solo servicios con `activo=true` pueden usarse |
| Disponibilidad de barbero | Verificar que no hay solapamiento de citas en el horario |
| Plan del tenant | Limitar cantidad de citas por mes según plan |
| Idempotencia | Si `Idempotency-Key` existe, no crear duplicado |

---

## Migraciones Requeridas

### Schema `public`

1. `001_create_tenants.ts` — Crear tabla Tenants
2. `002_create_users.ts` — Crear tabla Users con FK a Tenants
3. `003_create_idempotency_keys.ts` — Crear tabla IdempotencyKeys

### Schema `tenant` (Plantilla para replicar en cada tenant nuevo)

1. `001_create_clientes.ts` — Crear tabla Clientes
2. `002_create_servicios.ts` — Crear tabla Servicios
3. `003_create_citas.ts` — Crear tabla Citas con FK a Clientes y Users
4. `004_create_citas_detalle.ts` — Crear tabla CitasDetalle con FK a Citas y Servicios

**Ejecución:** Cuando un nuevo tenant se registra, las migraciones del schema `tenant` se ejecutan en un schema nuevo `tenant_<UUID>`.

