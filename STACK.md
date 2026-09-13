# Stack Tecnológico — Apps Web y Móviles

> Documento de referencia para todos los proyectos nuevos de apps web y móviles.
> **Autor:** Michael Santiago · **Última actualización:** 2026-09-12 · **Versión:** 1.1

Este documento define el stack por defecto. Desviarse de él es válido, pero debe justificarse
por escrito en el README del proyecto que se desvía.

Muchas secciones están escritas en **dos fases**: la opción más barata y simple para arrancar, y
la opción recomendable al escalar, con un **disparador explícito** de cuándo migrar. La regla es
no adelantar complejidad: se migra cuando se cumple el disparador, no antes.

---

## 1. Resumen ejecutivo

| Capa | Elección |
|---|---|
| Lenguaje | TypeScript (modo estricto) + Zod |
| Web | Next.js (App Router) |
| Móvil | React Native + Expo |
| UI | Tailwind CSS + shadcn/ui (web) · NativeWind (móvil) |
| Estado / datos | TanStack Query + Zustand |
| Backend | Next.js Route Handlers → Hono (servicio separado) → NestJS (dominio complejo) |
| Contrato de API | tRPC dentro del monorepo · OpenAPI para consumidores externos |
| Base de datos — cloud | Supabase (Postgres gestionado) |
| Base de datos — autohospedada | Postgres + Drizzle ORM |
| Auth | Supabase Auth |
| Caché | Caché de Next.js → Redis (Upstash) |
| Colas y cron | Vercel Cron + pg_cron → QStash / Inngest → BullMQ sobre Redis |
| Archivos | Cloudflare R2 (principal) · Supabase Storage (alternativa) |
| Monorepo | Turborepo + pnpm |
| Testing | Vitest (unit) + Playwright (E2E) |
| Lint / Format | Biome |
| Hosting web | Vercel (por defecto) · Docker + VPS (alternativa por costo) |
| Pagos | Stripe |
| Email | Resend |
| Observabilidad | Sentry (errores) + PostHog (producto) + Pino (logs) |
| Diseño / IA | v0 + Figma (Dev Mode MCP) + Claude Code con MCP |

---

## 2. Principios

1. **Un solo lenguaje de punta a punta.** TypeScript en web, móvil y backend. Los tipos se
   comparten, no se reescriben.
2. **Postgres siempre.** Sea Supabase o autohospedado, la base es Postgres. Migrar entre ambos
   escenarios es un cambio de conexión, no un rewrite.
3. **Empezar barato, escalar con criterio.** Cada migración tiene un disparador definido; no se
   adelanta infraestructura "por si acaso".
4. **Preferir componentes propios sobre librerías cerradas.** shadcn/ui se copia al repo — el
   código es nuestro y se modifica sin pelear con la librería.
5. **Validación en runtime, no solo en tipos.** TypeScript no protege en producción; Zod sí.
6. **Todo debe poder correr en dos lados.** Ninguna decisión debe atarnos a un proveedor en la
   lógica de negocio.

---

## 3. Lenguaje y validación

- **TypeScript** con `strict: true`. Sin `any` implícito.
- **Zod** para validar todo lo que cruza una frontera: formularios, respuestas de API, variables
  de entorno, payloads de webhooks.
- Los esquemas de Zod viven en `packages/shared` y se usan en web, móvil y backend. Un solo
  esquema es a la vez la validación y la fuente del tipo (`z.infer`).

---

## 4. Web — Next.js (App Router)

- **Next.js** con App Router y React Server Components.
- Route Handlers (`app/api/*`) para la API cuando el backend vive en el mismo proyecto.
- Server Actions para mutaciones desde formularios del propio dashboard.
- SSR/ISR para páginas públicas e indexables; client components solo donde hay interactividad real.

**Cuándo NO usar Next.js:** una landing puramente estática y sin sesión puede ir en Astro. Es la
excepción, no la regla.

---

## 5. Móvil — React Native + Expo

- **Expo** (managed workflow) con **Expo Router**, navegación basada en archivos igual que Next.js.
- **EAS Build** para compilar binarios iOS/Android.
- **EAS Update** para actualizaciones OTA de JS sin pasar por las tiendas.
- **expo-notifications** para push. Se configura desde el inicio: agregarlo después obliga a
  rehacer el onboarding y a pedir permisos en mal momento.
- **Deep links / universal links** definidos desde el día uno — cambiarlos después rompe enlaces
  ya publicados en emails y campañas.
- Código nativo custom solo vía config plugins; se evita salir a bare workflow salvo necesidad real.

---

## 6. UI y estilos

