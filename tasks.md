# GHL Guardian — Plan de Implementación (MVP)

> **Para Hermes:** Usar `subagent-driven-development` para implementar este plan task por task.
> **Arquitectura de referencia:** [`architecture.md`](./architecture.md) (v2)

**Goal:** Construir una plataforma de observabilidad para GoHighLevel capaz de responder: ¿cuántas subcuentas están sanas?, ¿cuáles tienen incidentes?, ¿qué servicio está afectado?, ¿desde cuándo?

**Arquitectura:** Clean Architecture en Next.js 16 full-stack. Monorepo: `src/domain/` → `use-cases/` → `adapters/` → `ui/`. Supabase como Event Store. Auth de agencia vía PIT + location token exchange.

**Tech Stack:** Next.js 16 + React 19 + TypeScript 6 · Supabase (Postgres + Auth) · Tailwind 4 + shadcn/ui + Recharts · Vitest

---

## Fase 0: Setup del Proyecto + Datos Reales

### Task 0.1: Inicializar Next.js con dependencias

**Objective:** Crear el esqueleto del proyecto con las dependencias del stack.

**Files:**
- Crear: scaffold completo vía `create-next-app`

**Steps:**

```bash
# Desde /home/leonardo/projects
npx create-next-app@latest ghl-guardian --typescript --tailwind --eslint --app --src-dir --import-alias "@/*"
cd ghl-guardian
npm install @supabase/ssr @supabase/supabase-js class-variance-authority clsx date-fns lucide-react recharts sonner tailwind-merge tw-animate-css
npm install -D vitest @types/node prettier prettier-plugin-tailwindcss
```

**Verification:** `npm run dev` arranca en localhost:3000. `npx tsc --noEmit` limpio.

---

### Task 0.2: Inicializar shadcn/ui

**Objective:** Agregar los componentes base de shadcn/ui que usaremos en el dashboard.

```bash
npx shadcn@latest init
npx shadcn@latest add card badge separator tooltip select tabs
```

**Verification:** `src/components/ui/` contiene los componentes. Build limpio.

---

### Task 0.3: Setup Supabase + Ejecutar Migrations

**Objective:** Crear proyecto Supabase, ejecutar schema SQL, configurar variables de entorno.

**Files:**
- Crear: `supabase/migrations/00001_schema.sql`
- Modificar: `.env.local`

**Schema SQL** (de architecture.md §6):

```sql
-- Clients
CREATE TABLE clients (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  active BOOLEAN NOT NULL DEFAULT true
);

-- Subaccounts
CREATE TABLE subaccounts (
  id TEXT PRIMARY KEY,
  client_id UUID NOT NULL REFERENCES clients(id),
  name TEXT NOT NULL,
  location_id TEXT NOT NULL UNIQUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  active BOOLEAN NOT NULL DEFAULT true
);
CREATE INDEX idx_subaccounts_client ON subaccounts (client_id);

-- Health Events (append-only)
CREATE TABLE health_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  subaccount_id TEXT NOT NULL,
  service TEXT NOT NULL CHECK (service IN ('workflow','oauth','whatsapp','email','phone')),
  status TEXT NOT NULL CHECK (status IN ('healthy','suspected','confirmed_offline','possibly_stuck','unknown')),
  raw_signal JSONB NOT NULL DEFAULT '{}',
  confidence_score REAL,
  signal_breakdown JSONB
);
CREATE INDEX idx_events_subaccount_service_time ON health_events (subaccount_id, service, created_at DESC);
CREATE INDEX idx_events_created_at ON health_events (created_at DESC);

-- Monitor Configs
CREATE TABLE monitor_configs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  subaccount_id TEXT NOT NULL REFERENCES subaccounts(id),
  service TEXT NOT NULL,
  workflow_timeout_seconds INTEGER,
  suspected_after_seconds INTEGER,
  confirmed_after_seconds INTEGER,
  activity_baseline INTEGER,
  UNIQUE (subaccount_id, service)
);

-- Agency Config (metadata, no secrets)
CREATE TABLE agency_config (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  company_id TEXT NOT NULL,
  pit_created_at TIMESTAMPTZ NOT NULL,
  scopes_granted TEXT[] NOT NULL,
  last_verified_at TIMESTAMPTZ
);

-- Materialized View for dashboard
CREATE MATERIALIZED VIEW current_health AS
SELECT DISTINCT ON (subaccount_id, service)
  subaccount_id, service, status, confidence_score, created_at as last_check
FROM health_events
ORDER BY subaccount_id, service, created_at DESC;
CREATE UNIQUE INDEX idx_current_health ON current_health (subaccount_id, service);

-- La vista se refresca desde run-all-checks.ts al final de cada ciclo de cron,
-- NO desde un trigger por fila (un REFRESH MATERIALIZED VIEW CONCURRENTLY en un trigger
-- AFTER INSERT bloquearía cada escritura del Event Store).
CREATE OR REPLACE FUNCTION refresh_current_health()
RETURNS void AS $$
BEGIN
  REFRESH MATERIALIZED VIEW CONCURRENTLY current_health;
END;
$$ LANGUAGE plpgsql;
```

