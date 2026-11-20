# Trabajo final · Corte 3 · Sistemas Operativos (103027) — Grupo "Los Relicarios"

Un frontend en AWS EC2 (Nginx a mano) y un backend con base de datos en Azure (Docker), cada uno con su subdominio y HTTPS.

| Pieza | Dónde | URL |
|---|---|---|
| Frontend (Nginx + Certbot) | AWS EC2 `3.149.246.187` | https://losrelicarios.scaleautomatization.tech |
| Backend + BD (Docker Compose + Nginx + Certbot) | Azure VM `68.211.181.249` | https://api-losrelicarios.scaleautomatization.tech/api/sistemas |
| Imagen del backend | Docker Hub | https://hub.docker.com/r/enderman869/so-backend (`enderman869/so-backend:1.0`) |

```
frontend/   index.html + config.js (API_URL del backend)
backend/    app.py, requirements.txt, Dockerfile (escrito por nosotros)
db/         init.sql
compose.yaml           Azure: servicios db + backend
compose.local.yaml     solo para probar todo en el PC
nginx/aws-front.conf   sitio del frontend (EC2)
nginx/azure-api.conf   proxy inverso del backend (Azure)
.env.example           variables (el .env real NO se sube)
```

## Parte 1 · Frontend en AWS EC2 (sin Docker)

1. EC2 Ubuntu 24.04, t3.micro, llave `.pem`, Security Group con 22, 80 y 443.
2. Instalar Nginx y clonar este repo:
   ```bash
   sudo apt update && sudo apt install -y nginx git
   git clone https://github.com/enderman859/so-losrelicarios.git && cd so-losrelicarios
   ```
3. Copiar el frontend a una carpeta que Nginx pueda servir:
   ```bash
   sudo mkdir -p /var/www/losrelicarios && sudo cp -r frontend/* /var/www/losrelicarios/
   ```
4. Poner la URL de la API en `/var/www/losrelicarios/config.js`:
   `const API_URL = "https://api-losrelicarios.scaleautomatization.tech";`
5. Crear el sitio (`nginx/aws-front.conf` → `/etc/nginx/sites-available/front`), activarlo y quitar el `default`:
   ```bash
   sudo ln -s /etc/nginx/sites-available/front /etc/nginx/sites-enabled/front
   sudo rm -f /etc/nginx/sites-enabled/default
   sudo nginx -t && sudo systemctl reload nginx
   ```
6. Con el subdominio ya resolviendo a la IP, sacar el certificado:
   ```bash
   sudo apt install -y certbot python3-certbot-nginx
   sudo certbot --nginx -d losrelicarios.scaleautomatization.tech
   ```

## Parte 2 · Backend y base de datos en Azure (Docker)

1. VM Ubuntu 24.04 (Azure for Students), llave SSH, NSG con 22, 80 y 443 y nada más. Instalar Docker: `curl -fsSL https://get.docker.com | sudo sh`.
2. Imagen construida en el PC y publicada en Docker Hub (no se construye en la VM):
   ```bash
   docker buildx build --platform linux/amd64 -t enderman869/so-backend:1.0 --push ./backend
   ```
3. En la VM: clonar el repo y crear el `.env` a partir de `.env.example` (con una clave propia):
   ```bash
   git clone https://github.com/enderman859/so-losrelicarios.git && cd so-losrelicarios
   cp .env.example .env && nano .env
   docker compose up -d
   docker compose ps
   ```
   - `db`: `postgres:16`, volumen `pgdata`, `db/init.sql` montado en `docker-entrypoint-initdb.d`, **sin puertos publicados**.
   - `backend`: `image: enderman869/so-backend:1.0`, `depends_on: db`, `restart: unless-stopped`.
   - Dentro de la red de Compose el host de la base de datos es `db` (el nombre del servicio), por eso `DATABASE_URL=postgresql://USUARIO:CLAVE@db:5432/BASE`.
   - El backend solo se publica en `127.0.0.1:8000`; Nginx es el único que lo expone.
4. Proxy inverso con HTTPS (Nginx + Certbot en la VM, `nginx/azure-api.conf`):
   ```bash
   sudo apt install -y nginx certbot python3-certbot-nginx
   sudo cp nginx/azure-api.conf /etc/nginx/sites-available/api   # versión inicial, sin SSL
   sudo ln -s /etc/nginx/sites-available/api /etc/nginx/sites-enabled/api
   sudo rm -f /etc/nginx/sites-enabled/default
   sudo nginx -t && sudo systemctl reload nginx
   sudo certbot --nginx -d api-losrelicarios.scaleautomatization.tech
   ```
   > Los archivos de `nginx/` son la versión **final** (ya con los bloques de Certbot). Para repetir el despliegue desde cero, se parte de un `server` en el puerto 80 y Certbot agrega el 443.

## Integración

- En AWS: `config.js` apunta a `https://api-losrelicarios.scaleautomatization.tech`.
- En Azure: `ORIGEN_PERMITIDO=https://losrelicarios.scaleautomatization.tech` en el `.env`, y se recrea el backend con `docker compose up -d`.
- Prueba: abrir https://losrelicarios.scaleautomatization.tech → la tabla se llena con los datos de la base de datos de Azure.

## Rutas de la API

| Ruta | Respuesta |
|---|---|
| `GET /api/salud` | `{"estado":"ok"}` si la base de datos contesta |
| `GET /api/sistemas` | lista de sistemas operativos (año, nombre, tipo) |

## Probar todo en local

Ver `README-profe.md` (Postgres, backend y frontend en tres terminales) o, más corto: `docker compose -f compose.local.yaml up -d --build`.
