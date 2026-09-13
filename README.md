# Stack Tecnológico — Apps Web y Móviles

> **Autor:** Michael Santiago · **Última actualización:** 2026-09-12 · **Versión:** 2.0

## Qué es esto

El stack por defecto con el que construyo apps web y móviles: TypeScript de punta a punta,
Next.js, React Native, Postgres y NestJS cuando hace falta. Está escrito como estándar de
trabajo, no como lista de tecnologías favoritas — cada elección trae la razón por la que está
ahí.

Es un estándar por defecto, no un dogma: desviarse es válido, pero debe justificarse por escrito
en el README del proyecto que se desvía. 

## Cómo leer este documento

Está organizado en **dos Stack de operación**:

| Stack | Qué es |
|---|---|
| **Stack A — Stack base** (§1–§20) | Sin backend dedicado. Toda la lógica de servidor vive dentro de Next.js: Server Actions y Route Handlers. **Es el punto de partida de todo proyecto nuevo** y cubre cómodamente la mayoría de los productos. |
| **Stack B — Backend dedicado** (§21–§27) | NestJS como servicio propio. No es un stack distinto: es un **delta** sobre el Stack A — ¿qué se retira?, ¿qué se reemplaza? y ¿qué se agrega?, para más información visitar §23. |

Las sección §28 aplican a los dos Stacks.

Muchas secciones están escritas en la opción más barata para arrancar y la
recomendable al escalar.

---

# STACK BASE

## 1. Resumen

| Capa | Stack A — Stack base | Stack B — Backend dedicado |
|---|---|---|
| Lenguaje | TypeScript (modo estricto) + Zod | igual |
| Web | Next.js (App Router) | igual, pero sin lógica de negocio |
| Móvil | React Native + Expo | igual |
| UI | Tailwind CSS + shadcn/ui (web) · NativeWind (móvil) | igual |
| Estado / datos | TanStack Query + Zustand | igual |
| Lógica de servidor | Server Actions + Route Handlers | **NestJS** (módulos, DI, guards) |
| Contrato de API | REST + OpenAPI generado desde Zod | igual, generado con `nestjs-zod` |
| Base de datos | Supabase (Postgres gestionado) · alternativa: Postgres + Drizzle | igual, pero el acceso pasa solo por el backend |
| Auth | Supabase Auth | Supabase Auth + verificación del JWT en NestJS |
| Autorización | RLS en la base | Guards en el backend · RLS como defensa en profundidad |
| Caché | Caché de Next.js → Redis (Upstash) | Redis, obligatorio desde el inicio |
| Colas y cron | Vercel Cron + pg_cron → QStash / Inngest | **BullMQ** sobre Redis, con worker propio |
| Archivos | Cloudflare R2 (principal) · Supabase Storage (alternativa) | igual |
| Tiempo real | Supabase Realtime | igual (+ MQTT si hay telemetría) |
| Monorepo | Turborepo + pnpm | igual, + `apps/api` y `apps/worker` |
| Testing | Vitest (unit) + Playwright (E2E) | + Supertest para la API |
| Lint / Format | Biome | igual |
| Hosting | Vercel | Vercel (frontends) + contenedor (API y worker) |
| Pagos | Stripe | igual, webhooks en el backend |
| Email | Resend | igual |
| Observabilidad | Sentry + PostHog + Pino | + OpenTelemetry |
| Diseño / IA | v0 + Figma (Dev Mode MCP) + Claude Code con MCP | igual |

---

## 2. Productos

| Producto | Qué es | Tecnología | Dominio |
|---|---|---|---|
| **Landing** | Sitio público de marketing: qué es el producto, precios, blog, SEO. | Next.js estático (SSG). Astro solo si es puramente informativa y jamás tendrá sesión. | `dominio.com` |
| **Web clientes** | El producto en sí, tras login. | Next.js (App Router) | `app.dominio.com` |
| **Web backoffice** | Panel interno: soporte, administración, métricas, gestión de cuentas. | Next.js (App Router) | `****.dominio.com` |
| **Móvil** | App iOS/Android para el cliente final. | React Native + Expo | tiendas + deep links |

---

## 3. Lenguaje y validación

- **TypeScript** con `strict: true`. Sin `any` implícito.
- **Zod** para validar todo lo que cruza una frontera: formularios, respuestas de API, variables
  de entorno, payloads de webhooks. Los esquemas de Zod viven en `packages/shared` y se usan en los cuatro productos y en el
  backend. Un solo esquema es a la vez la validación, la fuente del tipo (`z.infer`) y el esquema
  OpenAPI que consumen los clientes (§8).

---

## 4. Web — Next.js (App Router)

