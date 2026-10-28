# S3 Explorer

Consola web tipo MinIO para administrar buckets en cualquier almacenamiento compatible con S3 (AWS S3, MinIO, Vultr, Wasabi, Backblaze B2, Cloudflare R2, DigitalOcean Spaces).

**Funcionalidades:**
- Multi-conexión: cada usuario define sus propios endpoints y credenciales.
- Bóveda local cifrada (AES-256-GCM) — las credenciales S3 nunca se guardan en el servidor.
- Explorador de dos paneles: navegación independiente, "abrir en segundo panel", navegación sincronizada estilo FileZilla, y cierre de cualquiera de los dos paneles.
- Subida con resolución de conflictos: reemplazar / renombrar (auto-sufijo) / saltar cuando el archivo ya existe.
- CRUD de carpetas: crear, renombrar, eliminar (con delete-marker o purga) y restaurar.
- Búsqueda recursiva en streaming con agrupación por carpetas y alcance por prefijo.
- Versionado con vista de versiones anteriores, restaurar, mostrar eliminados.
- Preview inline para imágenes, video, audio, PDF, código.
- Enlaces temporales firmados (5min / 1h / 24h).
- Login con usuario + contraseña + TOTP (Google Authenticator); códigos de recuperación de 128 bits; sin replay de TOTP.
- Gestión de usuarios desde la UI (admin).
- Audit log JSON line a stdout.

---

## Requisitos

- Servidor Linux con Docker y Docker Compose v2.
- Puertos 80 y 443 abiertos.
- Para Let's Encrypt automático: dominio público con DNS apuntando al servidor.
- Para deploy interno: nombre de host accesible en la red corporativa.
- **Full Disk Encryption (FDE)** en el host de producción (LUKS / dm-crypt en Linux, BitLocker en Windows). El volumen `users_data` contiene hashes bcrypt y secrets TOTP cifrados; aún así, FDE es defensa-en-profundidad contra robo físico u offline-clone del disco. Sin FDE, asumir que cualquier persona con acceso físico al servidor puede leer los datos en reposo.

---

## Instalación

### 1. Clonar el repositorio

```bash
git clone https://github.com/<owner>/s3-explorer.git
cd s3-explorer
```

> Si es la primera vez en el servidor: configura tu identidad git y la llave SSH antes:
> ```bash
> git config --global user.name "Nombre Apellido"
> git config --global user.email "tu-correo@ejemplo.com"
> ssh-keygen -t ed25519 -C "deploy@s3-explorer-prod"  # pega la pub key en GitHub → Settings → SSH and GPG keys
> ```

### 2. Crear el archivo `.env`

```bash
cp .env.example .env
```

Genera los 3 secretos requeridos:

```bash
echo "ADMIN_PASS=$(openssl rand -base64 24 | tr -d '+/=' | head -c 20)"
echo "SESSION_SECRET=$(openssl rand -hex 32)"
echo "TOTP_ENCRYPTION_KEY=$(openssl rand -hex 32)"
```

Pega esos 3 valores en `.env`. Configura también el dominio y usuario admin:

```env
DOMAIN=s3.empresa.com           # tu FQDN
ADMIN_USER=admin                # usuario inicial (cámbialo si quieres)
```

Los valores de `SESSION_TTL_SECONDS`, `MAX_UPLOAD_BYTES`, y `CHECK_HIBP` ya tienen defaults razonables — modifícalos solo si necesitas cambios específicos.

> ⚠️ **No rotes `TOTP_ENCRYPTION_KEY` después del primer enrolamiento.** Si lo cambias, todos los usuarios pierden su TOTP y el admin tiene que resetearles desde la UI. Guarda este valor en tu password manager corporativo.

### 3. Modo de TLS

**A) Dominio público con Let's Encrypt automático** (recomendado para internet):

- Asegúrate que el DNS A/AAAA de `DOMAIN` apunte al servidor.
- Caddy negocia el certificado automáticamente al arrancar.
- Renueva los certificados solo.