**Verification:** Tablas creadas en Supabase Dashboard. `supabaseClient.from('clients').select('*')` devuelve array vacío.

---

### Task 0.4: Obtener datos reales de subcuentas (Step 0 del architecture.md)

**Objective:** Usar el PIT de agencia para llamar a `GET /locations/search` y poblar las tablas `clients` + `subaccounts` con datos reales.

**Files:**
- Crear: `scripts/seed-subaccounts.ts`

**Steps:**
1. Leer `GHL_AGENCY_PIT` y `GHL_COMPANY_ID` de `.env.local`
2. Llamar a `GET https://services.leadconnectorhq.com/locations/search?companyId={COMPANY_ID}&limit=100` con header `Authorization: Bearer {PIT}`. **Manejar paginación:** si la respuesta incluye `next` o `offset`, iterar hasta consumir todas las páginas. Con 300+ subcuentas, el límite default puede no devolver todos los resultados.
3. Parsear respuesta. Agrupar por cliente usando heurística de nombre. **Locations sin match claro se reportan como "sin asignar" para revisión manual** — no se asigna cliente automáticamente si hay ambigüedad.
4. Insertar en `clients` + `subaccounts`
5. Imprimir resumen: N clientes, N subcuentas

**Verification:** `SELECT count(*) FROM subaccounts` devuelve el número real (no 80 hardcodeado).

---

### Task 0.5: Commit inicial del scaffold

```bash
git add -A
git commit -m "scaffold: Next.js 16 + Supabase schema + shadcn/ui + datos reales de subcuentas"
git push
```

---

## Fase 1: Capa de Dominio

### Task 1.1: Value Objects y Enums

**Objective:** Definir los tipos inmutables del dominio.

**Files:**
- Crear: `src/domain/value-objects/health-status.ts`
- Crear: `src/domain/value-objects/service-type.ts`
- Crear: `src/domain/value-objects/confidence-score.ts`

**Step 1: Escribir tests**

```typescript
// src/domain/value-objects/health-status.test.ts
import { describe, it, expect } from 'vitest';
import { HealthStatus, isValidTransition } from './health-status';

describe('HealthStatus', () => {
  it('tiene los 5 estados definidos', () => {
    expect(HealthStatus.Healthy).toBe('healthy');
    expect(HealthStatus.Suspected).toBe('suspected');
    expect(HealthStatus.ConfirmedOffline).toBe('confirmed_offline');
    expect(HealthStatus.PossiblyStuck).toBe('possibly_stuck');
    expect(HealthStatus.Unknown).toBe('unknown');
  });

  it('permite transición healthy → suspected', () => {
    expect(isValidTransition(HealthStatus.Healthy, HealthStatus.Suspected)).toBe(true);
  });

  it('permite transición suspected → confirmed_offline', () => {
    expect(isValidTransition(HealthStatus.Suspected, HealthStatus.ConfirmedOffline)).toBe(true);
  });

  it('permite recuperación: suspected → healthy', () => {
    expect(isValidTransition(HealthStatus.Suspected, HealthStatus.Healthy)).toBe(true);
  });

  it('no permite transición healthy → confirmed_offline (saltear suspected)', () => {
    expect(isValidTransition(HealthStatus.Healthy, HealthStatus.ConfirmedOffline)).toBe(false);
  });
});
```