- **Next.js** con App Router y React Server Components.
- **Server Actions** para los cambios en los datos (mutaciones) de las apps web.
- **Route Handlers** (`app/api/*`) para todo lo que no es un formulario de la propia web: la API
  que consume la app móvil, los webhooks entrantes y los endpoints públicos (§8).
- SSR/ISR para páginas públicas e indexables; client components solo donde hay interactividad real.

**Nota:** una Server Action es código de servidor con apariencia de función local. Se
valida su entrada con Zod y se verifica la sesión **dentro** de la acción, siempre. Que el botón
esté detrás del login no es control de acceso.

**Cuándo NO usar Next.js:** una landing puramente estática y sin sesión puede ir en Astro.

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
| Web (landing, apps web) | Tailwind CSS + shadcn/ui (sobre Radix UI) |
| Móvil | Tailwind vía NativeWind |

---

## 7. Estado y datos en el cliente

- **TanStack Query** — todo lo que viene del servidor: caché, reintentos, invalidación, estados
  de carga y error. Funciona igual en web y en Expo.
- **Zustand** — estado global del frontend (tema, filtros, wizard abierto, sesión en memoria).
  Nunca datos que ya son del servidor.
- **useState / Context** — estado local de un componente o subárbol.

**Regla:** si el dato tiene dueño en la base de datos, es de TanStack Query. Si solo existe
mientras la pantalla está abierta, es de Zustand o useState.

---

## 8. API y contrato de datos

| Camino | Para quién | Cómo |
|---|---|---|
| **Server Actions** | Formularios y mutaciones de las apps web. | Función tipada del servidor. Sin endpoint ni contrato que mantener. |
| **REST + OpenAPI** | App móvil, integraciones, clientes externos, dispositivos IoT. | Route Handlers con esquemas Zod, documentados automáticamente. |

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

**Postgres + Drizzle ORM**.

- Aplica si: el cliente exige datos en su propia infraestructura, hay requisitos de compliance o
  residencia de datos, o el volumen hace que Supabase salga más caro que un VPS.
- Drizzle da tipado completo desde el esquema y genera SQL predecible, sin sorpresas de rendimiento.
- Migraciones con `drizzle-kit`, versionadas en el repo.
- Postgres en Docker, o servicio gestionado (Neon / RDS) si se prefiere no operarlo.

---

## 10. Autenticación — Supabase Auth

- Email + contraseña, magic links y OAuth (Google, Apple — **Apple es obligatorio** si hay login
  social en iOS).
- En Expo, la sesión se persiste en `expo-secure-store`, nunca en AsyncStorage plano.
- Roles y permisos de cliente en una tabla propia, referenciada desde las políticas RLS (Row Level Security), es una seguridad de Postgres.
- **El backoffice usa el mismo proveedor pero otro modelo de permisos**: tabla `staff_roles`
  separada, políticas propias y, desde el inicio, **2FA obligatorio** para cualquier cuenta con
  acceso a datos de clientes.

> En el escenario autohospedado (§9.2), Supabase Auth puede seguir usándose como servicio
> independiente de la base, o reemplazarse por **Better Auth** sobre el mismo Postgres (decisión por proyecto).

**Al escalar (clientes enterprise):** Para cuando una empresa te diga "Quiero que mis empleados entren con nuestro Microsoft/Google corporativo y 
que podamos controlar sus cuentas.". SSO con SAML/OIDC, SCIM para aprovisionamiento de usuarios,
y log de auditoría.

---

## 11. Caché, colas y trabajos programados

### 11.1 Caché

**Opción 1 — Sin infraestructura adicional ($0):**

- Caché nativa de Next.js: `revalidate` en fetch, `unstable_cache` para funciones costosas,
  ISR para páginas.
- CDN de Vercel para assets y respuestas estáticas.
- TanStack Query en el cliente: evita refetch innecesario en web y móvil.

Esto cubre bastante más de lo que la gente asume. No se agrega Redis antes de tiempo.

**Opción 2 — Redis, cuando ocurre alguna de estas señales:**

- Queries que tardan cientos de milisegundos y se repiten mucho.
- Necesitas rate limiting real (contadores compartidos entre instancias).
- Hay trabajos en segundo plano con reintentos.
- Necesitas invalidar caché desde varios productos a la vez.

### 11.2 Trabajos programados (cron)

**Opción 1 ($0):**

| Herramienta | Para qué |
|---|---|
| **Vercel Cron** | Tareas HTTP: reportes diarios, limpieza, sincronizaciones. Se define en `vercel.json`. Verificar la frecuencia permitida por el plan. |
| **pg_cron** (Supabase) | Tareas que son puro SQL: purgar registros viejos, refrescar vistas materializadas, agregar métricas. Corre dentro de la base, sin red de por medio. |

