# Portal PBX Multi-Tenant — Despliegue con Portainer

Portal web de reportería para PBXware con dashboard estilo Power BI. Cada tenant ingresa con sus credenciales y ve solo los datos de su empresa: llamadas del día/semana/mes, desglose por departamento y por usuario.

## Arquitectura

```
┌─────────┐    ┌─────────┐    ┌──────────┐    ┌─────────────┐
│ Browser │ ─→ │  Nginx  │ ─→ │ Frontend │    │ PBXware API │
│         │    │  :8080  │    │ (HTML/JS)│    │             │
└─────────┘    └────┬────┘    └──────────┘    └──────▲──────┘
                    │                                 │
                    └─→ ┌─────────┐    ┌───────────┐  │
                        │ Backend │ ←─→│ PostgreSQL│  │
                        │ Node.js │    │           │  │
                        └────┬────┘    └───────────┘  │
                             └─────────────────────────┘
                              (sincronización cada 5 min)
```

## Estructura del proyecto

```
pbx-portal/
├── docker-compose.yml      # Stack completo
├── .env.example            # Variables de entorno (copiar a .env)
├── backend/                # API Node.js
│   ├── Dockerfile
│   ├── package.json
│   └── src/
│       ├── server.js
│       ├── routes/         # auth, dashboard, calls, admin
│       ├── services/       # PBX client, sync, scheduler
│       ├── middleware/     # JWT auth
│       └── utils/          # db, crypto
├── frontend/               # SPA con HTML + Chart.js
│   ├── Dockerfile
│   ├── default.conf
│   └── public/
│       ├── index.html
│       ├── styles.css
│       └── app.js
├── nginx/                  # Reverse proxy
│   └── nginx.conf
└── db/
    └── init.sql            # Esquema PostgreSQL
```

## Despliegue en Portainer

### Opción A: Stack desde repositorio Git

1. En Portainer ir a **Stacks** → **Add stack**
2. Nombre: `pbx-portal`
3. Build method: **Repository**
4. Repository URL: el repositorio donde subas estos archivos
5. Compose path: `docker-compose.yml`
6. En **Environment variables** definir (mínimo):
   ```
   DB_PASSWORD=password_seguro_aqui
   JWT_SECRET=secreto_de_minimo_32_caracteres_aleatorios
   ENCRYPTION_KEY=clave_de_exactamente_32_caracter
   PBX_HOST=pbx.mifijo.com
   PORTAL_PORT=8080
   ```
7. **Deploy the stack**

### Opción B: Subir archivos manualmente

1. Comprimir la carpeta `pbx-portal` en un .zip
2. En el host del servidor descomprimirla en `/opt/pbx-portal/`
3. Copiar `.env.example` a `.env` y editar valores
4. En Portainer: **Stacks** → **Add stack** → Build method: **Upload**
5. Subir el `docker-compose.yml`
6. Definir variables de entorno
7. **Deploy the stack**

### Generar las claves seguras

En cualquier terminal Linux:

```bash
# JWT_SECRET (64 caracteres aleatorios)
openssl rand -base64 48

# ENCRYPTION_KEY (exactamente 32 caracteres)
openssl rand -base64 24 | cut -c1-32

# Password de BD
openssl rand -base64 24
```

## Primera configuración (después del despliegue)

### 1. Generar API key en PBXware

1. Ingresar al panel de PBXware
2. **Admin Settings → API key**
3. Generar una key (mínimo 10 caracteres). Guardarla.

### 2. Crear hash de password para el admin

El SQL inicial crea un usuario `admin` pero con password placeholder. Crear el hash real:

```bash
# Desde el host:
docker exec -it pbx_portal_backend node -e "console.log(require('bcrypt').hashSync('TuPasswordAdmin', 10))"
```

Copia el hash que imprime.

### 3. Actualizar en la base de datos

```bash
docker exec -it pbx_portal_db psql -U pbxportal -d pbxportal
```

Dentro de psql:

```sql
-- Reemplazar password del admin con el hash real
UPDATE users SET password_hash = 'EL_HASH_QUE_COPIASTE' WHERE username = 'admin';

-- Ver tenants
SELECT * FROM tenants;
```

### 4. Configurar API key del PBX

Hacer login en el portal en `http://tu-servidor:8080`:
- Tenant code: `001`
- Usuario: `admin`
- Contraseña: la que definiste

Luego, mediante la API del portal (Postman / curl):

```bash
# Obtener token
curl -X POST http://tu-servidor:8080/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"tenantCode":"001","username":"admin","password":"TuPasswordAdmin"}'

# Configurar API key del PBX (usar el token devuelto)
curl -X POST http://tu-servidor:8080/api/admin/tenant-settings \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer EL_TOKEN' \
  -d '{
    "pbxApiKey":"tu_api_key_del_pbxware",
    "pbxHost":"pbx.mifijo.com",
    "pbxServerId": 2
  }'

# Forzar sincronización inicial (últimos 30 días)
curl -X POST http://tu-servidor:8080/api/admin/sync \
  -H 'Authorization: Bearer EL_TOKEN' \
  -d '{"daysBack": 30}'
```