| Plataforma | Solución |
|---|---|
| Web | Tailwind CSS + shadcn/ui (sobre Radix UI) |
| Móvil | Tailwind vía NativeWind |

- Los **design tokens** (colores, espaciado, tipografía, radios) se definen una sola vez en
  `packages/ui/tokens` y alimentan ambas configuraciones de Tailwind.
- Los componentes de shadcn se copian al repo y se modifican libremente.
- Iconos: **Lucide** (`lucide-react` en web, `lucide-react-native` en móvil).
- No se comparten componentes visuales entre web y móvil — se comparten *tokens* y *lógica*.
  Intentar un componente universal casi siempre cuesta más de lo que ahorra.

---

## 7. Estado y datos en el cliente

- **TanStack Query** — todo lo que viene del servidor: caché, reintentos, invalidación, estados
  de carga y error. Funciona igual en web y en Expo.
- **Zustand** — estado global de UI únicamente (tema, filtros, wizard abierto, sesión en memoria).
  Nunca datos que ya son del servidor.
- **useState / Context** — estado local de un componente o subárbol.

**Regla:** si el dato tiene dueño en la base de datos, es de TanStack Query. Si solo existe
mientras la pantalla está abierta, es de Zustand o useState.

En web, cuando la página es Server Component, los datos se cargan en el servidor y TanStack Query
se usa solo en las partes interactivas.

---

## 8. Backend — stack, contrato de API y hosting

### 8.1 Dónde vive el backend — tres etapas

**Etapa 1 — Dentro de Next.js (por defecto).**
Route Handlers + Server Actions. Un solo repo, un solo deploy, un solo dominio, cero CORS.
Cubre perfectamente el 80% de los productos: CRUD, auth, webhooks, integraciones.

**Etapa 2 — Servicio separado con Hono.**
Se extrae el backend cuando aparece **cualquiera** de estas señales:

- La app móvil es el cliente principal y la web es secundaria (no tiene sentido que la API viva
  dentro del frontend web).
- Hay procesos que superan el límite de tiempo de las funciones serverless.
- Se necesita conexión persistente (WebSocket propio, MQTT, workers de cola).
- Un tercero va a consumir la API.

**Hono** es la elección: TypeScript, ultraligero, corre igual en Node, Bun, Docker o edge, y la
sintaxis es prácticamente Express con tipado real. Migrar Route Handlers a Hono es casi copiar y
pegar.

**Etapa 3 — NestJS, solo si el dominio lo justifica.**
Cuando hay muchos módulos de negocio, varios desarrolladores backend y se necesita estructura
impuesta: inyección de dependencias, módulos, guards, interceptores. Es más ceremonia; solo vale
la pena cuando el tamaño del equipo la paga.

### 8.2 Contrato de API

| Caso | Solución |
|---|---|
| Web y móvil dentro del monorepo | **tRPC** — tipado de punta a punta sin generar código ni escribir esquemas dos veces. Si cambias el backend, el frontend deja de compilar al instante. |
| Consumidores externos, clientes, IoT | **REST + OpenAPI**, generado desde Zod con `zod-openapi` o `hono/zod-openapi`. Documentación automática. |
| Ambos | tRPC para lo interno, una capa REST pública delgada encima del mismo servicio. |

GraphQL queda descartado: resuelve un problema de múltiples equipos frontend consumiendo un
backend compartido, que no es nuestro caso, y cuesta mucho en caché y control de queries.

### 8.3 Hosting del backend

| Fase | Dónde | Por qué |
|---|---|---|
| Inicio | **Vercel** (junto al frontend) | Cero configuración. Límite: funciones con tiempo máximo de ejecución y sin procesos persistentes. |
| Intermedio | **Railway** o **Fly.io** | Contenedor siempre encendido, barato, deploy por git push. Aquí ya puedes correr workers de cola y WebSockets. |
| Escala / control | **Docker en VPS** o **AWS ECS** | Control total de costos y recursos. Necesario para cargas pesadas, GPU o requisitos de compliance. |

**Nota crítica para SLM:** el procesamiento pesado y las cargas de GPU **nunca** van en funciones
serverless — los límites de tiempo y memoria no dan. Van en contenedores dedicados (ECS, EC2,
RunPod o infraestructura propia), invocados por cola desde la API.

---

## 9. Base de datos

Dos escenarios soportados. **Ambos son Postgres**, así que el esquema y la lógica SQL son portables.

### 9.1 Escenario A — Cloud (por defecto)

**Supabase**: Postgres gestionado + Auth + Storage + Realtime, con Row Level Security.

