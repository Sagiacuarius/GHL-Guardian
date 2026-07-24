# Arquitectura: GHL Guardian (MVP Interno) — v2

> **Decisión**: Next.js Full-Stack + Supabase (Opción A)
> **Fecha original**: 2026-07-21 · **Actualizado**: 2026-07-22
> **Stack base**: wacrm template patterns (Next.js 16, React 19, Supabase, shadcn/ui, Tailwind 4)
>
> **Resumen de cambios en esta revisión** (verificados contra documentación oficial de Vercel y HighLevel, y contra la aclaración de negocio de que "cliente" ≠ "subcuenta"):
> 1. Corregida la URL base de la API de GHL.
> 2. Reemplazado el supuesto de OAuth por subcuenta por autenticación de **Agencia** (PIT + Location Token Exchange) — único login, como pediste.
> 3. Agregado el modelo `Client → Subaccount[]` (antes se asumía 1 subcuenta = 1 cliente).
> 4. Señalado el bloqueo de Vercel Cron en plan Hobby (máx. 1 ejecución/día) — decisión pendiente antes de Fase 7.
> 5. Corregido el razonamiento del rate limiter en ADR-004 (el límite es por location, no global).
> 6. Marcado que el número real de subcuentas es mayor a 80 y debe obtenerse por API antes de recalcular capacidad.

---

## 1. Opciones Evaluadas

### Opción A: Next.js Full-Stack + Supabase ✅ (ELEGIDA)

| Capa | Tecnología | Justificación |
|---|---|---|
| Frontend | Next.js 16 (App Router) + React 19 | Single codebase, extraemos estilos de wacrm, SSR/SSG para landing futura |
| UI | shadcn/ui + Radix + Tailwind 4 + Recharts | Ya en wacrm, componentes accesibles, gráficos nativos para dashboard |
| Backend | Next.js API Routes + Server Actions | Sin servidor separado, lógica en `src/lib/` con Clean Architecture |
| Database | Supabase (PostgreSQL) | Event Store, Auth, RLS, Realtime para dashboard en vivo |
| ORM | Supabase JS Client (sin Prisma) | Consultas tipadas, RLS nativo, sin capa extra |
| Scheduling | Vercel Cron Jobs (Pro) **o** scheduler externo (GitHub Actions / cron-job.org) | Jobs cada 1-5 min para polling de GHL. **⚠️ Vercel Hobby limita los cron jobs a 1 ejecución/día — la cadencia de minutos requiere plan Pro (US$20/mes) o un scheduler externo que llame al endpoint por HTTP.** Decisión pendiente, ver ADR-008. |
| Testing | Vitest | Ya en wacrm, rápido, nativo para TypeScript |

**Ventajas**: single codebase, patrones de wacrm reutilizables, Supabase da realtime sin infra extra, shadcn/ui reduce tiempo de UI, TypeScript end-to-end.

**Desventajas**: Next.js API routes tienen menos estructura que NestJS (lo compensamos con Clean Architecture manual en `src/lib/`), Vercel Cron tiene límites en plan free — **específicamente, 1 ejecución/día en Hobby, lo cual es incompatible con polling cada 5 minutos sin upgrade o scheduler externo**.

**Riesgos**: rate limits de GHL (100req/10s **por location**, no global) — el volumen esperado por location es bajo (ver ADR-004 corregido), pero igual implementamos `p-limit` + backoff exponencial como margen de seguridad ante picos.

### Opción B: NestJS Backend + Next.js Frontend

Stack del brief original. **Descartada** por Leo: dos codebases agregan overhead sin beneficio para un MVP. NestJS brinda DI y scheduling nativos pero duplica infraestructura.

### Opción C: Híbrido Next.js + n8n

**Descartada**: agrega un sistema externo para tareas que Next.js + cron pueden manejar. n8n sería útil si el monitoreo tuviera workflows complejos multi-step, pero el MVP es polling + webhooks simples.

### Opción D: Conexión vía MCP oficial de GoHighLevel

**Evaluada y descartada** durante esta revisión. El MCP oficial de HighLevel (`services.leadconnectorhq.com/mcp/`) y los servidores MCP comunitarios están todos scoped a **una subcuenta (location) por conexión** — no existe una conexión MCP a nivel agencia que abarque todas las subcuentas de una vez. Con 150-400+ subcuentas reales, esto significaría una conexión MCP por subcuenta, lo cual no resuelve el requisito de login único. El camino correcto sigue siendo REST directo con token de Agencia (ver sección 4.4 y ADR-006).

---

## 2. Diagrama de Componentes (C4 — Nivel Contenedores)

```
┌─────────────────────────────────────────────────────────────┐
│  GHL Guardian (Next.js App)                                  │
│                                                              │
│  ┌─────────────┐  ┌──────────────────┐  ┌───────────────┐  │
│  │  (React SPA) │  │  API Routes      │  │  Cron Jobs    │  │
│  │              │  │  /api/guardian/* │  │  (Vercel Cron)│  │
│  └──────┬───────┘  └────────┬─────────┘  └───────┬───────┘  │
│         │                   │                     │          │
│         └───────────────────┼─────────────────────┘          │
│                             │                                │
│                    ┌────────┴────────┐                       │
│                    │   Health Engine │                       │
│                    │  (src/use-cases)│                       │
│                    └────────┬────────┘                       │
│                             │                                │
│         ┌───────────────────┼───────────────────┐           │
│         │                   │                   │           │
│  ┌──────┴──────┐   ┌───────┴───────┐   ┌───────┴──────┐    │
│  │ GHL Client  │   │  Event Store  │   │  Auth (Supabase)│   │
│  │ (REST API)  │   │  (PostgreSQL) │   │  + RLS         │   │
│  └──────┬──────┘   └───────────────┘   └───────────────┘    │
└─────────┼───────────────────────────────────────────────────┘
          │
    ┌─────┴─────┐
    │ GoHighLevel│
    │  (API)    │
    └───────────┘
```