> El `pbxServerId` corresponde al **Tenant ID** del PBXware (lo obtienes con la API `pbxware.tenant.list`). El ID 1 es siempre System, los tenants reales empiezan en 2.

## Agregar más tenants

Para cada nueva empresa cliente:

```sql
-- 1. Crear el tenant en la BD
INSERT INTO tenants (name, tenant_code, pbx_server_id, pbx_api_key_encrypted, pbx_host)
VALUES ('Empresa XYZ', '002', 3, 'PENDIENTE_CONFIGURAR', 'pbx.mifijo.com');

-- 2. Crear el usuario admin del tenant
INSERT INTO users (tenant_id, username, email, password_hash, role)
VALUES (
  (SELECT id FROM tenants WHERE tenant_code = '002'),
  'admin',
  'admin@empresaxyz.com',
  'HASH_BCRYPT_DEL_PASSWORD',
  'admin'
);
```

Luego el admin de esa empresa hace login y configura su API key via `/api/admin/tenant-settings`.

## Endpoints principales

| Método | Ruta | Descripción |
|--------|------|-------------|
| POST | `/api/auth/login` | Login (devuelve JWT) |
| GET  | `/api/auth/me` | Datos del usuario actual |
| POST | `/api/auth/change-password` | Cambiar contraseña |
| GET  | `/api/dashboard/summary` | KPIs hoy/semana/mes |
| GET  | `/api/dashboard/calls-by-day` | Serie diaria |
| GET  | `/api/dashboard/calls-by-hour` | Por hora |
| GET  | `/api/dashboard/calls-by-weekday` | Por día de la semana |
| GET  | `/api/dashboard/calls-by-department` | Por departamento |
| GET  | `/api/dashboard/calls-by-user` | Top usuarios |
| GET  | `/api/calls` | Listado paginado con filtros |
| POST | `/api/admin/sync` | Disparar sync manual |
| POST | `/api/admin/tenant-settings` | Actualizar API key del PBX |
| GET  | `/api/admin/users` | Listar usuarios del tenant |
| POST | `/api/admin/users` | Crear usuario |

## Sincronización

- **Cada 5 minutos** (configurable con `SYNC_INTERVAL_MINUTES`): sincroniza CDRs del último día.
- **Diariamente a las 3am**: sincroniza últimos 30 días + departamentos + extensiones.
- **Manual**: `POST /api/admin/sync` con `daysBack` para forzar.

## Solución de problemas

**No aparecen llamadas:**
- Verificar en BD: `SELECT * FROM sync_status;` — ver si hay error.
- Verificar logs del backend: `docker logs pbx_portal_backend`.
- Confirmar que la API key esté configurada: `SELECT pbx_api_key_encrypted FROM tenants;` no debe decir `PENDIENTE_CONFIGURAR`.
- Verificar conectividad al PBXware: `docker exec pbx_portal_backend wget -qO- https://pbx.mifijo.com/?apikey=TUKEY&action=pbxware.dashboard.calls`

**Error de SSL al conectar al PBX:**
- El cliente ya está configurado con `rejectUnauthorized: false` (acepta autofirmados). Si el PBX usa http en vez de https, modificar `backend/src/services/pbxClient.js`.

**Departamentos no se asignan a llamadas:**
- Solo se asignan a llamadas nuevas tras la sincronización. Para reprocesar histórico, ejecutar:
  ```sql
  UPDATE cdrs c SET department_id = (
    SELECT (department_ids)[1] FROM extensions e 
    WHERE e.tenant_id = c.tenant_id AND e.extension_number = c.to_extension
  ) WHERE c.department_id IS NULL;
  ```

## Seguridad

- ✅ Passwords con bcrypt (10 rounds)
- ✅ JWT con expiración de 12 horas
- ✅ API keys del PBX cifradas con AES-256-GCM
- ✅ Aislamiento de datos por `tenant_id` en cada query
- ✅ Rate limiting (300 req / 15 min por IP)
- ✅ Helmet (headers de seguridad)
- ✅ Contenedor backend corre como usuario no-root

**Recomendaciones adicionales:**
- Poner el portal detrás de un reverse proxy con HTTPS (Traefik, Caddy, Nginx Proxy Manager).
- Restringir el acceso por IP/VPN si es un portal interno.
- Hacer backup periódico del volumen `pbx_db_data`.

## Stack actualizable

Para actualizar la imagen en Portainer:
1. Pull de los cambios del repositorio (si usaste opción A).
2. **Stack → Editor → Update the stack** → marcar **Re-pull image and redeploy**.

Los datos en el volumen `pbx_db_data` persisten entre actualizaciones.
