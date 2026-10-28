# Graphify — notas de análisis del repositorio

Fecha del análisis: 2026-10-28
Herramienta: [Graphify](https://github.com/Graphify-Labs/graphify) `graphify 0.9.97`
Backend de extracción: **solo AST** (`--code-only`, sin LLM) — no se usó ningún proveedor
de modelo, así que las comunidades quedan con etiquetas genéricas (`Community N`) y las
aristas son 99 % `EXTRACTED` (1 % `INFERRED`).

## Cómo se generó

```bash
uv tool install graphifyy        # instala el CLI `graphify` (Python 3.12 + uv 0.12.9)
graphify extract . --code-only   # AST sobre 116 archivos de código
graphify cluster-only . --no-label
```

Salida en `graphify-out/` (ignorado por git, ver `.gitignore`):

| Archivo | Contenido |
|---|---|
| `graph.json` | 999 nodos, 2660 aristas, 54 comunidades |
| `GRAPH_REPORT.md` | reporte legible (hubs, comunidades, gaps, preguntas sugeridas) |
| `graph.html` | visor interactivo (D3) del grafo |

Corpus: 116 archivos de código (servidor `src/`, SPA `public/src/`, tests `tests/`).
19 archivos no clasificables (`.gitignore`, `Caddyfile`, `.gitleaks.toml`, etc.) se omitieron.
10 archivos de código no rindieron símbolos (re-exports o tests que solo importan).

## Nodos "god" (hubs arquitectónicos)

| # | Nodo | Grado | Lectura |
|---|---|---|---|
| 1 | `$` (`public/src/util/dom.ts`) | 69 | helper de query/selector: TODO el frontend pasa por acá |
| 2 | `getPane()` (`public/src/state.ts`) | 68 | resolución del panel activo bajo el Proxy de `PaneState` |
| 3 | `toast()` (`public/src/util/toast.ts`) | 55 | notificación: acoplada a casi todo módulo de UI |
| 4 | `paneEl()` (`public/src/util/dom.ts`) | 43 | acceso al DOM por panel |
| 5 | `loadObjects()` (`public/src/objects/grid.ts`) | 37 | carga de listados: punto de convergencia de router/panes/uploads |
| 6 | `errorIdOpts()` (`public/src/util/toast.ts`) | 36 | mapeo de errores de API → mensajes |
| 7 | `confirmDialog()` | 32 | UX de confirmación destructiva |
| 8 | `register()` (`public/src/router/index.ts`) | 30 | tabla de rutas de la SPA |
| 9 | `openModal()` | 29 | capa de modales |
| 10 | `setActivePane()` (`public/src/state.ts`) | 28 | cambio de panel activo |

En el backend los nodos con más grado son `buildClient()` (`src/s3/client.ts`, grado 8) y
`endpointBlockReason()` (grado 7, la guarda SSRF) — es decir, el grafo confirma que el
acoplamiento alto está **en el frontend, no en el servidor**.

## Comunidades principales (de 54; 48 con ≥3 nodos)

| Comunidad | Cohesión | Contenido |
|---|---|---|
| 0 | 0.05 | SPA core: `api()`, `authHeaders()`, filas de bucket, preview, panes |
| 1 | 0.06 | Auth backend: `requireAuth`, `requireAdmin`, TOTP, `audit()`, `bcryptjs` |
| 2 | 0.12 | Test harness: `buildTestApp()`, `createTestUser()`, mocks S3, vitest |
| 3 | 0.08 | Guarda SSRF de endpoint: `buildClient`, `hostToIp`, `METADATA_BLOCKLIST`, RFC-1918 |
| 4 | 0.09 | SQLite: `better-sqlite3`, `DB`, migraciones |
| 5 | 0.09 | Wiring de la SPA (`configure*`, `PANES`, `tryRestoreFromUrl`) |
| 6 | 0.13 | Contratos de API: `zod`, `validateBody`, routers, `buildCopySource` |
| 8 | 0.18 | Bóveda AES-256-GCM: `deriveKey`, `encryptVault`, `decryptVault`, PBKDF2 |
| 12 | 0.17 | Modelo de paneles: `PaneState`, `adoptPaneState`, `NAV_HISTORY_MAX` |
| 14–15 | 0.21 / 0.11 | Búsqueda: UI de search + índice FTS5 (`searchIndex`, `rebuildBucketIndex`) |
| 16 | 0.27 | Bootstrap de auth en el navegador (overlay, user chip) |
| 30 | 0.45 | Resizers de layout (la comunidad más cohesiva del frontend) |

## Enlaces cross-file notables

- `tests/fixtures/users.ts → src/auth/passwords.ts` (`hashPasswordSync`) y
  `→ src/auth/totp.ts` (`generateTotpSecret`, `encryptTotpSecret`): el harness de tests
  depende de helpers de producción reales, no de dobles.
- `tests/fixtures/users.ts → src/types.ts` (`UserRole`): el tipo compartido atraviesa
  fixtures y código de servidor.
- `src/server.ts → src/s3/client.ts` (`buildClient`): único punto donde el servidor
  construye clientes S3 a partir de headers.
- `src/s3/client.ts → src/s3/dual-client.ts`: la guarda SSRF se reutiliza en el cliente
  secundario de transferencias (source → dest).
- `public/src/main.ts` importa prácticamente todos los módulos de UI: es el nodo de
  entrada que convierte al frontend en un componente casi-monolítico desde el punto de
  vista del grafo.
- Cadenas largas tipo `initSearch() ← main.ts → objects/upload.ts → rename.ts ↔ vitest ↔
  @aws-sdk/client-s3 → indexer.ts → searchIndex()` (9 saltos, solo en modo `--undirected`)
  indican que el grafo **no** tiene un camino dirigido entre búsqueda-UI e índice: la
  relación pasa por la capa HTTP, que el AST no puede resolver.

## Observaciones

- **Sin ciclos de import** (`GRAPH_REPORT.md`: "Import Cycles: None detected").
- Nodos aislados: `RFC-1918`, `config:recommended`, `:pinAllExceptPeerDependencies`,
  `:semanticCommits` — etiquetas de documentación/config, no código muerto.
- Comunidades 0, 1 y 3 tienen cohesión < 0.10: candidatas a dividir en módulos más
  pequeños (SPA core, auth, guarda SSRF), pero el tamaño del proyecto no lo exige.
- `vitest` actúa como puente de betweenness (0.297) entre comunidades: el acoplamiento
  real que mide el grafo es *de tests*, no de producción.

## Cómo repetirlo

```bash
# requiere Python 3.10+ y uv (winget install astral-sh.uv)
uv tool install graphifyy
graphify extract . --code-only
graphify cluster-only . --no-label     # regenera GRAPH_REPORT.md + graph.html
graphify god-nodes --top 15
graphify explain "src/s3/client.ts::buildClient"
graphify query "how does a login become an authenticated session" --budget 3000
```

Nota: `public/src/util/dom.ts` declara `$` como export, por lo que Graphify reporta
`node extracted twice … keeping '$'` — es un falso positivo del extractor, no un bug.