**Step 2: Implementar**

```typescript
// src/domain/value-objects/health-status.ts
export const HealthStatus = {
  Healthy: 'healthy',
  Suspected: 'suspected',
  ConfirmedOffline: 'confirmed_offline',
  PossiblyStuck: 'possibly_stuck',
  Unknown: 'unknown',
} as const;

export type HealthStatus = (typeof HealthStatus)[keyof typeof HealthStatus];

const VALID_TRANSITIONS: Record<HealthStatus, HealthStatus[]> = {
  [HealthStatus.Healthy]: [HealthStatus.Suspected, HealthStatus.Unknown],
  [HealthStatus.Suspected]: [HealthStatus.Healthy, HealthStatus.ConfirmedOffline, HealthStatus.Unknown],
  [HealthStatus.ConfirmedOffline]: [HealthStatus.Healthy, HealthStatus.Unknown],
  [HealthStatus.PossiblyStuck]: [HealthStatus.Healthy, HealthStatus.Unknown],
  [HealthStatus.Unknown]: [HealthStatus.Healthy, HealthStatus.Suspected],
};

export function isValidTransition(from: HealthStatus, to: HealthStatus): boolean {
  return VALID_TRANSITIONS[from]?.includes(to) ?? false;
}
```

**Step 3: Verificar tests pasan**

```bash
npx vitest run src/domain/value-objects/health-status.test.ts
```

---

### Task 1.2: Entidades del Dominio

**Objective:** Definir Client, Subaccount, MonitorConfig, HealthEvent como tipos.

**Files:**
- Crear: `src/domain/entities/client.ts`
- Crear: `src/domain/entities/subaccount.ts`
- Crear: `src/domain/entities/health-event.ts`
- Crear: `src/domain/entities/monitor-config.ts`

**(Sin tests unitarios — son tipos planos, la validación vive en use-cases y adapters)**

```typescript
// src/domain/entities/client.ts
export interface Client {
  readonly id: string;
  readonly name: string;
  readonly createdAt: Date;
  readonly active: boolean;
}

// src/domain/entities/subaccount.ts
export interface Subaccount {
  readonly id: string;       // GHL location ID
  readonly clientId: string;
  readonly name: string;
  readonly locationId: string;
  readonly createdAt: Date;
  readonly active: boolean;
}

// src/domain/entities/health-event.ts
import type { HealthStatus } from '../value-objects/health-status';
import type { ServiceType } from '../value-objects/service-type';

export interface HealthEvent {
  readonly id: string;
  readonly timestamp: Date;
  readonly subaccountId: string;
  readonly service: ServiceType;
  readonly status: HealthStatus;
  readonly rawSignal: Record<string, unknown>;
  readonly confidenceScore: number | null;
  readonly signalBreakdown: Record<string, number> | null;
}

// src/domain/entities/monitor-config.ts
import type { ServiceType } from '../value-objects/service-type';

export interface MonitorConfig {
  readonly id: string;
  readonly subaccountId: string;
  readonly service: ServiceType;
  readonly workflowTimeoutSeconds?: number;
  readonly suspectedAfterSeconds?: number;
  readonly confirmedAfterSeconds?: number;
  readonly activityBaseline?: number;
}
```

**Step: Commit**

