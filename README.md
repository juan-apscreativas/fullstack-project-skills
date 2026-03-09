# fullstack-project-skills

Plugin de Claude Code con skills para desarrollo full-stack: Laravel Modular Monolith en backend, Next.js 15 Feature-Based en frontend, orquestación de proyectos desde PRD, gestión del contrato API y despliegue en Cloudflare Workers.

## Instalacion

Requiere Claude Code v1.0.33+.

```shell
# 1. Agregar el marketplace
/plugin marketplace add juan-apscreativas/fullstack-project-skills

# 2. Instalar el plugin
/plugin install fullstack-project-skills@fullstack-skills
```

Reemplaza `owner` con el usuario u organizacion de GitHub donde se aloja el repo.

Para desarrollo local:

```shell
claude --plugin-dir ./fullstack-project-skills
```

## Skills incluidas

| Skill | Tipo | Activa cuando... |
|-------|------|-----------------|
| `project-orchestration` | Proceso | Inicializar un proyecto nuevo desde un PRD |
| `prd-analysis` | Proceso | Analizar un PRD, identificar módulos/features, asignar niveles |
| `backend-module-dev` | Proceso | Implementar o modificar un módulo Laravel |
| `frontend-feature-dev` | Proceso | Implementar o modificar una feature Next.js |
| `api-contract-sync` | Proceso | Gestionar el contrato API entre repos (breaking changes, mapa) |
| `laravel-modular-monolith` | Tecnología | Cualquier trabajo en el repo Laravel backend |
| `nextjs-feature-based` | Tecnología | Cualquier trabajo en el repo Next.js frontend |
| `nextjs-cloudflare-deployment` | Tecnología | Despliegue del frontend en Cloudflare Workers (OpenNext) |

## Stack cubierto

**Backend:** Laravel 12, PostgreSQL, Redis, Laravel Sail, Reverb, Scramble (OpenAPI 3.1.0), Pest PHP, Laravel Pint (PSR-12), Nightwatch

**Frontend:** Next.js 15 (App Router), TypeScript strict, Tailwind CSS, shadcn/ui, Zustand, Zod, Vitest + Testing Library, Storybook, Docker standalone

**Deploy alternativo:** Cloudflare Workers vía `@opennextjs/cloudflare` (Next.js 15 — v16 no soportado aún)

**Contrato:** API REST documentada por Scramble, sincronizada en el root `CLAUDE.md` Module/Feature Map

## Principio central

Cada skill tiene una sola regla inquebrantable (HARD-GATE) y un anti-patrón principal a evitar. La documentación —`CLAUDE.md` raíz, CLAUDE.md por módulo/feature, ADRs— se actualiza en el **mismo commit** que el código. Un cambio sin actualización de docs está incompleto.

## Flujo típico de un proyecto nuevo

1. Recibir PRD → activar `project-orchestration`
2. `project-orchestration` delega a `prd-analysis` para extraer módulos, features y reglas
3. Se crean los repos siguiendo `laravel-modular-monolith` (backend) y `nextjs-feature-based` (frontend)
4. Cada tarea de implementación activa `backend-module-dev` o `frontend-feature-dev`
5. Cambios en endpoints activan `api-contract-sync` para mantener el mapa sincronizado
6. Si el destino es Cloudflare, `nextjs-cloudflare-deployment` reemplaza la sección Docker de `nextjs-feature-based`
