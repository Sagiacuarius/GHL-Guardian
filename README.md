# GHL Sentinel

Plataforma de observabilidad para GoHighLevel — monitoreo proactivo de subcuentas.

## Stack

- Next.js 16 + React 19 + TypeScript 6
- Supabase (PostgreSQL + Auth + RLS + Realtime)
- shadcn/ui + Radix + Tailwind 4 + Recharts
- Vitest (testing)

## Arquitectura

Clean Architecture en monorepo Next.js: `domain/` → `use-cases/` → `adapters/` → `ui/`

Ver [`architecture.md`](./architecture.md) para el diseño completo.

## Estado

MVP en planificación.