- Se usa para la mayoría de proyectos: MVPs, productos nuevos, clientes sin requisitos de
  residencia de datos.
- La seguridad vive en **RLS a nivel de base**, no solo en el código de la app.
- Migraciones versionadas con Supabase CLI, dentro del repo.

### 9.2 Escenario B — Autohospedado

**Postgres + Drizzle ORM**, cuando la base debe vivir en otro lado.

- Aplica si: el cliente exige datos en su propia infraestructura, hay requisitos de compliance o
  residencia de datos, o el volumen hace que Supabase salga más caro que un VPS.
- Drizzle da tipado completo desde el esquema y genera SQL predecible, sin sorpresas de rendimiento.
- Migraciones con `drizzle-kit`, versionadas en el repo.
- Postgres en Docker, o servicio gestionado (Neon / RDS) si se prefiere no operarlo.

### 9.3 Connection pooling — obligatorio con serverless

Cada invocación serverless abre su propia conexión a Postgres. Sin pooler, un pico de tráfico
agota el límite de conexiones y la base empieza a rechazar todo.

- Con Supabase: usar la cadena de conexión de **Supavisor** en modo transacción, no la directa.
- Autohospedado: **PgBouncer** delante de Postgres.

Esto no es optimización, es configuración base. Se hace desde el primer deploy.

### 9.4 Al escalar

- **Réplicas de lectura** para reportes y dashboards, separadas de la carga transaccional.
- Índices revisados con `EXPLAIN ANALYZE` sobre queries reales, no supuestas.
- Particionado o TimescaleDB para tablas de series de tiempo (ver §13).

---

## 10. Autenticación — Supabase Auth

- Email + contraseña, magic links y OAuth (Google, Apple — **Apple es obligatorio** si hay login
  social en iOS).
- Integrado con RLS: la política de la tabla lee `auth.uid()` directamente.
- En Expo, la sesión se persiste en `expo-secure-store`, nunca en AsyncStorage plano.
- Roles y permisos en una tabla `profiles` propia, referenciada desde las políticas RLS.

> En el escenario autohospedado (§9.2), Supabase Auth puede seguir usándose como servicio
> independiente de la base, o reemplazarse por **Better Auth** sobre el mismo Postgres.
> Decisión por proyecto.

**Al escalar (clientes enterprise):** SSO con SAML/OIDC, SCIM para aprovisionamiento de usuarios,
y log de auditoría. Suele ser requisito de venta, no técnico.

---

## 11. Caché, colas y trabajos programados

### 11.1 Caché — dos fases

**Fase 1 — Sin infraestructura adicional ($0):**

- Caché nativa de Next.js: `revalidate` en fetch, `unstable_cache` para funciones costosas,
  ISR para páginas.
- CDN de Vercel para assets y respuestas estáticas.
- TanStack Query en el cliente: evita refetch innecesario en web y móvil.

Esto cubre bastante más de lo que la gente asume. No se agrega Redis antes de tiempo.

**Fase 2 — Redis, cuando ocurre alguna de estas señales:**

- Queries que tardan cientos de milisegundos y se repiten mucho.
- Necesitas rate limiting real (contadores compartidos entre instancias).
- Hay trabajos en segundo plano con reintentos.
- Necesitas invalidar caché desde varios servicios a la vez.

### 11.2 Trabajos programados (cron) — dos fases

**Fase 1 ($0):**

| Herramienta | Para qué |
|---|---|
| **Vercel Cron** | Tareas HTTP: reportes diarios, limpieza, sincronizaciones. Se define en `vercel.json`. Verificar la frecuencia permitida por el plan. |
| **pg_cron** (Supabase) | Tareas que son puro SQL: purgar registros viejos, refrescar vistas materializadas, agregar métricas. Corre dentro de la base, sin red de por medio. |

**Fase 2 — cuando los trabajos crecen:**

| Herramienta | Para qué |
|---|---|
| **QStash** (Upstash) | Cola por HTTP con reintentos y programación. Funciona en serverless, sin servidor que mantener. Escalón natural desde Vercel Cron. |
| **Inngest** / **Trigger.dev** | Flujos de varios pasos, durables, con reintentos por paso y visibilidad de cada ejecución. Para procesos de negocio largos. |
| **BullMQ** sobre Redis | Máximo control y menor costo por trabajo. **Requiere un proceso worker siempre encendido** (Railway, Fly.io o VPS) — no funciona en Vercel serverless. |

**Disparador de migración:** cuando un trabajo fallido sin reintento automático empiece a costar
dinero o soporte, ya necesitas cola real.

### 11.3 Redis — cómo aprovecharlo concretamente

