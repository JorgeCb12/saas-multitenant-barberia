# Arquitectura del Proyecto

## Principios de Diseño

El proyecto implementa una arquitectura de capas (layered architecture) para separar responsabilidades y facilitar testing, mantenimiento y escalabilidad. Cada capa tiene una responsabilidad específica y depende únicamente de las capas inferiores.

## Capas de la Arquitectura

### 1. Domain (Capa de Dominio)

Contiene las entidades de negocio y sus definiciones. No depende de Next.js, TypeORM ni de ningún framework externo. Es la capa más pura y agnóstica.

**Responsabilidades:**

- Definir entidades (`Cliente`, `Cita`, `Servicio`)
- Decoradores de TypeORM (relaciones, índices, constraints)
- Validaciones primitivas que pertenecen a la entidad

**Independencia:** Una entidad de dominio puede ser trasladada a otra aplicación o framework sin cambios.

### 2. Core (Capa de Aplicación)

Contiene la lógica de negocio que orquesta entidades del dominio. Servicios, casos de uso, reglas de autenticación y limitación de peticiones. También incluye utilidades compartidas (validadores, errores, constantes).

**Responsabilidades:**

- Autenticación (JWT, hash de contraseñas con argon2)
- Rate limiting (ventana deslizante en memoria)
- Servicios de dominio (crear cliente, registrar cita, validar disponibilidad)
- Generación y verificación de claves de idempotencia
- Validadores Zod, errores personalizados y constantes compartidas

**Dependencias:** Depende de `Domain` e `Infrastructure`.

### 3. Infrastructure (Capa de Infraestructura)

Maneja la persistencia, conexión a bases de datos y detalles técnicos de bajo nivel. Aquí viven las migraciones, configuración de TypeORM y DataSources por tenant.

**Responsabilidades:**

- Gestionar `DataSource` de TypeORM (uno por tenant, cacheado en memoria)
- Ejecutar migraciones en schemas dinámicos
- Conectar a PostgreSQL
- Proporcionar repositorios de acceso a datos

**Dependencias:** Depende de `Domain`. No depende de `Core` ni `App`.

### 4. App (Capa de Presentación)

Next.js App Router. Rutas públicas, páginas del dashboard, endpoints de API y middlewares HTTP.

**Responsabilidades:**

- Rendizado de páginas (SSR, CSR, SSG según corresponda)
- Endpoints REST de la API
- Validación de entrada (Zod)
- Manejo de cookies y sesiones HTTP
- Resolución del tenant a partir del JWT

**Dependencias:** Depende de todas las capas anteriores.

---

## Estructura de Carpetas

```
src/
│
├── app/                          # Capa de Presentación (Next.js App Router)
│   ├── (public)/                 # Grupo de rutas públicas
│   │   ├── layout.tsx            # Layout público
│   │   ├── page.tsx              # Landing
│   │   ├── login/
│   │   │   └── page.tsx          # Página de inicio de sesión
│   │   └── register/
│   │       └── page.tsx          # Página de registro
│   │
│   ├── (dashboard)/              # Grupo de rutas privadas
│   │   ├── layout.tsx            # Layout con sidebar
│   │   ├── clientes/
│   │   │   ├── page.tsx          # Listado de clientes
│   │   │   ├── [id]/
│   │   │   │   └── page.tsx      # Detalle y edición de cliente
│   │   │   └── nuevo/
│   │   │       └── page.tsx      # Crear cliente
│   │   │
│   │   ├── citas/
│   │   │   ├── page.tsx          # Listado de citas
│   │   │   ├── [id]/
│   │   │   │   └── page.tsx      # Detalle de cita
│   │   │   └── nueva/
│   │   │       └── page.tsx      # Crear cita
│   │   │
│   │   ├── servicios/
│   │   │   ├── page.tsx          # Gestión de servicios
│   │   │   └── [id]/
│   │   │       └── page.tsx      # Detalle de servicio
│   │   │
│   │   └── page.tsx              # Dashboard principal
│   │
│   ├── api/                      # Capa de Presentación (API REST)
│   │   ├── auth/
│   │   │   ├── register/
│   │   │   │   └── route.ts      # POST /api/auth/register
│   │   │   ├── login/
│   │   │   │   └── route.ts      # POST /api/auth/login
│   │   │   └── logout/
│   │   │       └── route.ts      # POST /api/auth/logout
│   │   │
│   │   ├── clientes/
│   │   │   ├── route.ts          # GET (listar), POST (crear)
│   │   │   └── [id]/
│   │   │       └── route.ts      # GET, PUT, DELETE
│   │   │
│   │   ├── citas/
│   │   │   ├── route.ts          # GET (listar), POST (crear)
│   │   │   └── [id]/
│   │   │       └── route.ts      # GET, PUT, DELETE
│   │   │
│   │   └── servicios/
│   │       ├── route.ts          # GET (listar), POST (crear)
│   │       └── [id]/
│   │           └── route.ts      # GET, PUT, DELETE
│   │
│   ├── proxy.ts                  # Interceptor de peticiones
│   ├── layout.tsx                # Layout raíz
│   └── error.tsx                 # Página de error global
│
├── core/                         # Capa de Aplicación (Lógica de Negocio)
│   ├── auth/
│   │   ├── jwt.service.ts        # Generación y verificación de JWT
│   │   ├── password.service.ts   # Hash y verificación de contraseñas
│   │   └── auth.service.ts       # Orquestación de registro y login
│   │
│   ├── rate-limiter/
│   │   ├── sliding-window.ts     # Implementación de ventana deslizante
│   │   └── rate-limiter.service.ts # Servicio de rate limiting
│   │
│   ├── idempotency/
│   │   ├── idempotency-key.service.ts # Generación y validación de claves
│   │   └── idempotency.decorator.ts   # Decorador para operaciones idempotentes
│   │
│   ├── services/
│   │   ├── cliente.service.ts    # Lógica CRUD de clientes
│   │   ├── cita.service.ts       # Lógica CRUD de citas
│   │   └── servicio.service.ts   # Lógica CRUD de servicios
│   │
│   └── lib/
│       ├── validators.ts         # Esquemas Zod reutilizables
│       ├── errors.ts             # Clases de error personalizadas
│       └── constants.ts          # Constantes (TTL, límites, etc.)
│
├── domain/                       # Capa de Dominio (Entidades)
│   └── entities/
│       ├── tenant.entity.ts      # Entidad Tenant (en public schema)
│       ├── user.entity.ts        # Entidad User (en public schema)
│       ├── cliente.entity.ts     # Entidad Cliente (en tenant schema)
│       ├── cita.entity.ts        # Entidad Cita (en tenant schema)
│       └── servicio.entity.ts    # Entidad Servicio (en tenant schema)
│
├── infrastructure/               # Capa de Infraestructura
│   └── database/
│       ├── data-source.ts        # Factory de DataSources por tenant
│       ├── data-source-cache.ts  # Caché en memoria (Map) de DataSources
│       └── migrations/
│           ├── public/           # Migraciones del schema public
│           │   ├── 1_create_tenants.ts
│           │   └── 2_create_users.ts
│           └── tenant/           # Migraciones del schema por tenant
│               ├── 1_create_clientes.ts
│               ├── 2_create_servicios.ts
│               ├── 3_create_citas.ts
│               └── 4_create_idempotency_keys.ts
│
└── env.ts                        # Validación de variables de entorno
```