```bash
git add src/domain/
git commit -m "feat(domain): value objects + entities (HealthStatus, Client, Subaccount, HealthEvent, MonitorConfig)"
```

---

### Task 1.3: Puertos (Interfaces)

**Objective:** Definir las interfaces que el dominio expone y los adapters implementan.

**Files:**
- Crear: `src/domain/ports/health-check.ts`
- Crear: `src/domain/ports/event-store.ts`
- Crear: `src/domain/ports/ghl-client.ts`
- Crear: `src/domain/ports/agency-token-provider.ts`

```typescript
// src/domain/ports/ghl-client.ts
import type { HealthStatus } from '../value-objects/health-status';

export interface WorkflowInfo {
  id: string;
  name: string;
  status: 'active' | 'paused' | 'draft';
  lastExecutionAt?: Date;
}

export interface OAuthIntegration {
  id: string;
  provider: string;
  status: 'connected' | 'disconnected' | 'expired';
  lastSyncAt?: Date;
}

export interface WhatsAppNumber {
  id: string;
  phoneNumber: string;
  status: HealthStatus;
  lastActivityAt?: Date;
}

export interface GHLClient {
  getWorkflows(locationId: string, token: string): Promise<WorkflowInfo[]>;
  getOAuthIntegrations(locationId: string, token: string): Promise<OAuthIntegration[]>;
  getWhatsAppNumbers(locationId: string, token: string): Promise<WhatsAppNumber[]>;
}
```

```typescript
// src/domain/ports/health-check.ts
import type { HealthStatus } from '../value-objects/health-status';
import type { ServiceType } from '../value-objects/service-type';
import type { MonitorConfig } from '../entities/monitor-config';

export interface HealthCheckResult {
  status: HealthStatus;
  rawSignal: Record<string, unknown>;
  metadata?: { latencyMs: number; provider?: string };
}

export interface HealthCheck {
  readonly serviceType: ServiceType;
  readonly confidence: number;
  run(subaccountId: string, config: MonitorConfig): Promise<HealthCheckResult>;
}

// src/domain/ports/event-store.ts
import type { HealthEvent } from '../entities/health-event';

export interface EventStore {
  append(event: Omit<HealthEvent, 'id'>): Promise<HealthEvent>;
  queryLatest(subaccountId: string, service: string): Promise<HealthEvent | null>;
  queryTimeline(subaccountId: string, limit?: number): Promise<HealthEvent[]>;
  queryCurrentHealth(): Promise<Map<string, Map<string, { status: string; score: number | null; lastCheck: Date }>>>;
}
```

**Commit:**

```bash
git add src/domain/ports/
git commit -m "feat(domain): puertos (HealthCheck, EventStore, GHLClient, AgencyTokenProvider)"
```

---

## Fase 2: Adapters — Auth + GHL REST Client

### Task 2.1: Supabase Browser Client

**Objective:** Extraer el patrón de singleton de wacrm.

**Files:**
- Crear: `src/lib/supabase/client.ts` (browser)
- Crear: `src/lib/supabase/server.ts`  (server-side, para API routes)

**Verification:** `npm run build` limpio.

---

### Task 2.2: Agency Token Manager (ADR-006)

**Objective:** Implementar el intercambio PIT → location token con cache en memoria.

**Files:**
- Crear: `src/adapters/ghl/agency-token-manager.ts`
- Crear: `src/adapters/ghl/agency-token-manager.test.ts`

**Step 1: Escribir test**