**Proveedor: Upstash.** Es Redis serverless con API por HTTP, lo cual importa mucho: Redis
tradicional usa conexiones TCP persistentes, que en funciones serverless se agotan igual que las
de Postgres. Upstash evita ese problema y cobra por petición, así que arranca prácticamente en $0.
Si ya tienes VPS, Redis en Docker sale más barato a volumen alto.

Usos, ordenados por retorno:

**1. Rate limiting** — el de mayor impacto y el más fácil.
Protege login, registro, recuperación de contraseña, endpoints públicos y la API de dispositivos.
Sin esto, cualquiera puede hacerte fuerza bruta o inflar tu factura de Supabase.

```ts
import { Ratelimit } from '@upstash/ratelimit'
import { Redis } from '@upstash/redis'

const limiter = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, '10 s'),
})

const { success } = await limiter.limit(`login:${ip}`)
if (!success) return new Response('Demasiados intentos', { status: 429 })
```

**2. Caché de queries costosas (cache-aside).**
Dashboards, catálogos, agregaciones. Se lee de Redis; si no está, se consulta Postgres y se guarda
con TTL. Baja la carga de la base y el tiempo de respuesta de golpe.

```ts
const cached = await redis.get<Stats>(`stats:${orgId}`)
if (cached) return cached
const stats = await db.computeStats(orgId)   // query pesada
await redis.set(`stats:${orgId}`, stats, { ex: 300 })  // 5 min
return stats
```

**3. Idempotencia de webhooks.**
Stripe reintenta los webhooks. Si no deduplicas, cobras o envías el mismo email dos veces.
Guardar el `event.id` en Redis con TTL resuelve el problema en tres líneas:

```ts
const nuevo = await redis.set(`stripe:${event.id}`, 1, { nx: true, ex: 86400 })
if (!nuevo) return new Response('ok')  // ya procesado, salir
```

**4. Locks distribuidos.**
Evitan que dos instancias ejecuten el mismo cron a la vez, o que dos peticiones simultáneas creen
el mismo registro. `SET key value NX EX 30`.

**5. Colas de trabajo (BullMQ).**
Emails, generación de PDFs, procesamiento de imágenes, llamadas a APIs externas, reportes. Todo lo
que el usuario no debe esperar. Con reintentos, backoff y dead-letter queue.

**6. Contadores y rankings en vivo.**
`INCR` y sorted sets para métricas en tiempo real sin escribir a Postgres en cada evento. Se
persiste agregado cada N minutos.

**7. Buffer de telemetría IoT.**
Muy relevante para SLM: los dispositivos escriben a Redis a alta frecuencia y un proceso vuelca
lotes a Postgres/TimescaleDB cada X segundos. Evita miles de INSERT individuales que matarían la
base. También sirve para guardar el "último estado conocido" de cada dispositivo, que es la
consulta más frecuente de cualquier dashboard IoT.

**8. Sesiones, tokens de un solo uso y feature flags cacheados.**
Tokens de invitación, códigos OTP, flags leídos en cada request.

---

## 12. Almacenamiento de archivos

| Opción | Cuándo |
|---|---|
| **Cloudflare R2** (principal) | Por defecto. **Sin cargos de egreso**, que es la diferencia grande frente a S3 — con media o descargas frecuentes, la factura no se dispara. API compatible con S3, así que las librerías son las mismas. |
| **Supabase Storage** (alternativa) | Cuando el proyecto ya vive en Supabase y el volumen es bajo: una dependencia menos, y las políticas de acceso usan el mismo RLS y el mismo `auth.uid()` que el resto. Se paga egreso. |
| **AWS S3** | Solo si el cliente ya está en AWS o lo exige por compliance. |

Reglas, sin importar la opción:

- Los buckets son **privados**. El acceso es por URL firmada con expiración, nunca público.
- La subida va **directo del cliente al storage** con URL firmada — el archivo no pasa por tu API,
  que si no se convierte en cuello de botella y en costo de ancho de banda.
- Se valida tipo MIME y tamaño **en el servidor** antes de firmar la URL, no solo en el cliente.
- Imágenes servidas por Cloudflare Images o `next/image`; nunca el original de 8 MB al navegador.

---

## 13. Tiempo real y telemetría — dos fases

### Fase 1 — Inicio (mínimo costo)

**Supabase Realtime** únicamente.

- Los dispositivos y clientes escriben por API REST (Route Handler o PostgREST).
- La UI se suscribe a cambios de Postgres por WebSocket: actualizaciones en vivo sin
  infraestructura adicional.