**Nota sobre "GHL Client" (agregada en esta revisión):** este componente ya no asume un token único de larga vida por subcuenta. Internamente contiene un **Agency Token Manager** que guarda el PIT de agencia (estático, sin refresh automático) y lo intercambia on-demand por un location token de corta duración cada vez que el Health Engine necesita chequear una subcuenta puntual, cacheando ese location token en memoria con TTL corto. Ver sección 4.4 y ADR-006.

**Nota sobre "Auth (Supabase)":** este es el login de los operadores humanos al *dashboard de Guardian* — es un sistema de autenticación completamente separado del token de agencia de GHL. No hay que confundir "loguearse a Guardian" con "loguearse a GHL"; son dos capas de auth distintas.

---

## 3. Clean Architecture — Estructura de Carpetas

```
src/
├── domain/                    # Capa de dominio (entities, value objects, interfaces)
│   ├── entities/
│   │   ├── client.ts          # Client (id, name) — el negocio real que recibe el reclamo (NUEVO)
│   │   ├── subaccount.ts      # Subaccount (id, name, locationId, clientId) — actualizado con clientId
│   │   ├── health-event.ts    # HealthEvent (timestamp, service, status, signal)
│   │   └── monitor-config.ts  # MonitorConfig (thresholds, timeouts, baselines)
│   ├── value-objects/
│   │   ├── health-status.ts   # Healthy | Suspected | ConfirmedOffline | PossiblyStuck | Unknown
│   │   ├── service-type.ts    # workflow | oauth | whatsapp | email | phone
│   │   └── confidence-score.ts # 0-100 con breakdown de señales
│   └── ports/
│       ├── health-check.ts    # Interfaz: HealthCheck.run(subaccountId) → HealthEvent
│       ├── event-store.ts     # Interfaz: EventStore.append(event), query(filters)
│       ├── ghl-client.ts      # Interfaz: GHLClient.getWorkflows(), getOAuthStatus(), etc.
│       └── agency-token-provider.ts  # Interfaz: getLocationToken(locationId) → token (NUEVO)
│
├── use-cases/                 # Casos de uso (orquestan domain + ports)
│   ├── run-health-check.ts    # Ejecuta un HealthCheck, persiste evento
│   ├── run-all-checks.ts      # Itera subcuentas × monitores, aplica rate limiting
│   ├── calculate-confidence.ts # Lee eventos recientes, calcula score
│   ├── get-dashboard-state.ts  # Agrega estado actual de todas las subcuentas, con rollup por cliente
│   └── detect-incident.ts     # Detecta transición healthy→degraded, dispara alerta
│
├── adapters/                  # Implementaciones concretas de puertos
│   ├── ghl/
│   │   ├── ghl-rest-client.ts       # Cliente HTTP a GHL API v2
│   │   ├── agency-token-manager.ts  # PIT de agencia + intercambio por location token + cache (NUEVO)
│   │   ├── rate-limiter.ts          # Token bucket: 100 req/10s por location
│   │   └── oauth-strategies/        # Una estrategia por proveedor
│   │       ├── oauth-strategy.ts    # Interfaz común
│   │       ├── google-calendar.ts   # Detecta respuestas sospechosas (200 con items:[])
│   │       └── facebook.ts          # v2 — no incluido en MVP (ADR-002: un módulo por proveedor)
│   ├── monitors/               # Implementaciones de HealthCheck por servicio
│   │   ├── workflow-monitor.ts
│   │   ├── oauth-monitor.ts
│   │   ├── whatsapp-monitor.ts
│   │   ├── email-monitor.ts
│   │   └── phone-monitor.ts
│   ├── supabase/
│   │   ├── event-store.ts      # Implementación con Supabase JS Client
│   │   ├── client-repo.ts      # CRUD de clientes (NUEVO)
│   │   ├── subaccount-repo.ts  # CRUD de subcuentas
│   │   └── config-repo.ts      # Configuraciones de monitoreo
│   └── webhook/
│       └── ghl-webhook-handler.ts  # Recibe webhooks de GHL (workflow start/end, LCEmailStats)
│
├── ui/                        # Capa de presentación (React)
│   ├── dashboard/
│   │   ├── dashboard-page.tsx       # Página principal con grid de subcuentas
│   │   ├── client-group.tsx         # Agrupador visual: subcuentas por cliente (NUEVO)
│   │   ├── subaccount-card.tsx      # Card por subcuenta con health status
│   │   ├── service-indicator.tsx    # Indicador visual por servicio (🟢🟡🔴)
│   │   ├── confidence-score.tsx     # Componente de score con breakdown
│   │   └── incident-timeline.tsx    # Timeline de eventos por subcuenta
│   ├── config/
│   │   ├── monitor-config-form.tsx  # Form para timeouts, thresholds, baselines
│   │   └── subaccount-manager.tsx   # CRUD de subcuentas
│   ├── layout/
│   │   ├── app-layout.tsx           # Shell con sidebar + header
│   │   └── sidebar.tsx              # Nav: Dashboard, Config, Subcuentas
│   └── shared/
│       └── (componentes de wacrm: ui/*, tremor/*, etc.)
│
├── app/
│   ├── (dashboard)/guardian/        # Rutas del dashboard
│   │   ├── page.tsx                 # → DashboardPage
│   │   ├── config/page.tsx          # → ConfigPage
│   │   └── subaccounts/page.tsx     # → SubaccountManager
│   └── api/guardian/
│       ├── webhook/route.ts         # POST — recibe webhooks de GHL
│       ├── health/route.ts          # GET — dashboard state
│       ├── health/[subaccountId]/route.ts  # GET — detalle por subcuenta
│       ├── config/route.ts          # GET/PUT — configuración
│       └── cron/route.ts            # GET — endpoint para Vercel Cron o scheduler externo
│
└── lib/
    └── (extraído de wacrm: supabase client, auth, utils, themes, etc.)
```