```typescript
// src/adapters/ghl/agency-token-manager.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { AgencyTokenManager } from './agency-token-manager';

describe('AgencyTokenManager', () => {
  const mockFetch = vi.fn();
  let manager: AgencyTokenManager;

  beforeEach(() => {
    vi.clearAllMocks();
    manager = new AgencyTokenManager('pit_test123', 'company_xyz', 'https://services.leadconnectorhq.com', mockFetch);
  });

  it('intercambia PIT por location token exitosamente', async () => {
    mockFetch.mockResolvedValueOnce({
      ok: true,
      json: () => Promise.resolve({ access_token: 'loc_token_abc', expires_in: 3600 }),
    });

    const result = await manager.getLocationToken('loc_123');
    expect(result.token).toBe('loc_token_abc');
    expect(result.expiresAt).toBeInstanceOf(Date);
  });

  it('cachea el token y no repite intercambio dentro del TTL', async () => {
    mockFetch.mockResolvedValueOnce({
      ok: true,
      json: () => Promise.resolve({ access_token: 'loc_token_abc', expires_in: 3600 }),
    });

    await manager.getLocationToken('loc_123');
    await manager.getLocationToken('loc_123');

    expect(mockFetch).toHaveBeenCalledTimes(1); // Segunda llamada usa cache
  });

  it('devuelve error si el PIT es inválido', async () => {
    mockFetch.mockResolvedValueOnce({ ok: false, status: 401 });

    await expect(manager.getLocationToken('loc_123')).rejects.toThrow('token_exchange_failed');
  });
});
```

**Step 2: Implementar**

```typescript
// src/adapters/ghl/agency-token-manager.ts
import type { AgencyTokenProvider } from '@/domain/ports/agency-token-provider';

interface CachedToken {
  token: string;
  expiresAt: Date;
}

export class AgencyTokenManager implements AgencyTokenProvider {
  private cache = new Map<string, CachedToken>();
  private fetchFn: typeof fetch;

  constructor(
    private readonly pit: string,
    private readonly companyId: string,
    private readonly baseUrl: string,
    fetchFn: typeof fetch = fetch,
  ) {
    this.fetchFn = fetchFn;
  }

  async getLocationToken(locationId: string): Promise<{ token: string; expiresAt: Date }> {
    const cached = this.cache.get(locationId);
    if (cached && cached.expiresAt > new Date()) {
      return cached;
    }

    const response = await this.fetchFn(
      `${this.baseUrl}/oauth/locationToken`,
      {
        method: 'POST',
        headers: {
          Authorization: `Bearer ${this.pit}`,
          'Content-Type': 'application/x-www-form-urlencoded',
          Version: '2021-07-28',
        },
        body: new URLSearchParams({ companyId: this.companyId, locationId }),
      },
    );

    if (!response.ok) {
      throw new Error(`token_exchange_failed: ${response.status}`);
    }

    const data = await response.json();
    const entry: CachedToken = {
      token: data.access_token,
      expiresAt: new Date(Date.now() + (data.expires_in ?? 3600) * 1000),
    };

    this.cache.set(locationId, entry);
    return entry;
  }

  async listLocations(): Promise<{ locationId: string; name: string }[]> {
    // NOTA: si la agencia tiene 300+ locations, este endpoint puede paginar.
    // Implementar iteración sobre páginas (next/offset) si la respuesta lo requiere.
    const response = await this.fetchFn(
      `${this.baseUrl}/locations/search?companyId=${this.companyId}`,
      { headers: { Authorization: `Bearer ${this.pit}` } },
    );
    const data = await response.json();
    return (data.locations ?? []).map((loc: any) => ({
      locationId: loc.id,
      name: loc.name,
    }));
  }
}
```

**Commit:**

```bash
git add src/adapters/ghl/agency-token-manager*
git commit -m "feat(adapters): AgencyTokenManager — PIT + location token exchange con cache"
```

