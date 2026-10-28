# Contributing to S3 Explorer

Gracias por tomarte el tiempo de contribuir. Este documento describe cómo proponer
cambios, qué se espera de un pull request y qué se revisa antes de fusionarlo.

## Antes de empezar

- Lee [`README.md`](README.md) para el setup local y [`SECURITY.md`](SECURITY.md) para el
  modelo de seguridad. Hay invariantes que **no se deben romper** (credenciales S3 que
  nunca tocan el servidor, CSP estricto, guarda SSRF de endpoints, allowlists de
  Host/Origin, contenedor `read_only`).
- Para vulnerabilidades **no abras un issue público**: usa los advisories privados de
  GitHub (ver `SECURITY.md`).
- Al participar aceptas el [`CODE_OF_CONDUCT.md`](CODE_OF_CONDUCT.md).

## Setup local

```bash
npm install
npm run build       # tsc -b → dist/ + public/dist/
npm run dev         # build + start en 127.0.0.1:4569
npm test            # vitest (unit + integration)
```

No hay dev-server con hot reload: la app necesita auth real + bóveda + S3, así que la
verificación funcional se hace con `docker compose up -d --build`.

## Flujo de trabajo

1. Abre un issue describiendo el problema o la propuesta (salvo cambios triviales como
   typos).
2. Crea una rama desde `main`: `feat/*`, `fix/*`, `chore/*`, `security/*` o `docs/*`.
3. Haz commits pequeños y con mensajes en formato [Conventional Commits](https://www.conventionalcommits.org/)
   (`feat:`, `fix:`, `chore:`, `docs:`, `security:`…). El proyecto usa Semantic Versioning
   y Renovate asume `:semanticCommits`.
4. Abre un Pull Request contra `main`. Describe **qué** cambia y **por qué**, y enlaza el
   issue. Si toca UI, adjunta captura.
5. Asegúrate de que todos los jobs de CI pasen.

`main` está protegida: todo entra por PR con CI verde.

## Qué revisa CI

| Job | Qué valida |
|---|---|
| `secrets` | gitleaks contra el repo (`.gitleaks.toml`) |
| `audit` | `npm audit --audit-level=high` |
| `lint` | `npm run typecheck` + hadolint del `Dockerfile` |
| `test` | `npx vitest run` (unit + integration) |
| `caddyfile` | `caddy validate` del `Caddyfile` |
| `image-scan` | Trivy filesystem scan (HIGH/CRITICAL, `--ignore-unfixed`) |

Reproduce los gates localmente antes de abrir el PR:

```bash
npm run typecheck
npm test
npm audit --audit-level=high
caddy validate --config Caddyfile   # requiere DOMAIN set
hadolint Dockerfile
```

## Estándares de código

- **TypeScript en todo**: el backend vive en `src/`, la SPA en `public/src/`. Nada de JS
  suelto.
- **Dependencias**: el proyecto es deliberadamente delgado. Una dependencia nueva necesita
  justificación explícita en el PR.
- **Migraciones de BD**: agrega `src/db/migrations/00N_nombre.ts` con `version` + `sql` y
  regístrala en `src/db/migrations.ts`. Cambios aditivos con
  `ALTER TABLE ... ADD COLUMN ... DEFAULT`.
- **Strings de UI**: español. Comentarios: mezcla español/inglés, sin problema.
- **Errores**: todo handler pasa por `wrap()` y `sanitizeError()`; nunca filtres detalle
  interno al cliente. El detalle completo va a `console.error`.
- **Auditoría**: acciones destructivas o relevantes para auth deben emitir un evento con
  `audit(evento, campos)`. Nunca loguees passwords, secrets TOTP ni jtis de otros usuarios.
- **Tests**: los cambios de seguridad y de rutas necesitan tests. `tests/unit` para lógica
  pura, `tests/integration` con `supertest` y el mock de S3.

## Licencia de las contribuciones

Al enviar un PR aceptas que tu contribución se publique bajo la licencia MIT del proyecto
(ver [`LICENSE`](LICENSE)).