**Opción 2 — cuando los trabajos crecen:**

| Herramienta | Para qué |
|---|---|
| **QStash** (Upstash) | Cola por HTTP con reintentos y programación. Funciona en serverless, sin servidor que mantener. Escalón natural desde Vercel Cron. |
| **Inngest** / **Trigger.dev** | Flujos de varios pasos, durables, con reintentos por paso y visibilidad de cada ejecución. Para procesos de negocio largos. |
| **BullMQ** sobre Redis | Máximo control y menor costo por trabajo. **Requiere un proceso worker siempre encendido** — no funciona en Vercel serverless. En la práctica, si ya necesitas esto, estás en el stack B (§21). |

**Disparador de migración:** cuando un trabajo fallido sin reintento automático empiece a costar
dinero o soporte, ya necesitas cola real.

---

## 12. Almacenamiento de archivos

| Opción | Cuándo |
|---|---|
| **Cloudflare R2** (principal) | Por defecto. **Sin cargos de egreso**, que es la diferencia grande con la competencia. |
| **Supabase Storage** (alternativa) | Cuando el proyecto ya vive en Supabase y el volumen es bajo: una dependencia menos, y las políticas de acceso usan el mismo RLS y el mismo `auth.uid()` que el resto. Se paga egreso. |

---

## 13. Tiempo real y telemetría

### Opción 1 — Inicio (mínimo costo)

**Supabase Realtime**.

- Los dispositivos y clientes escriben por API REST (Route Handler o PostgREST).
- La UI se suscribe a cambios de Postgres por WebSocket: actualizaciones en vivo sin
  infraestructura adicional.
- Incluido en el plan de Supabase, sin servicio extra que operar.

**Suficiente para:** dashboards en vivo, notificaciones, presencia, chat, y telemetría de baja
frecuencia — decenas o cientos de dispositivos reportando cada varios segundos.

### Opción 2 — Al escalar

Migrar cuando ocurra **cualquiera** de estas señales:

- Miles de dispositivos conectados simultáneamente.
- Escrituras de alta frecuencia (sub-segundo por dispositivo) que inflan la tabla y degradan queries.
- Dispositivos con red intermitente o restricciones de batería/ancho de banda, donde HTTP resulta
  demasiado costoso.
- Se necesitan consultas de series de tiempo: agregaciones por ventana, retención, downsampling.

Entonces:

- **MQTT** (EMQX o Mosquitto) como broker de ingesta: protocolo ligero, QoS configurable,
  conexiones persistentes. Diseñado exactamente para este caso.
- **Redis como buffer** (Buffer de telemetría IoT). Los dispositivos escriben a Redis a alta frecuencia 
  y un proceso vuelca lotes a Postgres/TimescaleDB cada X segundos. Evita miles de INSERT individuales. 
  También sirve para guardar el "último estado conocido" de cada dispositivo, 
  que es la consulta más frecuente de cualquier dashboard IoT.
- **TimescaleDB** — extensión de Postgres para series de tiempo: hypertables, compresión,
  agregados continuos y políticas de retención. Sigue siendo Postgres, el resto del stack no cambia.
- Supabase Realtime se mantiene para el frontend: MQTT alimenta la base, la base alimenta el frontend.

> Un broker MQTT necesita un proceso siempre encendido. Al escalar implica, casi siempre,
> cruzar también al Stack B (§21).

---

## 14. Monorepo — Turborepo + pnpm

```
proyecto/
├─ apps/
│  ├─ landing/        # Next.js estático
│  ├─ app/            # Next.js
│  ├─ backoffice/     # Next.js
│  └─ mobile/         # Expo
├─ packages/
│  ├─ shared/         # tipos, esquemas Zod, utilidades puras
│  ├─ db/             # esquema, migraciones, cliente de datos
│  ├─ ui/             # design tokens, componentes web, config de Tailwind
│  ├─ api-client/     # cliente REST tipado, generado desde OpenAPI (§9.1)
│  └─ config/         # tsconfig, biome, presets compartidos
├─ turbo.json
└─ pnpm-workspace.yaml
```

En stack B se agregan `apps/api` (NestJS) y `apps/worker` (§27).

### Alternativas evaluadas

| Opción | Veredicto |
|---|---|
| **Nx** | Más potente: generadores, gráfico de dependencias, caché distribuida. Justificado con 6+ apps o varios equipos. Para nuestra escala, la curva no se paga. |
| **pnpm workspaces sin Turborepo** | Inviable con cuatro apps: el CI reconstruye todo en cada push. |
| **Bun workspaces** | Muy rápido, pero Metro (bundler de Expo) todavía tiene fricciones. Revisar en el futuro. |
| **Yarn Berry (PnP)** | PnP rompe React Native/Metro. Sin PnP no aporta nada sobre pnpm. |
| **Repos separados** | Obliga a publicar los tipos como paquete npm privado; cada cambio de contrato son dos PRs y un release. Solo si son productos verdaderamente independientes. |