---

## 4. Modelo de Dominio

### 4.1 Bounded Contexts

| Contexto | Responsabilidad |
|---|---|
| **Health Monitoring** | Ejecutar health checks, persistir eventos, calcular scores |
| **Configuration** | CRUD de subcuentas, timeouts, thresholds, baselines |
| **Dashboard** | Agregación de estado actual para UI, con rollup por cliente |
| **Webhook Ingestion** | Recibir y validar webhooks de GHL, convertirlos en HealthEvents |
| **Incident Detection** | Detectar transiciones de estado y notificar |
| **Agency Auth** | Gestionar el PIT de agencia y el intercambio por location tokens (NUEVO) |

### 4.2 Entidades Principales

```
Client                                  # NUEVO — el negocio real que recibe el reclamo
  ├── id: uuid
  ├── name: string
  └── subaccounts: Subaccount[]         // 1 a N+

Subaccount
  ├── id: string (GHL location ID)
  ├── clientId: string                  // FK a Client (NUEVO)
  ├── name: string
  ├── locationId: string
  └── monitors: MonitorConfig[]

MonitorConfig
  ├── serviceType: ServiceType
  ├── thresholds: { suspectedAfter, confirmedAfter }
  ├── workflowTimeout?: number       // por workflow
  └── activityBaseline?: number      // para Phone

HealthEvent (append-only, inmutable)
  ├── id: uuid
  ├── timestamp: DateTime
  ├── subaccountId: string
  ├── service: ServiceType
  ├── status: HealthStatus
  ├── rawSignal: JSON          // datos crudos de la señal
  ├── confidenceScore: number  // 0-100
  └── signalBreakdown: JSON    // desglose de señales que componen el score

HealthStatus (Value Object, enum)
  ├── Healthy
  ├── Suspected          // WhatsApp: sin actividad reciente, Workflows: possibly stuck
  ├── ConfirmedOffline   // WhatsApp: ventana superada, OAuth: token inválido
  ├── PossiblyStuck      // Workflows: pasó timeout sin webhook final
  └── Unknown            // Sin datos suficientes, o falla de auth (ver ADR-006)

ServiceType (Value Object, enum)
  ├── workflow
  ├── oauth
  ├── whatsapp
  ├── email
  └── phone
```

### 4.3 Health Check Interface (Puerto)

```typescript
// src/domain/ports/health-check.ts
export interface HealthCheckResult {
  status: HealthStatus;
  rawSignal: Record<string, unknown>;
  metadata?: { latencyMs: number; provider?: string };
}

export interface HealthCheck {
  readonly serviceType: ServiceType;
  readonly confidence: number; // Confiabilidad intrínseca del monitor (0-1)

  run(subaccountId: string, config: MonitorConfig): Promise<HealthCheckResult>;
}
```

**Confiabilidad por monitor** (basado en el brief):

| Monitor | Confidence | Mecanismo |
|---|---|---|
| Email (LC) | 1.0 | Webhook oficial LCEmailStats — señal nativa confiable |
| OAuth | 1.0 | HTTP status + validación de contenido (por proveedor) |
| Workflows | 0.95 | Webhooks start/end — no cubre colgados sin timeout |
| WhatsApp | 0.82 | Polling + webhook — latencia inherente |
| Phone (LC) | 0.65 | Sin webhook oficial — heurística de silencio |

### 4.4 Autenticación de Agencia (NUEVO)

```typescript
// src/domain/ports/agency-token-provider.ts
export interface AgencyTokenProvider {
  // El PIT de agencia es estático (no expira, no necesita refresh automático).
  // Vive en una variable de entorno / secrets manager, nunca en la base de datos.
  getLocationToken(locationId: string): Promise<{ token: string; expiresAt: Date }>;
  listLocations(): Promise<{ locationId: string; name: string }[]>;
}
```

**Flujo:**
1. Login único (manual, una vez): el owner de la agencia genera un PIT en `Settings → Private Integrations` a nivel Agencia (no dentro de una subcuenta). Requiere plan Agency Pro (US$497) — confirmado que los planes inferiores solo dan Location API Keys.
2. `AgencyTokenManager.listLocations()` llama a `GET /locations/search?companyId=...` con ese PIT — esto es lo que da el número real de subcuentas (ver sección 9, riesgo de capacidad).
3. Cuando el Health Engine necesita chequear una subcuenta puntual, `getLocationToken(locationId)` intercambia el PIT de agencia por un token scoped a esa location (endpoint `Get Location Access Token from Agency Token`), y lo cachea en memoria con TTL corto para no repetir el intercambio en cada request dentro del mismo ciclo.