- Incluido en el plan de Supabase, sin servicio extra que operar.

**Suficiente para:** dashboards en vivo, notificaciones, presencia, chat, y telemetría de baja
frecuencia — decenas o cientos de dispositivos reportando cada varios segundos.

### Fase 2 — Al escalar

Migrar cuando ocurra **cualquiera** de estas señales:

- Miles de dispositivos conectados simultáneamente.
- Escrituras de alta frecuencia (sub-segundo por dispositivo) que inflan la tabla y degradan queries.
- Dispositivos con red intermitente o restricciones de batería/ancho de banda, donde HTTP resulta
  demasiado costoso.
- Se necesitan consultas de series de tiempo: agregaciones por ventana, retención, downsampling.

Entonces:

- **MQTT** (EMQX o Mosquitto) como broker de ingesta: protocolo ligero, QoS configurable,
  conexiones persistentes. Diseñado exactamente para este caso.
- **Redis como buffer** entre MQTT y la base (§11.3, punto 7).
- **TimescaleDB** — extensión de Postgres para series de tiempo: hypertables, compresión,
  agregados continuos y políticas de retención. Sigue siendo Postgres, el resto del stack no cambia.
- Supabase Realtime se mantiene para la capa de UI: MQTT alimenta la base, la base alimenta la UI.

---

## 14. Monorepo — Turborepo + pnpm

```
proyecto/
├─ apps/
│  ├─ web/            # Next.js
│  ├─ mobile/         # Expo
│  └─ api/            # Hono (solo si el backend se extrajo, §8.1)
├─ packages/
│  ├─ shared/         # tipos, esquemas Zod, utilidades puras
│  ├─ db/             # esquema, migraciones, cliente de datos
│  ├─ ui/             # design tokens, config de Tailwind
│  └─ config/         # tsconfig, biome, presets compartidos
├─ turbo.json
└─ pnpm-workspace.yaml
```

- **pnpm**: enlaces simbólicos, mucho menos disco y más velocidad que npm.
- **Turborepo**: caché de builds y tareas en paralelo — solo se reconstruye lo que cambió.
- Expo requiere ajustes en `metro.config.js` para resolver los paquetes del workspace; es un paso
  conocido y documentado, no un bloqueante.

### Alternativas evaluadas

| Opción | Veredicto |
|---|---|
| **Nx** | Más potente: generadores, gráfico de dependencias, caché distribuida. Justificado con 5+ apps o varios equipos. Para nuestra escala, la curva no se paga. |
| **pnpm workspaces sin Turborepo** | Válido al inicio, pero sin caché el CI reconstruye todo en cada push. Se nota apenas hay 2 apps y 3 paquetes. |
| **Bun workspaces** | Muy rápido, pero Metro (bundler de Expo) todavía tiene fricciones. Revisar en 6 meses. |
| **Yarn Berry (PnP)** | PnP rompe React Native/Metro. Sin PnP no aporta nada sobre pnpm. |
| **Repos separados** | Obliga a publicar los tipos como paquete npm privado; cada cambio de contrato son dos PRs y un release. Solo si son productos verdaderamente independientes. |

---

## 15. Diseño, prototipado y desarrollo asistido por IA

Esta capa es parte del stack, no un extra. Define cómo se pasa de idea a código.

### 15.1 Flujo de diseño

| Etapa | Herramienta |
|---|---|
| Exploración / wireframe rápido | **Claude Code** con el skill `design` — genera un canvas de artboards editable visualmente, sin salir de la terminal. Para decidir estructura antes de abrir Figma. |
| Diseño de alta fidelidad | **Figma** — sigue siendo la fuente de verdad del diseño visual y de los tokens. |
| Diseño → código | **v0** (Vercel) — genera React + Tailwind + shadcn/ui, que es exactamente nuestro stack. Lo que sale es código utilizable, no un punto de partida a reescribir. |
| Catálogo de componentes | **Storybook** — se agrega cuando hay más de un desarrollador tocando la UI, o cuando el cliente quiere revisar componentes aislados. |

**Regla:** los design tokens se definen en Figma y se sincronizan a `packages/ui/tokens`. Nunca se
copian colores a mano desde una captura de pantalla.

### 15.2 MCP para Claude Code

Los servidores MCP le dan a Claude Code acceso directo a las herramientas del stack, en lugar de
que le peguemos capturas o texto copiado.