---

## 15. Desarrollo asistido por IA

### 15.1 MCP para Claude Code

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

### 15.2 Convenciones de repo para IA

Parte del andamiaje de cada proyecto:

- **`CLAUDE.md`** en la raíz: arquitectura, comandos, convenciones y decisiones del proyecto.
  Se genera con `/init` y se mantiene al día. Es lo que evita que la IA proponga cosas fuera del
  stack. En un monorepo de cuatro apps, cada app puede tener el suyo con sus particularidades.
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

**Al escalar:** tests de carga con **k6** y **Maestro** para E2E en móvil.

---

## 17. CI/CD

### Opción 1 — Inicio (mínimo costo)

- **Vercel** desplegando automáticamente con cada push: un proyecto por app web, preview por rama,
  producción en `main`.
- **EAS Build manual** — se compila cuando toca release. El tier gratuito alcanza al ritmo de
  releases de un producto nuevo.
- **EAS Update** para cambios de JS entre releases: sin build ni revisión de tienda.
- Checks mínimos en GitHub Actions al abrir PR: `typecheck`, `biome`, `vitest`, con filtros de
  Turborepo para correr solo lo afectado.

### Opción 2 — Al escalar

Migrar cuando ocurra **cualquiera** de estas señales:

- Más de un desarrollador tocando el repo a diario.
- Releases móviles con frecuencia semanal o mayor.
- El negocio ya depende de la app (downtime = pérdida de dinero).

Entonces:

- **GitHub Actions como pipeline único**: tests, typecheck, E2E de Playwright, build de las webs y
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

Las tres webs son tres proyectos de Vercel apuntando al mismo repo. El backoffice se protege
además a nivel de plataforma: restricción por IP o Vercel Authentication, para que `****.dominio.com` no sea
simplemente "otra URL pública con login".

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

**Nota crítica:** el procesamiento pesado y las cargas de GPU **nunca** van en funciones
serverless — los límites de tiempo y memoria no dan. Van en contenedores dedicados (ECS, EC2,
RunPod o infraestructura propia), invocados por cola. Si el producto tiene este componente, nace
directamente en el stack B.

---

## 19. Seguridad

**Desde el día uno:**

- **RLS activo en todas las tablas.** Una tabla sin política es una tabla pública.
- Validación de todo input con Zod en el servidor, **incluidas las Server Actions**. El cliente no
  es de fiar y una Server Action es un endpoint público con otro nombre.
- Variables de entorno validadas al arrancar — que falle en el build, no en producción.
- La `service_role` key de Supabase **jamás** sale del servidor.
- Headers de seguridad en Next.js: CSP, HSTS, `X-Frame-Options`, `Referrer-Policy`.
- CORS restringido a dominios conocidos.
- Rate limiting en auth y endpoints públicos (§12.3).
- Verificación de firma en todos los webhooks entrantes.
- **Backoffice:** 2FA obligatorio, sesión corta, y log de cada acción sobre datos de clientes.
- Dependabot o Renovate para parches de seguridad automáticos.
- Los secretos viven en el gestor de variables de entorno de cada plataforma (Vercel, EAS Secrets, 
  Docker secrets). **Nunca en el repo**, y ninguna clave secreta en bundles de cliente.

**Al escalar:**

- **Cloudflare WAF** delante de la app: bots, DDoS, reglas por país o patrón.
- **Log de auditoría** de acciones sensibles (quién cambió qué y cuándo). Requisito común en venta
  enterprise, y muy caro de agregar retroactivamente porque hay que reconstruir historia.
- Rotación de secretos y gestión centralizada (Doppler o Infisical).
- Pentest y revisión de dependencias antes de certificaciones.
- Cumplimiento: GDPR/habeas data — exportación y borrado de datos del usuario, retención definida,
  consentimiento de cookies.

---

## 20. Observabilidad

**Desde el día uno:**

- **Sentry** para errores y crashes, con alertas a Slack. Proyecto separado por superficie: un
  error del backoffice no debe perderse entre el ruido de la app pública.
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

# Stack B — BACKEND DEDICADO (NestJS)

Esta parte **no repite el stack**. Describe únicamente el delta contra el Stack base: cuándo cruzar,
qué se conserva, qué se retira, qué se reemplaza y qué se agrega. Lo que cambia es **dónde vive la lógica de negocio** y
**quién habla con la base**.