> ⚠️ **Pre-flight check:** Antes de escribir el código, verificar que el PIT tenga el scope `oauth.write` en el dropdown de GHL (`Settings → Private Integrations`). Varios usuarios de la comunidad reportan que este scope no siempre aparece, y sin él el endpoint `/oauth/locationToken` devuelve 401 persistente aunque el código sea correcto.
>
> ⚠️ **Tabla `agency_config` — solo metadata:** La tabla `agency_config` (creada en Task 0.3) almacena `company_id`, `scopes_granted`, `last_verified_at` — SOLO metadata operativa para el dashboard (ej. "PIT verificado hace 3 días"). **El valor del PIT nunca se persiste en esta tabla ni en ninguna otra** — vive exclusivamente en `GHL_AGENCY_PIT` (variable de entorno). Task 2.2 debe leer `agency_config` para exponer `last_verified_at` y `scopes_granted` en el dashboard, pero nunca escribir el secreto.

---

### Task 2.3: Rate Limiter (Token Bucket por location)

**Files:**
- Crear: `src/adapters/ghl/rate-limiter.ts`
- Crear: `src/adapters/ghl/rate-limiter.test.ts`

```typescript
// src/adapters/ghl/rate-limiter.ts
export class RateLimiter {
  private buckets = new Map<string, { tokens: number; lastRefill: number }>();
  private readonly maxTokens: number;
  private readonly refillIntervalMs: number;

  constructor(maxTokens = 100, refillIntervalMs = 10_000) {
    this.maxTokens = maxTokens;
    this.refillIntervalMs = refillIntervalMs;
  }

  async acquire(locationId: string): Promise<void> {
    // NOTA: buckets crece sin límite por diseño. En MVP con ~400 subcuentas + serverless
    // (instancia nueva por invocación), la memoria no es problema. Si se migra a un
    // proceso long-running, agregar evicción LRU con TTL.
    let bucket = this.buckets.get(locationId);
    const now = Date.now();

    if (!bucket) {
      bucket = { tokens: this.maxTokens, lastRefill: now };
      this.buckets.set(locationId, bucket);
    }

    // Refill
    const elapsed = now - bucket.lastRefill;
    if (elapsed >= this.refillIntervalMs) {
      bucket.tokens = this.maxTokens;
      bucket.lastRefill = now;
    }

    if (bucket.tokens <= 0) {
      // Wait for next window
      await new Promise(resolve => setTimeout(resolve, this.refillIntervalMs - elapsed + 500));
      return this.acquire(locationId);
    }

    bucket.tokens--;
  }
}
```

**Test cases:** adquiere tokens, espera cuando bucket vacío, refill después de ventana.

---

### Task 2.4: GHL REST Client

**Objective:** Cliente HTTP tipado para GHL API v2, usando AgencyTokenManager + RateLimiter.

**Files:**
- Crear: `src/adapters/ghl/ghl-rest-client.ts`
- Crear: `src/adapters/ghl/ghl-rest-client.test.ts`

**Métodos iniciales:**
- `getWorkflows(locationId)` → lista workflows con estado
- `getOAuthIntegrations(locationId)` → integraciones OAuth activas
- `getWhatsAppNumbers(locationId)` → números WhatsApp + estado
- `getPhoneStats(locationId)` → estadísticas LC Phone

**Verification:** Tests con mock de fetch para cada endpoint, respetando rate limiter.

> ⚠️ **Fail-fast:** Al inicializarse, `AgencyTokenManager` debe validar que `GHL_AGENCY_PIT` esté definida y no vacía. Si falta, lanzar error explícito al inicio (no en runtime durante el primer health check).

---

## Fase 3: Monitores — Workflow + Webhook

### Task 3.1: WorkflowMonitor

**Objective:** Implementar HealthCheck para workflows usando webhooks start/end + timeout.

**Files:**
- Crear: `src/adapters/monitors/workflow-monitor.ts`
- Crear: `src/adapters/monitors/workflow-monitor.test.ts`

**Lógica:**
1. Query `health_events` por `subaccount_id + service='workflow'` en los últimos N minutos
2. Si hay webhook `completed` sin `stuck` → healthy
3. Si hay `started` sin `completed` y pasó `workflowTimeoutSeconds` → possibly_stuck
4. Si no hay eventos → unknown

**Test cases:** T08, T09, T10 de architecture.md §11.

---