| Servidor MCP | Qué habilita |
|---|---|
| **Figma (Dev Mode MCP)** | Claude lee el diseño real — medidas, tokens, jerarquía — y genera el componente. Elimina el "ojímetro". Requiere asiento de pago en Figma. |
| **Supabase MCP** | Consultar el esquema, escribir migraciones y probar queries y políticas RLS sin salir del editor. |
| **Sentry MCP** | Traer un error de producción con su stack trace y contexto directo a la sesión. |
| **Stripe MCP** | Consultar objetos, productos y eventos en modo test al construir el flujo de pago. |
| **Playwright MCP** | Claude abre el navegador, navega la app y verifica que el cambio funciona de verdad. |
| **GitHub MCP** | PRs, issues y revisiones desde la sesión. |
| **Context7** | Documentación actualizada de librerías inyectada en contexto, en vez de versiones viejas de memoria. |

Configuración en `.mcp.json` del repo, versionada, para que todo el equipo tenga los mismos
servidores.

### 15.3 Convenciones de repo para IA

Parte del andamiaje de cada proyecto:

- **`CLAUDE.md`** en la raíz: arquitectura, comandos, convenciones y decisiones del proyecto.
  Se genera con `/init` y se mantiene al día. Es lo que evita que la IA proponga cosas fuera del
  stack.
- **`.claude/skills/`**: procedimientos repetitivos del proyecto (crear un módulo, agregar una
  migración, preparar un release) escritos una vez.
- **`.claude/settings.json`**: hooks y permisos compartidos por el equipo.
- **Reglas duras** en CLAUDE.md: nunca tocar migraciones aplicadas, nunca commitear secretos,
  siempre validar con Zod en los bordes.

---

## 16. Testing y calidad de código

| Herramienta | Uso |
|---|---|
| **Vitest** | Tests unitarios y de integración. Rápido, misma API que Jest. |
| **Testing Library** | Tests de componentes (`@testing-library/react` y `/react-native`). |
| **Playwright** | E2E de los flujos críticos en web. |
| **Biome** | Lint + formato en una sola herramienta (reemplaza ESLint + Prettier). |
| **TypeScript** | `tsc --noEmit` en CI como puerta de calidad. |

**Alcance esperado:** no se persigue cobertura alta por sí misma. Se testea obligatoriamente la
lógica de negocio pura, los cálculos de dinero, las validaciones, los webhooks de Stripe, y los
3–5 flujos E2E que el producto no puede permitirse romper (registro, login, checkout).

**Al escalar:** tests de carga con **k6** antes de campañas o lanzamientos grandes, y **Maestro**
para E2E en móvil.

---

## 17. CI/CD — dos fases

### Fase 1 — Inicio (mínimo costo)

- **Vercel** desplegando automáticamente con cada push: preview por rama, producción en `main`.
- **EAS Build manual** — se compila cuando toca release. El tier gratuito alcanza al ritmo de
  releases de un producto nuevo.
- **EAS Update** para cambios de JS entre releases: sin build ni revisión de tienda.
- Checks mínimos en GitHub Actions al abrir PR: `typecheck`, `biome`, `vitest`.

**Costo aproximado: $0.**

### Fase 2 — Al escalar

Migrar cuando ocurra **cualquiera** de estas señales:

- Más de un desarrollador tocando el repo a diario.
- Releases móviles con frecuencia semanal o mayor.
- El negocio ya depende de la app (downtime = pérdida de dinero).

Entonces:

- **GitHub Actions como pipeline único**: tests, typecheck, E2E de Playwright, build web y
  disparo de EAS Build/Update.
- Builds móviles automáticos por rama de release, con submit a TestFlight y al internal track
  de Google Play.
- **Plan de producción de EAS** cuando la cola gratuita sea cuello de botella.
- Migraciones de base de datos aplicadas desde el pipeline, nunca a mano.
- **Entorno de staging real**, con su propia base de datos y sus propias claves.

---

## 18. Infraestructura y hosting

### Por defecto: Vercel + Supabase Cloud

Cero operación: deploys automáticos, CDN global, escalado transparente, previews por PR.
Es la opción correcta mientras el costo sea menor al tiempo de ingeniería que ahorra.

### Alternativa: Docker + VPS

Se migra cuando el costo mensual de Vercel + Supabase supere de forma sostenida lo que cuesta un
VPS más el tiempo de mantenerlo. En la práctica el disparador suele ser ancho de banda alto,
funciones de larga duración, o procesamiento pesado.

- Next.js en modo `standalone` dentro de un contenedor Docker.
- **Caddy** o Traefik como reverse proxy, con TLS automático.
- Postgres en contenedor con backups automatizados **y verificados** — un backup sin restore
  probado no es un backup.
- Proveedores: Hetzner (mejor precio/rendimiento), DigitalOcean, o infraestructura propia.