**No agregar un backend dedicado si:** lo único que pasa es que el proyecto "va creciendo". Crecer no
es un disparador. El stack base aguanta mucho más de lo que la gente supone. El backend dedicado agrega un
servicio que operar, desplegar, monitorear y pagar.

## 21. Cuándo pasar al backend dedicado

Cuando se cumple **cualquiera** de estas señales.

| Señal | Por qué obliga |
|---|---|
| **La app móvil es el cliente principal.** | No tiene sentido que la API de la que depende el producto viva dentro de un frontend web que es secundario. |
| **Procesos que superan el límite de las funciones serverless.** | Generación de reportes pesados, procesamiento de video, ETL, cargas de GPU. Los límites de tiempo y memoria no dan. |
| **Se necesita conexión persistente.** | WebSocket propio, broker MQTT, workers de cola consumiendo sin parar. Serverless no sostiene procesos. |
| **Un tercero va a consumir la API.** | Clientes, integradores o dispositivos necesitan un contrato estable, versionado, con su propio ciclo de vida y su propio rate limit. |
| **Lógica de negocio con muchos módulos y reglas.** | Facturación, inventario, agendamiento, motores de reglas. Cuando la lógica deja de ser CRUD, necesita estructura impuesta. |
| **Más de un desarrollador backend.** | Módulos, inyección de dependencias y límites claros dejan de ser ceremonia y pasan a ser lo que evita pisarse. |
| **Trabajos en cola que fallan y cuestan dinero.** | Si necesitas BullMQ, necesitas un proceso encendido (§12.2). |

### 21.1 Proyectos que nacen en stack B

Algunos productos arrancan directamente aquí:

- Plataformas IoT con ingesta de telemetría (§13, opción 2).
- Productos donde la app móvil **es** el producto y la web es secundaria.
- Sistemas con procesamiento pesado o GPU desde el día uno.
- Proyectos donde el cliente exige la API como entregable.

---

## 22. Qué NO cambia

- **TypeScript estricto + Zod**. Los mismos esquemas que validaban Server Actions ahora validan controllers de Nest.
- **Supabase como base de datos** (o Postgres + Drizzle autohospedado, §9.2).
- **Supabase Auth** como proveedor de identidad. El login sigue siendo el mismo lo que cambia es quién valida el token (§24).
- **Supabase Realtime** para la capa de UI en vivo.
- **Cloudflare R2** como almacenamiento principal, Supabase Storage como alternativa.
- **Los cuatro productos**: landing, web clientes, web backoffice y móvil. Siguen siendo Next.js
  y Expo, con Tailwind, shadcn/ui, NativeWind, TanStack Query y Zustand.
- **REST + OpenAPI desde Zod** como contrato (§8). Es el mismo contrato: solo cambia quién lo sirve.
- **Turborepo + pnpm**, **Biome**, **Vitest**, **Playwright**.
- **Stripe, Resend, Sentry, PostHog**.
- Todo lo de §19 (seguridad) y §20 (observabilidad), con las adiciones de §25.

---

## 23. Qué se retira

| Se retira | Por qué |
|---|---|
| **Server Actions como capa de negocio** | La mutación ya no la ejecuta el frontend. Las webs llaman a la API. Se conservan solo para cosas que son estrictamente del frontend: revalidación de caché, formularios de contacto de la landing, preferencias de UI. Ninguna Server Action vuelve a tocar la base directamente. |
| **Route Handlers como API del producto** | `app/api/*` deja de ser el backend. Sobreviven únicamente los que son de la propia web: callback de auth, health check, revalidate, y proxies de sesión. |
| **Acceso directo a la base desde el frontend** | `supabase-js` deja de usarse como cliente de datos en web y móvil. Las lecturas y escrituras pasan por la API. `supabase-js` se conserva **solo** para auth y para suscripciones Realtime. |
| **`service_role` en el frontend** | Deja de existir fuera del backend. Ninguna app web vuelve a tener una clave con poder sobre la base. |
| **Vercel Cron como orquestador** | Lo reemplaza el scheduler del backend (§26). `pg_cron` sobrevive, pero solo para mantenimiento puramente SQL: purgas, vistas materializadas, agregados. |
| **QStash / Inngest / Trigger.dev** | Fueron la solución a "colas sin servidor". Ya hay servidor: BullMQ es más barato por trabajo y da control total. Inngest se conserva solo si el producto tiene flujos de negocio largos de varios pasos donde su visibilidad vale el costo. |
| **Supavisor en modo transacción para el backend** | Era necesario porque cada invocación serverless abría su conexión. NestJS es un proceso persistente con pool propio. El pooler sigue si quedan funciones serverless en las webs. |

**Lo que NO se retira aunque lo parezca:** la caché nativa de Next.js sigue viva para páginas e
ISR. Lo que se va es `unstable_cache` sobre datos de negocio, que ahora cachea el backend.