### Task 3.2: Webhook Receiver

**Objective:** Endpoint que recibe webhooks de GHL, valida HMAC, persiste HealthEvent.

> ⚠️ **Deduplicación:** GHL puede enviar webhooks duplicados. Usar el `eventId` del payload como idempotency key — verificar si ya existe un `health_event` con ese `eventId` antes de insertar.

**Files:**
- Crear: `src/app/api/guardian/webhook/route.ts`
- Crear: `src/app/api/guardian/webhook/route.test.ts`
- Crear: `src/adapters/webhook/ghl-webhook-handler.ts`

**Test cases:** T04, T05, T06, T07 de architecture.md §11.

---

### Task 3.3: Supabase Event Store + Repos

**Objective:** Implementar EventStore + repositorios de dominio con Supabase JS Client.

**Files:**
- Crear: `src/adapters/supabase/event-store.ts`
- Crear: `src/adapters/supabase/event-store.test.ts`
- Crear: `src/adapters/supabase/client-repo.ts`
- Crear: `src/adapters/supabase/subaccount-repo.ts`
- Crear: `src/adapters/supabase/config-repo.ts`

---

## Fase 4: Monitores restantes

### Task 4.1: WhatsAppMonitor

**Files:**
- Crear: `src/adapters/monitors/whatsapp-monitor.ts`
- Crear: `src/adapters/monitors/whatsapp-monitor.test.ts`

**Test cases:** T11, T12, T13 de architecture.md §11.

### Task 4.2: OAuthMonitor + Estrategias

**Files:**
- Crear: `src/adapters/ghl/oauth-strategies/oauth-strategy.ts`
- Crear: `src/adapters/ghl/oauth-strategies/google-calendar.ts`
- Crear: `src/adapters/monitors/oauth-monitor.ts`
- Crear: `src/adapters/monitors/oauth-monitor.test.ts`

### Task 4.3: EmailMonitor + PhoneMonitor (stub)

**Files:**
- Crear: `src/adapters/monitors/email-monitor.ts`
- Crear: `src/adapters/monitors/phone-monitor.ts`

**Alcance MVP:** EmailMonitor se implementa completo (webhook LCEmailStats). PhoneMonitor es un **stub** que devuelve `HealthStatus.Unknown` — la heurística de silencio requiere baseline de datos reales que no existen en MVP. Se completa en v2.

---

## Fase 5: Use Cases

### Task 5.1: Run Health Check (orquestador principal)

**Objective:** `run-all-checks.ts` itera subcuentas × monitores, respetando rate limits.

> ⚠️ **Idempotencia:** Si una ejecución del cron tarda más que el intervalo configurado, dos instancias podrían solaparse. Agregar un lock simple (ej. flag en `agency_config` o `SELECT ... FOR UPDATE`) para prevenir ejecuciones concurrentes.
>
> ⚠️ **Cache en serverless:** El `Map` de location tokens (`AgencyTokenManager`) y los buckets del `RateLimiter` solo sobreviven dentro de una misma invocación del cron. En Vercel (serverless) no hay garantía de reúso de instancia entre ticks de 5 minutos. Esto no rompe nada porque `run-all-checks.ts` itera todas las subcuentas dentro de una sola invocación, pero el cache **no ahorra** llamadas entre ciclos — solo evita intercambios repetidos dentro del mismo ciclo.

**Files:**
- Crear: `src/use-cases/run-health-check.ts`
- Crear: `src/use-cases/run-all-checks.ts`
- Crear: `src/use-cases/run-all-checks.test.ts`

### Task 5.2: Confidence Score Calculator

**Files:**
- Crear: `src/use-cases/calculate-confidence.ts`
- Crear: `src/use-cases/calculate-confidence.test.ts`

### Task 5.3: Dashboard Aggregator + Incident Detector

**Files:**
- Crear: `src/use-cases/get-dashboard-state.ts`
- Crear: `src/use-cases/detect-incident.ts`