No hay "login por subcuenta": el operador humano nunca ve ni gestiona 80+ credenciales distintas, solo el PIT de agencia.

---

## 5. Contratos de API

### 5.1 Dashboard State

```
GET /api/guardian/health
Response 200:
{
  "clients": [                                    // NUEVO — rollup por cliente real
    {
      "id": "client_xyz",
      "name": "Cliente A",
      "hasIncident": true,
      "subaccounts": [
        {
          "id": "loc_abc123",
          "name": "Cliente A - Sucursal Centro",
          "services": {
            "workflow":  { "status": "healthy",     "score": 95, "lastCheck": "..." },
            "oauth":     { "status": "healthy",     "score": 100, "lastCheck": "..." },
            "whatsapp":  { "status": "suspected",   "score": 72, "lastCheck": "..." },
            "email":     { "status": "healthy",     "score": 100, "lastCheck": "..." },
            "phone":     { "status": "unknown",     "score": null, "lastCheck": null }
          },
          "overallScore": 88,
          "incidentCount": 1
        }
      ]
    }
  ],
  "summary": {
    "totalClients": 80,
    "totalSubaccounts": null,          // pendiente de confirmar vía /locations/search — ver sección 9
    "fullyHealthy": null,
    "withIncidents": null,
    "degradedServices": { "whatsapp": 5, "workflow": 2, "oauth": 1 }
  }
}
```

*Nota: `totalSubaccounts` y los conteos derivados quedan como `null`/pendientes en este contrato hasta correr la consulta real a `/locations/search`; no conviene hardcodear 80 ahí, porque 80 es la cantidad de clientes, no de subcuentas.*

### 5.2 Detalle por Subcuenta

```
GET /api/guardian/health/{subaccountId}
Response 200:
{
  "subaccount": { ..., "clientId": "client_xyz", "clientName": "Cliente A" },
  "timeline": [                           // Últimos 20 eventos
    {
      "id": "evt_...",
      "timestamp": "2026-07-21T14:30:00Z",
      "service": "whatsapp",
      "status": "suspected",
      "rawSignal": { "lastActivity": "2026-07-21T14:20:00Z", "webhookReceived": true },
      "confidenceScore": 72,
      "signalBreakdown": {
        "activeNumbers": 0.9,
        "lastMessageSent": 0.5,
        "lastMessageReceived": 0.5,
        "webhookActivity": 0.6,
        "silencePenalty": -0.3
      }
    }
  ]
}
```

### 5.3 Webhook Receiver

```
POST /api/guardian/webhook
Headers: X-GHL-Signature: {hmac_sha256}
Body:
{
  "event": "workflow.started" | "workflow.completed" | "lc_email.stats",
  "subaccountId": "loc_abc123",
  "timestamp": "2026-07-21T14:30:00Z",
  "payload": { ... }   // datos específicos del evento
}
Response 200: { "received": true, "eventId": "evt_..." }
Response 401: Invalid signature
Response 429: Rate limited
```

*Nota: estos nombres de evento (`workflow.started`, etc.) son un esquema propio, definido en la acción de webhook que se configura manualmente dentro de cada workflow — GHL no expone un catálogo nativo de eventos de workflow, es el payload que nosotros mismos diseñamos al instrumentar cada flujo (ver riesgo operativo en sección 9).*

### 5.4 Configuración

```
GET /api/guardian/config
Response 200:
{
  "subaccounts": [...],
  "defaults": {
    "whatsappSuspectedWindow": 300,   // 5 minutos
    "whatsappConfirmedWindow": 900,   // 15 minutos
    "workflowDefaultTimeout": 3600,   // 1 hora (override por workflow)
    "phoneSilenceBaseline": 86400     // 24 horas (override por cliente)
  }
}

PUT /api/guardian/config
Body: { ... }   // Actualiza timeouts/baselines
Response 200: { "updated": true }
```

---

## 6. Modelo de Datos (Event Store + Config)

### 6.1 health_events (append-only)

```sql
CREATE TABLE health_events (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  subaccount_id TEXT NOT NULL,
  service     TEXT NOT NULL CHECK (service IN ('workflow','oauth','whatsapp','email','phone')),
  status      TEXT NOT NULL CHECK (status IN ('healthy','suspected','confirmed_offline','possibly_stuck','unknown')),
  raw_signal  JSONB NOT NULL DEFAULT '{}',
  confidence_score REAL,
  signal_breakdown JSONB
);

-- Índices para queries del dashboard
CREATE INDEX idx_events_subaccount_service_time
  ON health_events (subaccount_id, service, created_at DESC);

CREATE INDEX idx_events_created_at ON health_events (created_at DESC);

-- Política de retención (a definir antes de producción)
-- DELETE FROM health_events WHERE created_at < NOW() - INTERVAL '90 days';
```

### 6.2 clients (NUEVO)

```sql
CREATE TABLE clients (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name        TEXT NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  active      BOOLEAN NOT NULL DEFAULT true
);
```

*Cada `client` es el negocio real que hace el reclamo; puede tener una o varias `subaccounts` (locations de GHL). El dashboard rollup (sección 5.1) agrupa por esta tabla, no por subcuenta directamente.*

### 6.3 subaccounts (actualizada con client_id)