**Regla de portabilidad:** el proyecto debe poder desplegarse en ambos desde el día uno. Nada de
APIs exclusivas de Vercel en la lógica de negocio.

---

## 19. Servicios de terceros (estándar en todo proyecto)

| Servicio | Para qué | Notas |
|---|---|---|
| **Stripe** | Pagos y suscripciones | Checkout + Customer Portal. El estado se sincroniza por **webhook**, con idempotencia (§11.3). Nunca se confía en el redirect del cliente. |
| **Resend** | Email transaccional | Plantillas con React Email, versionadas en el repo. Dominio propio con SPF/DKIM/DMARC. |
| **Sentry** | Errores y crashes | Web y móvil. Source maps por build, releases etiquetadas. |
| **PostHog** | Analítica de producto | Eventos, funnels, session replay y feature flags. Los flags sirven también para rollout gradual. |
| **Cloudflare R2** | Archivos y media | Ver §12. |

Los secretos viven en el gestor de variables de entorno de cada plataforma (Vercel, EAS Secrets,
Docker secrets). **Nunca en el repo**, y ninguna clave secreta en bundles de cliente.

---

## 20. Seguridad

**Desde el día uno:**

- **RLS activo en todas las tablas.** Una tabla sin política es una tabla pública.
- Validación de todo input con Zod en el servidor. El cliente no es de fiar.
- Variables de entorno validadas al arrancar — que falle en el build, no en producción.
- La `service_role` key de Supabase **jamás** sale del servidor.
- Headers de seguridad en Next.js: CSP, HSTS, `X-Frame-Options`, `Referrer-Policy`.
- CORS restringido a dominios conocidos.
- Rate limiting en auth y endpoints públicos (§11.3).
- Verificación de firma en todos los webhooks entrantes.
- Dependabot o Renovate para parches de seguridad automáticos.

**Al escalar:**

- **Cloudflare WAF** delante de la app: bots, DDoS, reglas por país o patrón.
- **Log de auditoría** de acciones sensibles (quién cambió qué y cuándo). Requisito común en venta
  enterprise, y muy caro de agregar retroactivamente porque hay que reconstruir historia.
- Rotación de secretos y gestión centralizada (Doppler o Infisical).
- Pentest y revisión de dependencias antes de certificaciones.
- Cumplimiento: GDPR/habeas data — exportación y borrado de datos del usuario, retención definida,
  consentimiento de cookies.

---

## 21. Observabilidad

**Desde el día uno:**

- **Sentry** para errores y crashes, con alertas a Slack.
- **PostHog** para comportamiento de producto.
- **Pino** para logs estructurados en JSON. Con `requestId` correlacionado. Nunca `console.log` en
  producción, y nunca datos personales ni tokens en los logs.
- **Uptime monitoring** externo (BetterStack o UptimeRobot, tier gratuito). Si tu app se cae, te
  enteras tú antes que el cliente.

**Al escalar:**

- **OpenTelemetry** para trazas distribuidas cuando hay más de un servicio.
- Logs centralizados y buscables (BetterStack, Axiom o Grafana Loki).
- Dashboards de métricas de negocio, no solo técnicas.
- Alertas con on-call definido y umbrales acordados.

---

## 22. Fundamentos: qué es imprescindible en cada fase

### Fase 1 — Imprescindible para arrancar

Sin esto no se lanza. Todo cabe en tiers gratuitos o casi.

| Área | Qué |
|---|---|
| Tipado y validación | TypeScript estricto + Zod en todos los bordes |
| Seguridad | RLS en todas las tablas, secretos fuera del repo, headers, rate limit en auth |
| Datos | Backups automáticos **con restore probado** |
| Conexiones | Pooler de Postgres configurado (§9.3) |
| Errores | Sentry en web y móvil |
| Producto | PostHog con los eventos clave definidos |
| Logs | Pino estructurado con requestId |
| Uptime | Monitor externo con alerta |
| CI | typecheck + lint + tests en cada PR |
| Deploy | Previews por rama y rollback de un clic |
| Móvil | Push notifications y deep links configurados |
| Legal | Términos, privacidad, borrado de cuenta (**Apple lo exige**) |
| i18n | Decidir si habrá multi-idioma. Con `next-intl` desde el inicio cuesta poco; retroactivo es carísimo |
| Accesibilidad | Semántica correcta, foco visible, contraste. Es barato al escribir y caro al corregir |
| Documentación | README + `CLAUDE.md` + ADRs para decisiones importantes |
| Onboarding | Un comando para levantar el proyecto desde cero |

### Fase 2 — Cuando escala