---

## Fase 6: API Routes + Dashboard UI

### Task 6.1: API Routes

**Files:**
- Crear: `src/app/api/guardian/cron/route.ts`
- Crear: `src/app/api/guardian/health/route.ts`
- Crear: `src/app/api/guardian/health/[subaccountId]/route.ts`
- Crear: `src/app/api/guardian/config/route.ts`

### Task 6.2: Dashboard UI — Layout + Navegación

**Files:**
- Crear: `src/ui/layout/app-layout.tsx`
- Crear: `src/ui/layout/sidebar.tsx`
- Modificar: `src/app/layout.tsx`

### Task 6.3: Dashboard UI — Página Principal

**Files:**
- Crear: `src/ui/dashboard/dashboard-page.tsx`
- Crear: `src/ui/dashboard/client-group.tsx`
- Crear: `src/ui/dashboard/subaccount-card.tsx`
- Crear: `src/ui/dashboard/service-indicator.tsx`
- Crear: `src/ui/dashboard/confidence-score.tsx`
- Crear: `src/ui/dashboard/incident-timeline.tsx`

### Task 6.4: Dashboard UI — Configuración

**Files:**
- Crear: `src/ui/config/monitor-config-form.tsx`
- Crear: `src/ui/config/subaccount-manager.tsx`
- Crear: `src/app/(dashboard)/guardian/config/page.tsx`
- Crear: `src/app/(dashboard)/guardian/subaccounts/page.tsx`

---

## Fase 7: Cron + Deploy

### Task 7.1: Decidir Vercel Pro vs Scheduler Externo (ADR-008)

**Acción:** Evaluar costo (Vercel Pro $20/mes vs GitHub Actions gratuito con límite de frecuencia). Decisión de Leo.

### Task 7.2: Configurar Cron Job + CRON_SECRET

**Files:**
- Crear: `vercel.json` (si Vercel Pro) o `.github/workflows/cron.yml` (si GH Actions)

### Task 7.3: Deploy a Vercel + Variables de Entorno

```bash
# Instalar Vercel CLI si no está
npm i -g vercel
vercel --prod
```

**Variables en Vercel Dashboard:** `GHL_AGENCY_PIT`, `GHL_COMPANY_ID`, `GHL_WEBHOOK_SECRET`, `CRON_SECRET`, `SUPABASE_*`

### Task 7.4: Smoke Test End-to-End

- Cron se ejecuta → eventos en `health_events`
- Dashboard muestra estados actualizados
- Timeline muestra progresión de degradación

---

## Resumen de Archivos (87 archivos total estimado)

```
src/
├── domain/
│   ├── entities/          (4 archivos)
│   ├── value-objects/     (3 + 3 tests)
│   └── ports/             (4 archivos)
├── use-cases/             (5 + 3 tests)
├── adapters/
│   ├── ghl/               (5 + 3 tests, incluyendo oauth-strategies/)
│   ├── monitors/          (5 + 2 tests)
│   ├── supabase/          (4 + 1 test, incluyendo repos)
│   └── webhook/           (1 archivo)
├── ui/
│   ├── dashboard/         (6 archivos)
│   ├── config/            (2 archivos)
│   ├── layout/            (2 archivos)
│   └── shared/            (componentes de wacrm: ui/*)
├── app/
│   ├── (dashboard)/guardian/  (3 page.tsx)
│   └── api/guardian/          (5 route.ts + 1 test)
├── lib/
│   └── supabase/          (2 archivos)
└── scripts/               (1 seed script)
```

---

## Principios

- **TDD:** RED (failing test) → GREEN (minimal code) → REFACTOR
- **Commits atómicos** después de cada task
- **DRY:** extraer helpers, no repetir lógica entre monitores
- **YAGNI:** no implementar Phone Monitor completo si el MVP no lo requiere
- **Clean Architecture:** domain/ no importa de adapters/ ni de ui/