```sql
CREATE TABLE subaccounts (
  id          TEXT PRIMARY KEY,     -- GHL location ID
  client_id   UUID NOT NULL REFERENCES clients(id),   -- NUEVO
  name        TEXT NOT NULL,
  location_id TEXT NOT NULL UNIQUE,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  active      BOOLEAN NOT NULL DEFAULT true
);

CREATE INDEX idx_subaccounts_client ON subaccounts (client_id);
```

### 6.4 monitor_configs

```sql
CREATE TABLE monitor_configs (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  subaccount_id   TEXT NOT NULL REFERENCES subaccounts(id),
  service         TEXT NOT NULL,
  workflow_timeout_seconds INTEGER,   -- solo para workflow, por workflow específico
  suspected_after_seconds INTEGER,    -- ventana healthy → suspected
  confirmed_after_seconds INTEGER,    -- ventana suspected → confirmed
  activity_baseline INTEGER,          -- para Phone: llamadas/día esperadas
  UNIQUE (subaccount_id, service)
);
```

### 6.5 current_health (vista materializada para el dashboard)

```sql
CREATE MATERIALIZED VIEW current_health AS
SELECT DISTINCT ON (subaccount_id, service)
  subaccount_id,
  service,
  status,
  confidence_score,
  created_at as last_check
FROM health_events
ORDER BY subaccount_id, service, created_at DESC;

-- La vista se refresca desde run-all-checks.ts al final de cada ciclo de cron.
-- NO usar trigger AFTER INSERT — ver tasks.md Task 0.3.
CREATE UNIQUE INDEX idx_current_health ON current_health (subaccount_id, service);
```

### 6.6 agency_config (NUEVO — metadata, no el secreto en sí)

```sql
CREATE TABLE agency_config (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  company_id      TEXT NOT NULL,          -- Company ID de GHL, usado en /locations/search
  pit_created_at  TIMESTAMPTZ NOT NULL,   -- para saber cuándo rotar el PIT
  scopes_granted  TEXT[] NOT NULL,        -- para auditar qué permisos tiene el token activo
  last_verified_at TIMESTAMPTZ            -- última vez que el PIT respondió OK
);
```

*El valor del PIT **no** se guarda acá — vive únicamente en variable de entorno / secrets manager (ver sección 8). Esta tabla es solo metadata operativa para saber qué token está activo y cuándo rotarlo.*

---

## 7. Decisiones de Arquitectura (ADR)

### ADR-001: Event Store Append-Only sobre State Table

**Contexto**: El brief especifica guardar cada observación con timestamp, no solo el estado actual.

**Decisión**: Usar `health_events` como tabla append-only + vista materializada `current_health` para el dashboard.

**Consecuencias**:
- ✅ Permite construir timelines, métricas de uptime, MTTR a futuro
- ✅ El Confidence Score se calcula sobre datos históricos, no sobre un solo snapshot
- ✅ Bajo costo al inicio, aunque la estimación exacta depende del número real de subcuentas (ver nota en sección 9)
- ⚠️ Requiere política de retención antes de producción (90 días razonable para MVP)
- ⚠️ Vista materializada necesita refresh periódico

### ADR-002: Estrategia por Proveedor para OAuth (no monitor genérico)

**Contexto**: El brief confirma que OAuth no puede ser un solo monitor — Google Calendar devuelve 200 con `items: []` cuando el token expiró.

**Decisión**: Implementar `OAuthStrategy` como interfaz con implementaciones por proveedor. Cada integración nueva agrega un módulo, no reutiliza el existente.

**Consecuencias**:
- ✅ Encaja con SOLID (Open/Closed Principle): abierto para extensión, cerrado para modificación
- ✅ Cada estrategia conoce las respuestas "sospechosas" de su proveedor
- ⚠️ Costo de mantenimiento crece con cada proveedor nuevo (pero es explícito y acotado)

### ADR-003: Confidence Score como Capa de Presentación (MVP)

**Contexto**: El brief especifica que el score debe ser "explicable" y calibrarse con datos reales.

**Decisión**: El motor de detección trabaja con estados discretos (Healthy/Suspected/Confirmed). El Confidence Score se calcula como capa de presentación sobre esos estados + señales.

**Consecuencias**:
- ✅ El MVP no depende de calibrar una fórmula de scoring sin datos históricos
- ✅ Los pesos de cada señal son configurables (no hardcodeados)
- ✅ El score se afina iterativamente con datos reales del Event Store
- ✅ El breakdown de señales hace el score "explicable" para el equipo

### ADR-004: Rate Limiter con Token Bucket (corregido en esta revisión)

**Contexto**: GHL API tiene rate limit de 100 requests / 10 segundos, **por app, por location** (confirmado — no es un límite agregado entre locations).

**Decisión**: Implementar token bucket **por location**. Cada health check consume 1 token del bucket de su propia location.

**Consecuencias**:
- ✅ Con ~3 monitores de polling por subcuenta (OAuth, WhatsApp, Phone), cada location consume solo 3 de sus 100 tokens disponibles por ventana de 10s — muy lejos del límite, incluso sin espaciar entre locations distintas.
- ⚠️ **Corrección respecto a la versión anterior de este ADR**: el cálculo previo ("240 reqs por ciclo, necesitamos espaciar ~30 segundos") trataba el límite como si fuera global entre las 80+ subcuentas, lo cual no es así — cada location tiene su propio balde de 100 tokens/10s. No hace falta espaciar el ciclo completo por este motivo.
- ✅ Igual mantenemos `p-limit` + backoff exponencial como margen de seguridad ante reintentos o picos inesperados, no como requisito de rate limiting per se.
- ⚠️ Si el número real de subcuentas resulta ser muy alto (100s), el cuello de botella real no es el rate limit por location, sino la **duración total de ejecución del cron job** iterando todas las locations en serie — a validar según el número real (sección 9).