---

## 24. Qué se reemplaza

| Del stack base | Pasa a ser | Nota |
|---|---|---|
| Server Actions + Route Handlers | **NestJS**: controllers, providers, módulos, guards, interceptores, pipes | Un módulo por dominio de negocio. La estructura impuesta es el punto: es lo que hace que el código de tres desarrolladores se parezca. |
| Zod suelto en cada handler | **`nestjs-zod`** (`createZodDto` + `ZodValidationPipe`) | Consume los **mismos** esquemas de `packages/shared`. No se reescribe nada. |
| OpenAPI con `zod-openapi` | **`@nestjs/swagger` + `nestjs-zod`** | El documento se genera del mismo Zod. Los clientes de `packages/api-client` se regeneran y siguen tipados igual. |
| Sesión leída con `supabase-js` en el servidor de Next | **Verificación del JWT de Supabase en NestJS** | Guard con `passport-jwt` validando contra el JWKS de Supabase. El token que emite Supabase Auth es el mismo; cambia quién lo verifica. |
| **Autorización por RLS** | **Guards + CASL en el backend** | Este es el cambio conceptual más importante. Ver §26.1. |
| `supabase-js` como cliente de datos | **Cliente OpenAPI generado** (`orval` / `openapi-fetch`) en los cuatro productos | Una sola forma de hablar con los datos, idéntica en web y móvil. |
| Cliente de base en el frontend | **Drizzle dentro de NestJS**, con el esquema en `packages/db` | El esquema deja de ser compartido con el frontend: solo el backend importa `packages/db`. |
| Migraciones con Supabase CLI | **`drizzle-kit` ejecutado desde el pipeline** | Sigue siendo el Postgres de Supabase. Cambia quién es dueño del esquema: el repo del backend, no el dashboard. |
| `unstable_cache` de Next | **`CacheModule` de Nest sobre Redis** | La caché de páginas de Next se queda; la de datos de negocio se muda. |
| Vercel Cron | **`@nestjs/schedule`** + BullMQ repeatable jobs | Los trabajos programados viven junto a la lógica que ejecutan, con reintentos reales. |
| QStash / Inngest | **BullMQ sobre Redis**, con `apps/worker` | Ver §27. |
| Supavisor (modo transacción) | **Pool de conexiones del proceso** (`pg.Pool` vía Drizzle) | Un proceso, un pool, conexiones reutilizadas. Más eficiente que cualquier pooler externo. |
| Pino en Route Handlers | **`nestjs-pino`** con `AsyncLocalStorage` | `requestId` propagado automáticamente por toda la cadena, incluidos los jobs de la cola. |
| Sentry solo en frontends | **+ Sentry Node SDK en API y worker** | Los errores de negocio ahora ocurren en el backend; ahí es donde hay que verlos. |
| Vercel como único hosting | **Vercel (landing, app, admin) + contenedor (api, worker)** | Ver §28. |

### 24.1 Qué pasa con RLS

En stack base, RLS es la línea de defensa principal: el cliente habla con Postgres y la base decide
qué puede ver. Ahora, NestJS se conecta con una identidad de servicio, así que **RLS ya no
puede ser la única defensa** — pasaría todo.

La decisión:

- **La autorización real vive en el backend**: guards de NestJS + CASL para permisos finos, con
  el `userId` y el rol extraídos del JWT verificado. Toda query lleva el filtro de tenencia
  (`orgId`, `userId`) explícito en el código, no implícito en una política.
- **RLS se mantiene activo**, como defensa en profundidad y porque **sigue gobernando el acceso
  directo del cliente**: las suscripciones de Supabase Realtime siguen conectándose con la clave
  anónima y el JWT del usuario. Si RLS se apagara, cualquiera con la anon key leería todo por
  Realtime.
- **Regla dura:** una tabla nueva sigue naciendo con RLS activo y política restrictiva, aunque el
  backend sea quien la consulte. El costo es cero y el día que alguien exponga una lectura directa,
  la base ya está cubierta.

---

## 25. Qué se agrega

### 25.1 El backend

- **NestJS** con estructura por módulos de dominio.
- **`@nestjs/config` + Zod** para validar las variables de entorno al arrancar. Si falta una, el
  proceso no levanta.
- **`@nestjs/terminus`** para health checks: `/health/live` y `/health/ready`. Los orquestadores
  (Railway, Fly, ECS) los necesitan para no enrutar tráfico a un contenedor que aún no está listo.
- **`@nestjs/throttler` sobre Redis** para rate limiting, ahora por endpoint y por rol.
- **CORS explícito**: solo los dominios de las tres webs. La app móvil no lo necesita.
- **Tests de API con Supertest** (`@nestjs/testing`), además de los Vitest existentes.