No requiere acción extra.

**B) Dominio interno con CA local de Caddy** (intranet sin DNS público):

Edita `Caddyfile` y descomenta la línea `tls internal`:

```caddyfile
{$DOMAIN} {
    tls internal
    ...
}
```

Luego de levantar, exporta el root cert y distribúyelo a los usuarios (Group Policy, MDM, manual):

```bash
docker exec s3-explorer-caddy cat /data/caddy/pki/authorities/local/root.crt > caddy-root.crt
```

Cada usuario instala `caddy-root.crt` en su SO como CA confiable (Windows: doble click → Equipo local → Entidades de certificación raíz de confianza).

### 4. Levantar el stack

```bash
docker compose up -d --build
```

Verifica:

```bash
docker compose ps
docker compose logs -f s3-explorer
```

Debes ver:

```
admin_bootstrapped username=admin
S3 Explorer listening on http://0.0.0.0:4569
  - Auth: username + password + TOTP
  - Max upload: 500 MB
```

### 5. Deploy en Coolify (alternativa a docker compose)

Coolify v4 por defecto despliega aplicaciones en modo **"Dockerfile-only application"** y **NO interpreta el `docker-compose.yml`** del repo (ignora `volumes:`, `environment:`, el contenedor de Caddy y la red). Si esa es la ruta que vas a usar, hay que configurar dos cosas en el UI antes del primer redeploy o perderás los TOTP enrolados cada vez:

1. **Environment Variables → Add:**
   - `DATA_DIR=/data` — sin esto la app cae al fallback `__dirname/../../data` que resuelve a `/app/data` (dentro de la imagen, se rehace en cada build).
   - El resto de variables del `.env` (`ADMIN_USER`, `ADMIN_PASS`, `SESSION_SECRET`, `TOTP_ENCRYPTION_KEY`, `DOMAIN`, `ALLOWED_HOSTS`, `ALLOWED_ORIGINS`, etc.) — Coolify NO las lee del `.env` del repo.

2. **Persistent Storage → + Add → Volume Mount:**
   - **Volume Name:** `s3-explorer-data`
   - **Destination Path in container:** `/data`
   - **Source Path:** vacío (managed volume)

Verificación post-deploy:

```bash
# El mount debe ser un named volume, NO un hash random
docker inspect <coolify-container> --format '{{json .Mounts}}'
# Esperado: "Name": "s3-explorer-data", no "Name": "9a73c926f..."

# /data debe contener la DB SQLite tras el primer login
docker exec <coolify-container> ls /data
# Esperado: app.db  app.db-shm  app.db-wal
```

Si quieres usar el stack completo de `docker-compose.yml` (con Caddy + healthcheck + cap_drop + read-only FS), cambia el deploy type en Coolify a **"Docker Compose"** en lugar de "Dockerfile". En ese caso el bloque `volumes:` y `environment:` del compose sí surten efecto y este paso 5 no aplica.

---

## Primer login (admin)

1. Abre `https://<tu-dominio>` en el navegador.
2. Login con `ADMIN_USER` / `ADMIN_PASS` del `.env`.
3. La app te lleva al **enrolamiento TOTP**: escanea el QR con Google Authenticator, Authy, 1Password, o Microsoft Authenticator. Ingresa el código de 6 dígitos para confirmar.
4. **Guarda los 8 códigos de recuperación** en un lugar seguro (1Password, papel impreso). Son tu salvavidas si pierdes el celular. No se vuelven a mostrar.
5. Ya estás dentro.

---

## Crear usuarios para el equipo

Como admin, click en el ícono **👥** abajo en la barra lateral.

1. **Crear usuario:** username + contraseña inicial (mín. 8 chars) + rol (User / Admin).
2. Comparte por canal seguro (1Password Send, Bitwarden Send) la contraseña inicial al usuario.
3. El usuario en su primer login:
   - Es forzado a cambiar la contraseña.
   - Enrola su TOTP.
   - Guarda sus códigos de recuperación.