### ADR-005: Supabase sobre Prisma

**Contexto**: El brief propone Prisma para independencia de DB. El template wacrm usa Supabase JS Client.

**Decisión**: Usar Supabase JS Client directo. No agregar Prisma como capa extra.

**Consecuencias**:
- ✅ Menos dependencias, queries más simples para MVP
- ✅ RLS de Supabase funciona nativamente con el cliente
- ✅ Realtime subscriptions para el dashboard sin infra extra
- ⚠️ Migración a PostgreSQL standalone requeriría cambiar queries (pero Postgres es Postgres — el SQL es portable)

### ADR-006: Autenticación de Agencia — PIT + Location Token Exchange (NUEVO)

**Contexto**: El requisito de negocio es login único, acceso a todas las subcuentas. Se evaluaron tres alternativas: (a) OAuth por subcuenta, (b) conexión MCP, (c) PIT de agencia + intercambio por location token.

**Decisión**: Usar un **Private Integration Token de Agencia** (nivel Company, no subcuenta), disponible porque la agencia está en el plan Agency Pro (US$497 — confirmado que planes inferiores solo dan Location API Keys). Para operar sobre una subcuenta puntual, se intercambia ese PIT por un location token vía el endpoint oficial de HighLevel, cacheado en memoria con TTL corto.

**Por qué se descartó MCP**: tanto el servidor MCP oficial de HighLevel como los comunitarios están scoped a una sola location por conexión — no existe una conexión MCP a nivel agencia. Con 150-400+ subcuentas esto hubiera significado gestionar cientos de conexiones, lo opuesto al login único buscado.

**Consecuencias**:
- ✅ Login único real: el operador humano gestiona un solo PIT, no N credenciales.
- ✅ El PIT es estático — no expira ni necesita lógica de refresh automático (a diferencia de un token OAuth `authorization_code`, que expira cada 24h).
- ⚠️ Los location tokens obtenidos por intercambio sí tienen expiración corta — deben cachearse con TTL y no persistirse en la base de datos.
- ⚠️ Reporte de la comunidad (no confirmado si sigue vigente) de dificultad para encontrar el scope `oauth.write` al configurar el PIT para este intercambio — vigilar si aparece un 401 específico en este paso durante la implementación.

### ADR-007: Modelo Client → Subaccount (NUEVO)

**Contexto**: Se asumía 1 subcuenta = 1 cliente (80 subcuentas = 80 clientes). En la realidad, la agencia tiene ~80 clientes, cada uno con 1 a 10+ subcuentas — el número real de subcuentas es sustancialmente mayor a 80 y aún no está confirmado.

**Decisión**: Agregar una entidad `Client` con relación 1:N hacia `Subaccount`. El dashboard agrupa y hace rollup de incidentes por `Client`, ya que es quien efectivamente hace el reclamo — un cliente con 10 subcuentas y solo una con WhatsApp caído sigue siendo "ese cliente tiene un incidente" a nivel de negocio.

**Consecuencias**:
- ✅ El dashboard responde la pregunta real de negocio ("¿qué cliente tiene un problema?"), no solo la técnica ("¿qué location tiene un problema?").
- ⚠️ Todas las estimaciones de capacidad de la versión anterior de este documento (Event Store, rate limiting) asumían 80 subcuentas — deben recalcularse una vez confirmado el número real (ver sección 9).

### ADR-008: Estrategia de Scheduling — Vercel Cron vs. Externo (NUEVO, decisión pendiente)

**Contexto**: Se confirmó que **Vercel Cron Jobs en plan Hobby está limitado a 1 ejecución por día** — incompatible con la cadencia de 1-5 minutos que asume el resto del diseño (ventanas de Suspected/Confirmed, timeouts de workflow).

**Opciones**:
1. **Upgrade a Vercel Pro** (US$20/mes/usuario) — mantiene todo el código igual, cron nativo cada minuto.
2. **Scheduler externo** (GitHub Actions con cron schedule, o servicio tipo cron-job.org) que llama por HTTP a `GET /api/guardian/cron` — sin costo adicional de Vercel, pero agrega una dependencia externa y un secreto compartido (`CRON_SECRET`) para proteger el endpoint.

**Decisión**: pendiente — a definir antes de Fase 7 (Cron Jobs + Deploy). No bloquea el desarrollo de las Fases 1-6, que no dependen del mecanismo de disparo.

**Consecuencias**: si se elige la opción externa, el endpoint `/api/guardian/cron` debe diseñarse igual (idempotente, protegido por secreto) independientemente de quién lo dispare — esto ya está contemplado en la sección 8.

---

## 8. Infraestructura y Deploy

### Entornos

| Entorno | Rama | URL | DB |
|---|---|---|---|
| **dev** | `develop` | `guardian-dev.vercel.app` | Supabase dev project |
| **prod** | `main` | `guardian.wacrm.tech` (o dominio propio) | Supabase prod project |

### CI/CD