### 25.2 Colas y trabajos

- **Redis deja de ser opción 2 y pasa a ser obligatorio.** En stack B es infraestructura base: caché,
  colas, rate limiting, locks e idempotencia.
- **BullMQ** con colas por tipo de trabajo, reintentos con backoff exponencial y **dead-letter
  queue** revisable desde el backoffice.
- **`apps/worker`**: proceso separado que consume las colas. Separado de la API a propósito — un
  job pesado no puede degradar el tiempo de respuesta de la API, y cada uno escala por su lado.
- **Bull Board** montado en el backoffice para ver, reintentar y purgar trabajos sin entrar a Redis.

### 25.3 Infraestructura

- **Docker** para API y worker, con **docker-compose** para el entorno de desarrollo local
  (Postgres, Redis, API, worker en un solo comando).
- **Staging real**: su propio proyecto de Supabase, su propia Redis y su propia API desplegada.
  Con dos servicios más, probar en producción deja de ser una opción.
- **Migraciones aplicadas desde el pipeline**, con la API arrancando solo después de que corran.

### 25.4 Observabilidad y seguridad

- **OpenTelemetry**: ya hay más de un servicio, así que las trazas distribuidas dejan de ser lujo.
  Una petición debe poder seguirse desde el clic en la web hasta el job en la cola.
- **Log de auditoría** en tabla propia: quién, qué, cuándo, desde dónde. En stack B es barato porque
  todo pasa por el mismo interceptor de NestJS.
- **Secretos centralizados** (Doppler o Infisical): ahora hay cinco entornos de ejecución
  distintos con las mismas claves, y copiarlas a mano en cada uno se vuelve la fuente de errores.

### 25.5 Opcionales, según el producto

| Se agrega | Cuándo |
|---|---|
| **`@nestjs/websockets`** (Socket.IO) | Cuando se necesita push del servidor al cliente que Supabase Realtime no cubre: notificaciones de procesos largos, colaboración en vivo. |
| **Broker MQTT** (EMQX / Mosquitto) | Telemetría IoT de alta frecuencia (§14, fase 2). NestJS lo consume con `@nestjs/microservices`. |
| **TimescaleDB** | Series de tiempo con agregación y retención. |

---

## 26. Arquitectura y monorepo en stack B

```
proyecto/
├─ apps/
│  ├─ landing/        # Next.js estático — dominio.com
│  ├─ app/            # Next.js — app.dominio.com (clientes)
│  ├─ backoffice/     # Next.js — ****.dominio.com (backoffice)
│  ├─ mobile/         # Expo
│  ├─ api/            # NestJS — api.dominio.com
│  └─ worker/         # Consumidor de colas BullMQ
├─ packages/
│  ├─ shared/         # tipos y esquemas Zod — los consumen frontends Y backend
│  ├─ db/             # esquema Drizzle y migraciones — SOLO lo importa el backend
│  ├─ ui/             # design tokens, componentes web
│  ├─ api-client/     # cliente REST tipado, generado desde el OpenAPI de apps/api
│  └─ config/         # tsconfig, biome, presets
├─ docker-compose.yml
├─ turbo.json
└─ pnpm-workspace.yaml
```

**Flujo de una petición:**

```
Cliente (web o móvil)
  └─ token de Supabase Auth
      └─ NestJS  ──guard: verifica JWT (JWKS de Supabase)
                 ──guard: CASL, permisos del rol
                 ──pipe:  valida con el Zod de packages/shared
                 ──service ──> Postgres (Supabase) vía Drizzle
                            └─> Redis (caché / cola)
                                 └─> worker ──> Postgres, R2, Resend, Stripe

UI en vivo: cliente ──suscripción Realtime──> Supabase (gobernado por RLS)
```

**Hosting:**

| Componente | Dónde | Por qué |
|---|---|---|
| landing, app, admin | **Vercel** | Sin cambios. CDN, previews por rama, ISR. |
| **api** | **Railway** o **Fly.io** al inicio; VPS al escalar | Contenedor siempre encendido, deploy por git push, barato. |
| **worker** | El mismo proveedor que la API, servicio aparte | Escala independiente. Un pico de trabajos no afecta la latencia de la API. |
| **Redis** | Upstash, o Redis en contenedor junto a la API | Si ya hay contenedores, Redis propio sale más barato a volumen alto. |
| Postgres | **Supabase** (sin cambios) | O autohospedado según §10.2. |
| Cargas GPU / procesamiento pesado | Contenedor dedicado (ECS, EC2, RunPod, infraestructura propia) | **Nunca** en la API ni en serverless. Se invoca por cola desde el worker. |

