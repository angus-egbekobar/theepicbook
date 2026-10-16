# Environment Variables, Ports, Storage and Health Checks - The EpicBook

Repository analysed: https://github.com/pravinmishraaws/theepicbook

## 1. Application analysis

TheEpicBook (`package.json` name `mvc_project2`) is a server-rendered **monolith**: Node.js + Express, Handlebars views (`express-handlebars`), Sequelize ORM with `mysql2`. The start command is `npm start` (`node server.js`). There is no frontend build step and no separate API service, so the stack has three containers: `nginx`, `app` and `mysql`.

## 2. Services and ports

| Service | Image / build | Internal port | Published to host | Networks |
|---------|---------------|---------------|-------------------|----------|
| nginx   | `nginx:alpine` | 80   | **80:80 (public)** | edge-net |
| app     | built from the multi-stage Dockerfile | 8080 (`PORT`, default 8080 in `server.js`) | none (`expose` only) | edge-net, db-net |
| mysql   | `mysql:8.0` | 3306 | none | db-net (`internal: true`) |

Cloud firewall (inbound): 22 from my IP only, 80 from anywhere. 8080 and 3306 stay closed.

## 3. Environment variable names

Variables the application code reads (found with `grep -rn "process.env"`):

| Variable | Where it is read | Purpose | Value in the stack |
|----------|------------------|---------|--------------------|
| `PORT` | `server.js` | Express listen port, default `8080` | `8080` |
| `NODE_ENV` | `models/index.js` | Selects the block in `config/config.json`, default `development` | `production` |
| `JAWSDB_URL` | `models/index.js`, via `"use_env_variable"` in the `production` block | Full MySQL connection URL | `mysql://<DB_USER>:<DB_PASSWORD>@mysql:3306/<DB_NAME>` |

With `NODE_ENV=production`, the app ignores the hard-coded `development` and `test` database settings in `config/config.json` and connects with the URL in `JAWSDB_URL`. No source change is needed. The host in that URL is the Compose service name `mysql`.

Variables defined in the git-ignored `.env` and used by Compose:

| Variable | Used by | Purpose |
|----------|---------|---------|
| `DB_NAME` | mysql, `JAWSDB_URL` | Database name (`epicbook`) |
| `DB_USER` | mysql, `JAWSDB_URL` | Application database user |
| `DB_PASSWORD` | mysql, `JAWSDB_URL` | Application user password |
| `MYSQL_ROOT_PASSWORD` | mysql, health check | Root password |

Mapping to the official `mysql` image variables: `MYSQL_DATABASE=${DB_NAME}`, `MYSQL_USER=${DB_USER}`, `MYSQL_PASSWORD=${DB_PASSWORD}`.

Secrets are random hex strings (`openssl rand -hex 16`), so they need no URL-encoding inside `JAWSDB_URL`. Real values live only in `.env`, which is ignored by Git. A `.env.example` with placeholders is committed. `config/config.json` is left as shipped and contains no real credentials.

## 4. Persistent data and initial data

| Data | Location in container | Host storage | Type |
|------|----------------------|--------------|------|
| MySQL database files | `/var/lib/mysql` | named volume `mysql_data` | Docker volume |
| Nginx access and error logs | `/var/log/nginx` | `./logs/nginx` | bind mount |

Initial data comes from the repo's `db/` folder (`BuyTheBook_Schema.sql`, `author_seed.sql`, `books_seed.sql`, plus the CSV sources). These files are mounted read-only into `/docker-entrypoint-initdb.d` and run in order (schema, authors, books). MySQL runs them only on first start, when `mysql_data` is empty.

`docker compose down` keeps both the volume and the logs. `docker compose down -v` deletes `mysql_data` and must not be used before persistence evidence is captured.

## 5. Health-check method

| Service | Check | Healthy when |
|---------|-------|--------------|
| mysql | `mysqladmin ping -h 127.0.0.1 -uroot -p"$MYSQL_ROOT_PASSWORD"` | exit code 0 ("mysqld is alive") |
| app | `wget -qO- http://127.0.0.1:8080/ > /dev/null` | HTTP 200 from the home page |
| nginx | `wget -qO- http://127.0.0.1/ > /dev/null` | HTTP 200 through the proxy |

Start order is enforced with `depends_on: condition: service_healthy`: `app` waits for a healthy `mysql`, and `nginx` waits for a healthy `app`.

## 6. How these values were discovered (run from the repo root)

```bash
cat package.json                                                    # start script and dependencies
grep -rn "process.env" --include=*.js . --exclude-dir=node_modules  # PORT, NODE_ENV, JAWSDB_URL
cat config/config.json                                              # development, test, production blocks
ls db                                                               # schema and seed files
```

## 7. Verification

```bash
docker compose ps                         # all services Up, mysql and app healthy
docker compose port nginx 80              # only nginx publishes a port
ss -tlnp | grep -E '8080|3306' || echo "app and mysql are not exposed on the host"
curl -I http://localhost                  # 200 through the reverse proxy
```