**Acciones disponibles por usuario:**
- 🔑 Resetear TOTP (lo obliga a re-enrolar).
- 🔒 Resetear contraseña (lo obliga a cambiarla en el siguiente login).
- ⏸ Deshabilitar (su sesión actual expira en máximo `SESSION_TTL_SECONDS`).
- ▶ Re-habilitar.

---

## Crear una conexión S3 (cada usuario)

En la barra lateral, **+** junto a "Conexiones":

| Campo | Notas |
|---|---|
| Nombre | Cualquier alias |
| Endpoint | URL completa o vacío para AWS S3 oficial |
| Región | `us-east-1` default; ajusta según el proveedor |
| Path-style | ON para MinIO/Vultr/etc., OFF para AWS |
| Access Key / Secret Key | Credenciales del proveedor |
| Session Token | Solo si usas STS temporal |
| **Buckets** | Opcional. Si tu IAM no permite `s3:ListAllMyBuckets`, lista aquí los buckets uno por línea. |

Botón **Probar** valida antes de guardar. Si especificaste buckets, valida con `HeadBucket`. Si no, con `ListBuckets`.

**La primera vez** que guardes una conexión, la app te pide crear una **passphrase de bóveda**. Esta passphrase cifra tus conexiones localmente con AES-256-GCM derivado de PBKDF2 (250k iteraciones). **No se envía al servidor.** Si la olvidas, debes recrear tus conexiones.

---

## Permisos IAM mínimos (AWS)

Para que un usuario pueda usar la app sin `s3:ListAllMyBuckets`:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "s3:ListBucket",
      "s3:ListBucketVersions",
      "s3:GetObject",
      "s3:GetObjectVersion",
      "s3:PutObject",
      "s3:DeleteObject",
      "s3:DeleteObjectVersion",
      "s3:GetBucketVersioning",
      "s3:PutBucketVersioning"
    ],
    "Resource": [
      "arn:aws:s3:::mi-bucket-1",
      "arn:aws:s3:::mi-bucket-1/*",
      "arn:aws:s3:::mi-bucket-2",
      "arn:aws:s3:::mi-bucket-2/*"
    ]
  }]
}
```

En MinIO/Vultr/etc. usa la política equivalente del proveedor.

---

## Variables de entorno

| Variable | Default | Descripción |
|---|---|---|
| `DOMAIN` | — | FQDN público o interno (requerido) |
| `ADMIN_USER` | — | Usuario admin inicial (solo se usa al primer arranque) |
| `ADMIN_PASS` | — | Contraseña admin inicial (cámbiala desde la UI tras el primer login) |
| `SESSION_SECRET` | aleatorio efímero | Clave HMAC para firmar cookies. **DEBE estar set en producción.** Generar: `openssl rand -hex 32` |
| `TOTP_ENCRYPTION_KEY` | aleatorio efímero | Clave AES-256-GCM (64 hex chars) para cifrar TOTP secrets en la base de datos SQLite (`app.db`). **DEBE estar set en producción** o los TOTP enrolados se invalidan al reiniciar. Generar: `openssl rand -hex 32` |
| `SESSION_TTL_SECONDS` | `14400` (4h) | TTL absoluto de la sesión desde el login |
| `MAX_UPLOAD_BYTES` | `524288000` (500 MB) | Límite por archivo subido |
| `ALLOWED_HOSTS` | `localhost:4569,...` | Host headers aceptados (CSRF/DNS rebinding). Docker compose lo setea con `${DOMAIN}` |
| `ALLOWED_ORIGINS` | `https://${DOMAIN}` | Origins aceptados |
| `MAX_PRESIGN_SECONDS` | `86400` (24h) | Tope máximo del TTL de enlaces firmados |
| `DEFAULT_PRESIGN_SECONDS` | `300` (5min) | TTL default si el cliente no especifica |
| `DATA_DIR` | `/data` | Donde se guarda la DB SQLite (`app.db`) y el índice de archivos. Debe apuntar a un volumen persistente. |
| `SECURE_COOKIE` | `1` | Pone `Secure` flag en cookies (requiere HTTPS) |
| `TRUST_PROXY` | `1` | Confía en headers X-Forwarded-* del reverse proxy |