---

## 27. Migración A → B, paso a paso

El orden importa: cada paso deja el sistema funcionando. No hay un "gran switch".

1. **Crear `apps/api` vacío pero desplegado.** NestJS con health check, config validada y CI. Que
   exista y se despliegue antes de que tenga lógica.
2. **Mover el esquema a `packages/db` con Drizzle**, apuntando al mismo Postgres de Supabase.
   Generar el esquema desde la base existente (`drizzle-kit introspect`). Nada cambia todavía.
3. **Montar Redis y el worker**, aunque la primera cola sea trivial (envío de emails). Valida el
   despliegue de un proceso persistente antes de que sea crítico.
4. **Migrar el primer módulo de dominio**, el más aislado y menos crítico. Endpoint en Nest,
   regenerar `packages/api-client`, y que la web lo consuma. Server Action correspondiente se borra.
5. **Migrar los webhooks.** Stripe y cualquier entrante pasan a Nest, con idempotencia en Redis. 
   Se cambia la URL en el dashboard del proveedor y se verifica con un evento de prueba.
6. **Migrar los trabajos programados**: de Vercel Cron a `@nestjs/schedule` + BullMQ, uno por uno.
7. **Migrar el resto de módulos**, del menos al más crítico. Auth y pagos al final.
8. **Cortar el acceso directo del frontend a la base.** Rotar la `service_role` key y quitarla de
   las variables de entorno de Vercel. Este es el punto de no retorno y la verificación real de que
   la migración terminó.
9. **Revisar RLS** bajo el criterio de §26.1: sigue activo, ahora como defensa en profundidad.
10. **Agregar OpenTelemetry** y verificar que una traza cruza web → API → worker completa.

**Qué NO hacer:** migrar todo en una rama larga. Cada módulo migrado va a `main` y a producción por
su cuenta. Una rama de migración de tres semanas se vuelve imposible de mergear y nadie se atreve a
desplegarla.

---

# TRANSVERSAL

## 28. Evaluadas y descartadas

| Opción | Por qué no |
|---|---|
| **tRPC** | Tipado sin generar código, pero solo dentro del monorepo y con backend TypeScript acoplado. Se rompe con consumidores externos, dispositivos y NestJS. OpenAPI desde Zod da lo mismo y sobrevive al cambio de modo (§9.2). |
| **Hono** | Excelente framework, y fue durante un tiempo nuestra etapa intermedia de backend. Se elimina porque obligaba a migrar dos veces: primero sacar el backend de Next.js a Hono, después reestructurarlo a NestJS al crecer el dominio. Dos modos claros cuestan menos que tres. Sigue siendo la elección correcta para un microservicio suelto con un solo propósito. |
| Express | Sin tipado real y sin estructura. Para un servicio ligero hay opciones mejores; para uno grande, NestJS. No hay razón para empezar un proyecto nuevo en Express. |
| Flutter | Excelente framework, pero no comparte código ni ecosistema con la web en React. Obligaría a mantener dos culturas técnicas. |
| Firebase | NoSQL complica reportes y relaciones; el lock-in es mucho más fuerte que con Supabase, que al final es Postgres estándar. |
| GraphQL | Resuelve coordinación entre varios equipos frontend y un backend compartido. No es nuestro caso, y cuesta caro en caché, control de queries y rate limiting. REST versionado es más simple de documentar y de consumir desde dispositivos. |
| Redux Toolkit | Resuelve un problema que TanStack Query + Zustand ya cubren con menos código. |
| Clerk | Muy buen producto, pero fragmenta la auth fuera de la base y encarece al escalar. |
| Tamagui | La UI universal suena mejor de lo que resulta; compartir tokens rinde casi lo mismo con mucha menos complejidad. |
| ESLint + Prettier | Biome hace ambos, en una herramienta y órdenes de magnitud más rápido. |
| Prisma | Buen DX, pero el cliente generado pesa en serverless y el SQL que produce sorprende en queries complejas. Drizzle es más predecible y el esquema es TypeScript plano. |
| Kafka | Sobredimensionado para nuestro volumen. MQTT + Postgres cubre la ingesta IoT con una fracción de la operación. |

---

## Licencia

© 2026 Michael Santiago. Este documento se publica bajo licencia
[Creative Commons Attribution 4.0 International](https://creativecommons.org/licenses/by/4.0/)
(CC BY 4.0): puedes copiarlo, adaptarlo y usarlo en tus propios proyectos, incluso
comercialmente, siempre que des crédito.

Si lo adaptas para tu equipo, me interesa saberlo — sobre todo si llegaste a una conclusión
distinta en alguna decisión. El texto completo de la licencia está en [LICENSE](LICENSE).