---

## Flujo de una Petición

### Caso: Crear una cita (POST /api/citas)

1. **Presentación (App):**
   - `app/api/citas/route.ts` recibe la petición POST
   - Valida el payload con Zod

2. **Proxy.ts (Next.js):**
   - Intercepta la petición
   - Valida que exista una cookie httpOnly con JWT válido
   - Si no hay token o es inválido, redirige a login
   - La extracción del `tenant_id` se realiza directamente en el `route.ts` de cada endpoint API

3. **Aplicación (Core):**
   - `core/services/cita.service.ts` recibe la solicitud
   - Valida que la `Idempotency-Key` no exista o es nueva
   - Valida reglas de negocio (cliente existe, servicio existe, horario disponible)
   - Valida rate limiting (máximo X citas/minuto)

4. **Infraestructura (Infrastructure):**
   - `infrastructure/database/data-source-cache.ts` obtiene el DataSource cacheado del tenant actual
   - Si no existe, crea uno nuevo con TypeORM
   - Ejecuta la operación en el schema del tenant (no puede tocar otros schemas)

5. **Persistencia:**
   - PostgreSQL recibe la query: `INSERT INTO tenant_abc123.citas ...`
   - La cita se guarda solo en el schema del tenant

6. **Respuesta:**
   - `core/services/cita.service.ts` guarda la `Idempotency-Key` con TTL 24h
   - `app/api/citas/route.ts` devuelve 201 Created con los datos de la cita

---

## Aislamiento de Tenants

El aislamiento ocurre a nivel de código, no de red:

1. **DataSource por tenant:** Cada tenant tiene su propio objeto `DataSource` cacheado. Una query de un DataSource nunca conoce otros schemas.

2. **Resolución en proxy.ts:** Antes de cualquier endpoint, se valida que el JWT sea válido y se extrae el `tenant_id`.

3. **Validación en servicios:** Los servicios de Core nunca aceptan un `tenant_id` falso o ausente.

4. **Tabla idempotency_keys en public:** Las claves de idempotencia se guardan en un schema compartido, pero cada operación está asociada a un tenant_id, evitando collisiones.

---

## Decisiones de Diseño

### ¿Por qué DataSource por tenant y no un solo DataSource?

Un solo DataSource con `schema` como parámetro es más simple de configurar pero más frágil: un error en la lógica de resolución del tenant puede exponer datos. Con DataSource por tenant, **la arquitectura impide accesos cruzados por construcción**.

### ¿Por qué Core depende de Infrastructure?

Porque los servicios de negocio necesitan persistir datos. La inyección de dependencias permite testear los servicios sin tocar la BD (mockeando el DataSource).

### ¿Por qué App depende de todas las capas?

Porque Next.js es el punto de entrada. Las rutas HTTP y páginas coordinan dominio, lógica y persistencia.

---

## Testing por Capa

| Capa               | Qué testear                                    | Cómo                              |
| ------------------ | ---------------------------------------------- | --------------------------------- |
| **Domain**         | Validaciones de entidad                        | Unitarias, sin BD                 |
| **Core**           | Lógica de negocio, rate limiting, idempotencia | Unitarias, mockeando DataSource   |
| **Infrastructure** | Conexión, migraciones, repositorios            | Integración, con BD de test       |
| **App**            | Endpoints, validación de entrada, códigos HTTP | Integración, con servidor de test |
