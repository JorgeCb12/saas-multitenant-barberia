# Stack Tecnológico

## Runtime y aplicación

| Tecnología | Versión | Rol                  | Justificación técnica                                                                                                        |
| ---------- | ------- | -------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| Node.js    | 22 LTS  | Runtime              | LTS con soporte extendido y compatibilidad con TypeORM y Next.js 16. Node 20 está EOL.                                       |
| Next.js    | 16.3.8  | Fullstack (API + UI) | Opción B del enunciado. `proxy.ts` reemplaza `middleware.ts` deprecado. `output: standalone` reduce tamaño de imagen Docker. |
| TypeScript | 5.x     | Tipado estricto      | Exigido para que Zod y TypeORM aporten seguridad de tipos en tiempo de desarrollo y ejecución.                               |
| pnpm       | 9.x     | Gestor de paquetes   | Lockfile determinista en CI e instalación más rápida que npm.                                                                |

## Persistencia

| Tecnología                    | Versión   | Rol                    | Justificación técnica                                                                                                                                                                                                                                               |
| ----------------------------- | --------- | ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| PostgreSQL                    | 16-alpine | Base de datos          | Requisito 4.1 del enunciado. Soporta `CREATE SCHEMA` transaccional, necesario para provisionar schemas de nuevos tenants dinámicamente.                                                                                                                             |
| TypeORM                       | 1.1.1     | ORM                    | Requisito 4.3. Soporta múltiples `DataSource` y migraciones parametrizables por schema. Maneja bien schema-per-tenant dinámico.                                                                                                                                     |
| pg                            | 8.x       | Driver PostgreSQL      | Dependencia de TypeORM para conectar a PostgreSQL.                                                                                                                                                                                                                  |
| DataSource por tenant (Vía A) | —         | Estrategia multitenant | Cada tenant (salón) tiene su propio `DataSource` cacheado en memoria (`Map`). Aislamiento a nivel de código: una query de un salón no puede cruzar a otro schema. Demostrable en sustentación: servicios, citas y clientes de un salón jamás son visibles por otro. |

## Autenticación

| Tecnología      | Versión | Rol                         | Justificación técnica                                                                                                                                                           |
| --------------- | ------- | --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| argon2          | 0.41.x  | Hash de contraseñas         | Recomendado por OWASP. Memory-hard, resistente a ataques con GPU. Superior a bcrypt en seguridad.                                                                               |
| jose            | 5.x     | Firma y verificación de JWT | Funciona en Node runtime. Necesario porque `proxy.ts` corre en Node en Next.js 16.                                                                                              |
| Cookie httpOnly | —       | Sesión                      | Requisito 4.2: JWT almacenado en cookie httpOnly, no en localStorage. `SameSite=Lax`, `Secure` en producción, `Path=/`.                                                         |
| Auth propia     | —       | Registro/login              | Requisito 4.6 exige que `docker compose up` levante todo. Auth propia permite inyectar `tenant_id` y `role` en el JWT sin dependencias externas ni sincronización con Supabase. |

## Validación y robustez

| Tecnología                                | Versión | Rol                                 | Justificación técnica                                                                                                                                                                                   |
| ----------------------------------------- | ------- | ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Zod                                       | 3.x     | Validación en runtime               | Requisito 4.3. Tipos inferidos con `z.infer`, sin duplicar interfaces. Valida datos antes de persistir en BD.                                                                                           |
| Rate limiting en memoria (sliding window) | —       | Limitación de peticiones            | Requisito 4.5. Sin dependencias externas. Trade-off: no escala horizontal; Redis sería el paso siguiente. Documentado en decisión arquitectónica.                                                       |
| Tabla `public.idempotency_keys`           | —       | Persistencia de claves idempotentes | Requisito 4.4. Almacena claves con TTL de 24 horas. Sobrevive a reinicios del contenedor. Casos de uso: crear cita, crear servicio o crear cliente con la misma `Idempotency-Key` no duplica registros. |

## Infraestructura

| Tecnología     | Versión | Rol                               | Justificación técnica                                                                                                                         |
| -------------- | ------- | --------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| Docker         | 27.x    | Contenedores                      | Requisito 4.6. Empaquetamiento de aplicación y BD.                                                                                            |
| Docker Compose | v2      | Orquestación local                | `docker compose up --build` levanta `db` + `app`. Healthcheck en `db` y `depends_on: condition: service_healthy` para evitar race conditions. |
| `.env.example` | —       | Variables de entorno documentadas | Requisito 4.6 y regla 10: sin secretos reales en el repositorio.                                                                              |

## Calidad y CI/CD

| Tecnología           | Versión       | Rol                 | Justificación técnica                                                              |
| -------------------- | ------------- | ------------------- | ---------------------------------------------------------------------------------- |
| ESLint               | Built-in Next | Lint                | Requisito 4.7: el pipeline falla si el lint falla.                                 |
| Prettier             | 3.x           | Formato de código   | Consistencia en commits y diffs. No bloquea el pipeline.                           |
| Vitest               | 2.x           | Pruebas unitarias   | Pruebas de aislamiento entre tenants e idempotencia. Ejecución rápida.             |
| GitHub Actions       | —             | CI/CD               | Requisito 4.7: install, lint, typecheck, build, test. Falla si algún paso no pasa. |
| Conventional Commits | —             | Historial coherente | Entregable del enunciado. Commits `feat`, `fix`, `chore`, etc.                     |

## Tecnologías descartadas y por qué

| Tecnología                | Motivo del descarte                                                                                                                                     |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Supabase (Cloud)          | Rompe el requisito 4.6 de "un solo comando". La BD ya está levantada en la nube, `docker compose up` no levanta todo.                                   |
| Supabase Auth (híbrido)   | PostgreSQL local + Auth remota introduce sincronización de usuarios entre dos sistemas, duplicación del `tenant_id` y debilita el requisito Docker.     |
| Prisma                    | No soporta bien schema-per-tenant dinámico. TypeORM con DataSource por tenant es más directo para crear schemas en tiempo de ejecución.                 |
| Redis                     | Innecesario para una instancia única. Rate limiting en memoria es suficiente. Se documenta como evolución futura para escalado horizontal.              |
| Auth.js / NextAuth.js     | Añade una capa que oculta el JWT. Para sustentación conviene control explícito sobre el token con `jose`.                                               |
| tenancyjs-adapter-typeorm | Limita la API (sin relations ni query builder avanzado). Restrictivo para CRUD completo de entidades relacionales.                                      |
| Middleware de Next.js     | `middleware.ts` está deprecado en Next.js 16 y renombrado a `proxy.ts`. `proxy.ts` es la vía vigente para interceptar peticiones en el runtime de Node. |