---

## Flujo de desarrollo (GitHub)

El repo vive en GitHub y sigue un flujo de **pull requests** (no se commitea directo a `main`).

### Para hacer un cambio

```bash
git checkout main && git pull
git checkout -b feat/mi-cambio       # o fix/, chore/, security/, docs/
# ... edita archivos ...
git add .
git commit -m "feat: descripción corta del cambio"
git push -u origin feat/mi-cambio
```

GitHub muestra un link "Create pull request" en el push output (o usa `gh pr create`). Ábrelo, asigna reviewer.

### Reglas de repo (configurar una vez en GitHub)

**Settings → General → Pull Requests:**
- ✅ "Allow squash merging" (deja un historial limpio); desactiva merge commits si quieres historia lineal.
- ✅ "Automatically delete head branches".

**Settings → Branches → Branch protection rules → `main`:**
- ✅ "Require a pull request before merging" (1 approval)
- ✅ "Require status checks to pass before merging" → selecciona los jobs de `.github/workflows/ci.yml`
- ✅ "Require conversation resolution before merging"
- ✅ "Do not allow bypassing the above settings"

Con esto, ni siquiera el owner del repo puede pisar `main` directo — todo pasa por PR + CI.

### Pipelines CI

El archivo `.github/workflows/ci.yml` define 6 jobs independientes:

| Job | Runner | Qué hace |
|---|---|---|
| `secrets` | `gitleaks/gitleaks-action` | busca secretos accidentales (config `.gitleaks.toml`) |
| `audit` | node 20, `npm ci --ignore-scripts` | `npm audit --audit-level=high` |
| `lint` | node 20 | `npm run typecheck` + hadolint del `Dockerfile` |
| `test` | node 20 + python3/make/g++ | `npm ci` completo + `npx vitest run` (necesita el binding nativo de `better-sqlite3`) |
| `caddyfile` | `caddy:2-alpine` | `caddy validate --config Caddyfile` con `DOMAIN=example.com` |
| `image-scan` | `aquasec/trivy` | Trivy filesystem scan (HIGH/CRITICAL, `--ignore-unfixed`) |

Corren en cada push a `main` y en cada pull request. La duración típica es ~3-5 min.

### Renovate (auto-update de dependencias)

`renovate.json` ya está en el repo y habilita `github-actions`, `dockerfile` y `docker-compose`. Para activarlo en GitHub:

1. Instala la app **Renovate** desde https://github.com/apps/renovate
2. Autoriza la app y selecciona este repositorio
3. Renovate abre el primer PR de "Configure Renovate" — fusiónalo
4. A partir de ahí, cada lunes Renovate abre PRs con updates de npm, imágenes base Docker, GitHub Actions y digests

Para self-hosted: corre el contenedor `renovate/renovate` con un token de GitHub (~15 min).

### Deploy en el servidor de producción

```bash
# Primera vez:
git clone https://github.com/<owner>/s3-explorer.git
cd s3-explorer
cp .env.example .env
# ... edita .env con secretos ...
docker compose up -d --build

# Actualizaciones (después de merge a main):
git pull origin main
docker compose up -d --build
docker compose logs -f --tail=50
```

> Tu `.env` está gitignored y nunca viaja en commits. Cada servidor tiene su propio `.env` que NO se versiona.

---

## Operaciones

### Ver logs

```bash
docker compose logs -f s3-explorer       # solo la app
docker compose logs -f caddy             # solo el proxy/TLS
docker compose logs --tail=100 -f        # ambos
```

El audit log son JSON lines en stdout de s3-explorer:

```json
{"t":"...","event":"login_ok","username":"juan","ip":"10.0.0.5","jti":"abc..."}
{"t":"...","event":"totp_fail","username":"juan","ip":"10.0.0.5"}
{"t":"...","event":"user_created","by":"admin","username":"juan","role":"user"}
{"t":"...","event":"versioning_enabled","username":"admin","bucket":"my-bucket"}
```

Si tienes ELK/Loki/Splunk, redirige stdout para auditoría centralizada.

### Backup de usuarios

Todo el estado de la app vive en un único archivo SQLite: `${DATA_DIR}/app.db` (`/data/app.db` dentro del contenedor, en el volumen `users_data`). Ya **no** existe `users.json` — la 1.2.0 migró a SQLite y un importador de una sola vez (`src/startup/migrate-users-json.ts`) sólo absorbe el archivo legado si lo encuentra.

La base corre en modo **WAL**, así que copiar `app.db` con el proceso vivo puede perder las últimas transacciones (quedan en `app.db-wal`). La forma consistente es usar la API de backup de SQLite desde el propio contenedor:

```bash
# 1. Snapshot consistente dentro del contenedor (soporta WAL)
docker compose exec s3-explorer node -e "require('better-sqlite3')(process.env.DATA_DIR + '/app.db').backup(process.env.DATA_DIR + '/app-backup.db')"

# 2. Lo traes al host y limpias el temporal
docker compose cp s3-explorer:/data/app-backup.db ./app-backup-$(date +%F).db
docker compose exec s3-explorer rm /data/app-backup.db
```

Alternativa equivalente y más simple si puedes asumir el downtime: `docker compose stop s3-explorer`, luego `docker compose cp s3-explorer:/data/app.db ./app-backup-$(date +%F).db`, y `docker compose start s3-explorer`.

Qué contiene (`src/db/migrations/`, `src/auth/`):

| Dato | Cómo se guarda |
|---|---|
| `users.password_hash` | **argon2id** (m=19 MiB, t=2, p=1). Los hashes bcrypt heredados se aceptan y se re-hashean a argon2id en el siguiente login |
| `users.totp_secret` | Cifrado **AES-256-GCM** con `TOTP_ENCRYPTION_KEY` (formato `enc:v1:…`); filas antiguas pueden estar en claro |
| `users.recovery_codes_hashed` | Array JSON de hashes **bcrypt** de los códigos de recuperación (los usados quedan en `null`) |
| `revoked_sessions` | Lista de revocación de sesiones (`jti` + expiración) |
| `bucket_index`, `bucket_index_fts`, `bucket_index_status` | Índice de claves por usuario/conexión/bucket; `bucket_index_fts` es la tabla virtual **FTS5** (tokenizer `unicode61`) que hace la búsqueda rápida |

**Trátalo como secret**: contiene material de autenticación. Cifra el archivo en reposo, no lo commitees y no lo subas a almacenamiento sin cifrar. Sin `TOTP_ENCRYPTION_KEY`, los secrets TOTP del backup son indescifrables — guarda esa clave por separado (password manager), nunca junto al backup.

### Backup de certificados Caddy (Let's Encrypt)

```bash
docker run --rm -v s3-explorer_caddy_data:/data -v $(pwd):/backup alpine tar czf /backup/caddy-data-$(date +%F).tar.gz -C /data .
```