- GitHub Actions: typecheck + lint + test en cada PR
- Vercel: deploy automático en merge a `main`
- Disparo de health checks: **pendiente de decisión** — Vercel Cron Jobs (requiere plan Pro) o GitHub Actions scheduled workflow llamando al endpoint por HTTP (ver ADR-008)

### Variables de Entorno

```
# Supabase
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=

# GHL Agency Auth (actualizado — reemplaza GHL_API_KEY genérico)
GHL_AGENCY_PIT=            # Private Integration Token de agencia (Settings > Private Integrations, nivel Company)
GHL_COMPANY_ID=            # ID de la agencia, usado en /locations/search y en el intercambio agencia→location
GHL_API_BASE_URL=https://services.leadconnectorhq.com   # CORREGIDO — no rest.gohighlevel.com/v2/

# Webhook
GHL_WEBHOOK_SECRET=    # HMAC shared secret

# Cron
CRON_SECRET=           # Para proteger GET /api/guardian/cron, sea Vercel Cron o scheduler externo
```

---

## 9. Riesgos Técnicos

| Riesgo | Probabilidad | Impacto | Mitigación |
|---|---|---|---|
| Rate limits de GHL bloquean polling | Baja *(corregido — el límite es por location, no global; ver ADR-004)* | Bajo | Token bucket por location + `p-limit`/backoff como margen de seguridad |
| Workflows sin webhook final nunca se detectan como colgados | Media | Alto | Timeout configurable por workflow; si no se configura, no se monitorea ese workflow |
| WhatsApp reporta healthy pero está desconectado | Alta | Bajo | Estados intermedios (Suspected → Confirmed) + comunicación interna de latencia |
| Falsos positivos de OAuth (token OK pero respuesta "sospechosa") | Media | Medio | Estrategia por proveedor bien calibrada; whitelist de respuestas normales |
| Escalado del Event Store | Baja | Medio | Política de retención a 90 días; recalcular tamaño real una vez confirmado el número de subcuentas |
| Instrumentación manual de workflows bloquea adopción | Alta | Alto | Documentar como tarea explícita de setup; template de webhook pre-configurado para copiar/pegar |
| **Vercel Cron Hobby limita a 1 ejecución/día** *(NUEVO)* | Alta (si no se decide) | Alto | Decidir entre Vercel Pro y scheduler externo antes de Fase 7 — ver ADR-008 |
| **Número real de subcuentas desconocido** *(NUEVO)* | Alta | Medio | Correr `GET /locations/search` con el PIT de agencia antes de finalizar cualquier estimación de capacidad — 80 es el número de *clientes*, no de subcuentas |
| **Falla en el intercambio agencia→location token** *(NUEVO)* | Media | Alto | Ver CP-005; puede dejar subcuentas puntuales sin monitoreo aunque el resto del sistema funcione |

---

## 10. Critical Points (para monitoreo operativo futuro)

```yaml
client: ghl-guardian-internal
critical_points:
  - id: CP-001
    name: health_check_cron
    severity: P1
    description: El cron job de health checks no se ejecuta
    impact: Sin datos frescos en el dashboard. Ventana ciega de monitoreo.
    playbook:
      pattern: "cron job failed|timeout.*cron|500.*cron"
      auto_fix: |
        1. Verificar Vercel Cron logs (o logs del scheduler externo, según ADR-008)
        2. Si rate limited por GHL → esperar y reintentar
        3. Notificar a Leo por Telegram

  - id: CP-002
    name: webhook_not_receiving
    severity: P1
    description: Los webhooks de GHL no están llegando al endpoint
    impact: Workflows y Email no reciben eventos. Solo dependemos de polling.
    playbook:
      pattern: "no webhook events in last 10 minutes"
      auto_fix: null  # Verificar configuración de webhooks en GHL

  - id: CP-003
    name: dashboard_stale_data
    severity: P2
    description: El dashboard muestra datos con más de 10 minutos de antigüedad
    impact: Operadores toman decisiones con información desactualizada
    playbook:
      pattern: "last_check > 10 minutes ago"
      auto_fix: null

  - id: CP-004
    name: event_store_growth
    severity: P3
    description: Event Store creciendo sin política de retención aplicada
    impact: Costo de storage aumentando linealmente
    playbook:
      pattern: "table size > 1GB"
      auto_fix: |
        1. Aplicar política de retención (DELETE < 90 días)
        2. Verificar que la vista materializada se refresca correctamente

  - id: CP-005
    name: location_token_exchange_failure
    severity: P1
    description: Falla el intercambio del PIT de agencia por un location token para una o más subcuentas
    impact: Esas subcuentas puntuales quedan sin monitoreo, aunque el resto del sistema siga funcionando con normalidad
    playbook:
      pattern: "401|403.*location.*token|token exchange failed"
      auto_fix: |
        1. Verificar que el PIT de agencia no fue revocado o rotado (ver agency_config.last_verified_at)
        2. Verificar scopes del PIT (oauth.readonly/oauth.write pueden faltar — caso reportado por otros usuarios de la comunidad)
        3. Reintentar el intercambio; si persiste, marcar la subcuenta como "unknown" (no "confirmed_offline") y notificar a Leo
```

---

## 11. Tests Esperados (TDD de Pipeline)

### Contrato: GET /api/guardian/health