Se agrega cuando el disparador correspondiente se cumple, no antes.

| Área | Qué | Disparador |
|---|---|---|
| Caché | Redis / Upstash | Queries lentas repetidas, o necesidad de rate limit distribuido |
| Colas | QStash → Inngest → BullMQ | Trabajos que fallan sin reintento y cuestan dinero |
| Base de datos | Réplicas de lectura, particionado, TimescaleDB | Reportes que degradan la operación |
| Backend | Servicio separado en Hono | Ver §8.1 |
| Tiempo real | MQTT + TimescaleDB | Miles de dispositivos o alta frecuencia |
| CI/CD | Pipeline completo con builds móviles automáticos | Equipo de 2+ o releases semanales |
| Entornos | Staging real con datos propios | Primer cliente que no tolera un bug en producción |
| Observabilidad | OpenTelemetry + logs centralizados | Más de un servicio desplegado |
| Seguridad | WAF, audit log, rotación de secretos | Primer cliente enterprise o dato regulado |
| Auth | SSO SAML/OIDC + SCIM | Requisito de venta corporativa |
| Rendimiento | Tests de carga con k6 | Antes de una campaña o lanzamiento grande |
| Infraestructura | VPS/ECS, multi-región | Costo sostenido de Vercel > VPS + operación |
| Datos | Data warehouse + BI | Cuando PostHog ya no responde las preguntas del negocio |
| Soporte | Intercom o similar, con estado de la app | Volumen de tickets que no da el email |

---

## 23. Checklist para arrancar un proyecto nuevo

- [ ] Monorepo con Turborepo + pnpm, estructura de §14
- [ ] `tsconfig` estricto y Biome desde `packages/config`
- [ ] Variables de entorno validadas con Zod al arrancar
- [ ] Supabase creado, esquema inicial y **RLS activo en todas las tablas**
- [ ] Pooler de conexiones configurado (§9.3)
- [ ] Auth en web y móvil, con sesión en `expo-secure-store`
- [ ] Design tokens definidos antes del primer componente
- [ ] `CLAUDE.md` y `.mcp.json` versionados en el repo
- [ ] Sentry, PostHog y Pino instalados desde el día uno
- [ ] Monitor de uptime externo con alerta
- [ ] Rate limiting en login y registro
- [ ] GitHub Actions con `typecheck` + `biome` + `vitest` en PR
- [ ] Vercel conectado con previews por rama
- [ ] EAS configurado, con un build de desarrollo en dispositivo real
- [ ] Push notifications y deep links funcionando
- [ ] Backups configurados **y un restore probado**
- [ ] Términos, privacidad y borrado de cuenta publicados
- [ ] README con las decisiones que se desvían de este documento

---

## 24. Evaluadas y descartadas (y por qué)

| Opción | Por qué no |
|---|---|
| Flutter | Excelente framework, pero no comparte código ni ecosistema con la web en React. Obligaría a mantener dos culturas técnicas. |
| Firebase | NoSQL complica reportes y relaciones; el lock-in es mucho más fuerte que con Supabase, que al final es Postgres estándar. |
| GraphQL | Resuelve coordinación entre varios equipos frontend y un backend compartido. No es nuestro caso, y cuesta caro en caché y control de queries. |
| Redux Toolkit | Resuelve un problema que TanStack Query + Zustand ya cubren con menos código. |
| Clerk | Muy buen producto, pero fragmenta la auth fuera de la base y encarece al escalar. |
| Tamagui | La UI universal suena mejor de lo que resulta; compartir tokens rinde casi lo mismo con mucha menos complejidad. |
| ESLint + Prettier | Biome hace ambos, en una herramienta y órdenes de magnitud más rápido. |
| Express | Hono hace lo mismo con tipado real y corre en más runtimes. No hay razón para empezar un proyecto nuevo en Express. |
| Kafka | Sobredimensionado para nuestro volumen. MQTT + Postgres cubre la ingesta IoT con una fracción de la operación. |

---

## 25. Revisión

Este documento se revisa **cada 6 meses**, o antes si una decisión se demuestra equivocada en un
proyecto real. Los cambios se registran aquí.

| Fecha | Versión | Cambio |
|---|---|---|
| 2026-09-12 | 1.0 | Versión inicial. |
| 2026-09-12 | 1.1 | Backend (stack, contrato de API, hosting); caché, colas, cron y Redis; almacenamiento de archivos como sección propia (R2 principal, Supabase Storage alternativa); diseño/prototipado con IA y MCP; seguridad; observabilidad; fundamentos por fase; connection pooling; alternativas de monorepo evaluadas. |