Si pierdes este volumen, Caddy re-emite certs automáticamente (sin downtime perceptible pero cuenta contra el rate-limit de Let's Encrypt).

### Reiniciar / actualizar

```bash
git pull
docker compose up -d --build
```

Reiniciar **invalida todas las sesiones activas** (la lista de revocación está en memoria). Los usuarios deben volver a logearse.

### Detener / borrar todo

```bash
docker compose down              # detiene, conserva datos
docker compose down -v           # detiene + borra app.db, certs Caddy (DESTRUCTIVO)
```

---

## Troubleshooting

**"Host not allowed"** al entrar al sitio
→ `ALLOWED_HOSTS` en el server no incluye tu dominio. Verifica `DOMAIN` en `.env`. Docker compose pasa `ALLOWED_HOSTS=${DOMAIN}` automáticamente.

**Caddy reinicia en loop con "wrong argument count for email"**
→ Versión vieja del Caddyfile con `email {$ACME_EMAIL}` cuando ACME_EMAIL está vacío. Asegúrate que tu Caddyfile no tiene esa línea (debe estar comentada o eliminada).

**Cliente ve "InvalidAccessKeyId" o `not authorized for s3:ListAllMyBuckets`**
→ La IAM del cliente no tiene `s3:ListAllMyBuckets`. Edita la conexión y agrega los buckets manualmente en el campo "Buckets" (uno por línea).

**Usuario olvidó passphrase de bóveda**
→ La passphrase es local; el servidor no la tiene. El usuario debe entrar a su navegador, abrir DevTools → Application → Local Storage → borrar `s3-explorer-vault-v1` y recrear sus conexiones desde cero.

**Usuario perdió celular con TOTP**
→ Si tiene códigos de recuperación, usa uno (8 disponibles, uno por uso). Si no, el admin entra a "Usuarios" y le hace **Resetear TOTP**: en el siguiente login se le pide re-enrolar.

**Quiero matar todas las sesiones ahora (incidente de seguridad)**
→ `docker compose restart s3-explorer`. La in-memory revocation list se pierde pero `SESSION_SECRET` permanece, así que las cookies viejas siguen siendo firmas válidas. Para invalidar de verdad: regenera `SESSION_SECRET` en `.env` y reinicia. Todos relogean.

**El stack está en healthy pero https da 502**
→ Caddy aún no tiene cert. Mira `docker compose logs caddy` — probablemente DNS o puerto 80 bloqueado. Mientras se resuelve, Caddy sirve HTTP en 80 con un challenge ACME activo.

**Quiero rotar `SESSION_SECRET`**
→ Cambia el valor en `.env`, `docker compose up -d --force-recreate s3-explorer`. Todas las sesiones existentes quedan inválidas y los usuarios relogean.

---

## Modelo de seguridad

| Capa | Mecanismo | Qué protege |
|---|---|---|
| Transport | TLS via Caddy (Let's Encrypt o CA local) | Snooping del Wi-Fi/LAN |
| Auth | Username + password (bcrypt cost 11) + TOTP (HMAC-SHA1) | Acceso no autorizado al app |
| Sesión | Cookie firmada HMAC-SHA256, httpOnly, secure, sameSite=strict, TTL absoluto | Robo de cookie (XSS, MitM) |
| CSRF | Origin/Host allowlist | Sites maliciosos haciendo requests |
| Content | CSP estricto + nosniff + X-Frame-Options DENY | XSS / clickjacking |
| Bóveda S3 | AES-256-GCM con clave PBKDF2 de passphrase del usuario, almacenada solo en navegador | Volcado de localStorage / acceso al perfil del navegador |
| Versionado | Backend rechaza desactivarlo | Pérdida accidental de datos |
| Inline preview | Force `text/plain` para HTML/SVG/JS/XML | XSS de same-origin via blobs |
| Rate limit | Login y TOTP: 5 intentos / 15 min / IP+user | Brute force |

**Lo que NO protege:**
- Acceso físico al equipo del usuario con sesión activa → atacante puede operar como el user (la passphrase de bóveda se cachea en memoria del browser hasta cerrar la pestaña).
- Compromiso del servidor → atacante puede leer `app.db` (passwords hasheados con argon2id, secrets TOTP cifrados con AES-256-GCM).
- Phishing — si el usuario teclea su password+TOTP en un sitio falso, el atacante puede entrar antes que expire el código (30s).

Para entornos de alto riesgo, considera FIDO2/WebAuthn (no implementado).

---

## Licencia

[MIT](LICENSE) — Copyright (c) 2026 S3 Explorer contributors.

## Contribuir

Ver [CONTRIBUTING.md](CONTRIBUTING.md) y [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md).
Para reportar vulnerabilidades, sigue [SECURITY.md](SECURITY.md) (advisories privados de GitHub).