| # | Caso | Setup | HTTP Esperado | Body Esperado |
|---|---|---|---|---|
| T01 | Dashboard con datos | 3 subcuentas con eventos recientes | 200 | summary.totalSubaccounts=3 |
| T02 | Sin subcuentas | DB vacía | 200 | { clients: [], summary: { totalSubaccounts: 0 } } |
| T03 | Subcuenta sin eventos | 1 subcuenta, 0 health_events | 200 | services con status "unknown", score null |
| T14 | Rollup por cliente *(NUEVO)* | Cliente con 3 subcuentas, 1 con incidente | 200 | `clients[x].hasIncident = true`, aunque 2/3 subcuentas estén healthy |

### Contrato: POST /api/guardian/webhook

| # | Caso | Input | HTTP Esperado | Body Esperado |
|---|---|---|---|---|
| T04 | Webhook workflow.started válido | HMAC correcto, payload workflow.started | 200 | { received: true, eventId: "..." } |
| T05 | Firma inválida | HMAC incorrecto | 401 | { error: "invalid_signature" } |
| T06 | Evento desconocido | event: "unknown.event" | 400 | { error: "unknown_event_type" } |
| T07 | Rate limit por location | >100 req en 10s para misma location | 429 | { error: "rate_limited" } |
| — | *(Nota: el límite real de GHL es por location, no por IP. Rate-limitar el webhook receiver por subaccountId tiene sentido de negocio: evita que una sola subcuenta ruidosa sature el ingestion.)* |

### Health Engine: WorkflowMonitor

| # | Caso | Setup | Resultado Esperado |
|---|---|---|---|
| T08 | Workflow completado a tiempo | Webhook start + end en < timeout | status: healthy |
| T09 | Workflow possibly stuck | Webhook start, sin end, > timeout | status: possibly_stuck |
| T10 | Sin webhooks | Sin eventos para este workflow | status: unknown |

### Health Engine: WhatsAppMonitor

| # | Caso | Setup | Resultado Esperado |
|---|---|---|---|
| T11 | WhatsApp healthy | Actividad reciente (< suspectedWindow) | status: healthy |
| T12 | WhatsApp suspected | Sin actividad > suspectedWindow, < confirmedWindow | status: suspected |
| T13 | WhatsApp confirmed offline | Sin actividad > confirmedWindow | status: confirmed_offline |

### Agency Token Manager *(NUEVO)*

| # | Caso | Setup | Resultado Esperado |
|---|---|---|---|
| T15 | Intercambio exitoso | PIT válido, locationId existente | Devuelve location token válido, cacheado con TTL |
| T16 | Falla de intercambio | PIT revocado o locationId inexistente | Subcuenta marcada como "unknown" con motivo "auth_error" — nunca como "confirmed_offline" |
| T17 | Cache de location token | Dos checks seguidos a la misma location dentro del TTL | Solo un intercambio real contra GHL; el segundo usa el token cacheado |

### End-to-End

| # | Flujo | Pasos | Resultado esperado |
|---|---|---|---|
| E2E-01 | Ciclo completo de health check | Cron/scheduler → intercambio de token → poll GHL → persistir eventos → dashboard refresca | Dashboard muestra estados actualizados, agrupados por cliente |
| E2E-02 | Degradación detectada | WhatsApp pasa healthy→suspected→confirmed | Timeline muestra la progresión, score baja gradualmente |
| E2E-03 | Recuperación | Servicio vuelve a healthy | Último evento es healthy, score sube |

---

## 12. Parámetros Configurables (NO hardcodeados)

Estos valores viven en `monitor_configs` y se ajustan con datos reales:

| Parámetro | Default | Descripción |
|---|---|---|
| `workflow_timeout_seconds` | 3600 (1h) | Por workflow específico. Override manual al instrumentar. |
| `suspected_after_seconds` | 300 (5min) | Ventana healthy→suspected para WhatsApp |
| `confirmed_after_seconds` | 900 (15min) | Ventana suspected→confirmed para WhatsApp |
| `activity_baseline` | null (calcular) | Llamadas/día esperadas para Phone. Se calcula con datos reales. |
| `score_weights` | { activity: 0.4, recency: 0.3, webhook: 0.2, silence: 0.1 } | Pesos del Confidence Score |

*Nota: la cadencia del cron (1-5 min) también debería ser configurable, ya que su valor real depende de la decisión pendiente en ADR-008 (Vercel Pro permite granularidad de minuto; un scheduler externo tipo GitHub Actions tiene su propio piso de frecuencia a verificar).*

---

## 13. Próximos Pasos

0. **(NUEVO, primero) Confirmar el número real de subcuentas**: correr `GET /locations/search` con el PIT de agencia y poblar las tablas `clients` + `subaccounts` con datos reales, antes de recalcular cualquier estimación de capacidad (Event Store, duración de cron, etc.)
1. **Crear repo** con Next.js + shadcn/ui inicializado (extraer de wacrm)
2. **Definir tasks.md** con backlog de implementación (Plan)
3. **Implementar en orden**:
   - Fase 1: Domain entities (incluyendo `Client`) + Event Store + Supabase setup
   - Fase 2: Agency Token Manager + GHL REST Client + Rate Limiter
   - Fase 3: WorkflowMonitor + Webhook Receiver
   - Fase 4: WhatsAppMonitor + OAuthMonitor
   - Fase 5: Dashboard UI con datos reales, agrupado por cliente
   - Fase 6: Confidence Score + Configuración
   - Fase 7: Decidir Vercel Pro vs. scheduler externo (ADR-008), Cron Jobs + Deploy a Vercel
