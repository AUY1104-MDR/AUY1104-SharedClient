# AUY1104-SharedClient

Repositorio de la **aplicación del ramo** desplegada en k3s para la Evaluación Sumativa 2 (AUY1104 — Ciclo de Vida del Software II).

Cubre el requisito de la sección 6 de la pauta: aplicación propia con `Dockerfile`, imagen publicada en Docker Hub, manifiestos Kubernetes y endpoint `/health` expuesto vía NodePort en el puerto **30090**.

## Estructura del repositorio

| Ruta | Propósito |
|---|---|
| `src/index.js` | Express app con rutas `/health`, `/api/saludo`, `/api/echo`, `/api/suma`. |
| `src/lib/ejemplo.js` | Lógica pura (probada con Jest). |
| `tests/*.test.js` + `test.js` | Suite Jest ejecutada por el pipeline reutilizable. |
| `Dockerfile` | Imagen `node:20-alpine`. Expone `3000`. |
| `k8s/deployment.yml` | `Deployment` con `readinessProbe` y `livenessProbe` apuntando a `/health`. |
| `k8s/services.yml` | `Service` tipo `NodePort` en el puerto `30090` (→ `targetPort 3000`). |
| `.github/workflows/client.yaml` | Llama al workflow reutilizable `deploy-api.yaml` del repo `AUY1104-SharedWorkflows`. |

## Flujo CI/CD

```
push tag v*
   ↓
AUY1104-SharedWorkflows/deploy-api.yaml@main
   ├─ npm install + npm test
   ├─ docker build + push (demo-api:<tag> + :latest)
   └─ scp k8s/ → ssh → kubectl apply → kubectl get pods
```

Disparador único: `push` de un tag `v*` (ej. `v1.0.0`). El versionamiento es obligatorio y no se permite ejecución manual.

Secrets/Variables necesarios:
- `secrets.DOCKER_USERNAME` / `secrets.DOCKER_PASSWORD` (organización)
- `secrets.EA2_SSH_PRIVATE_KEY` (organización)
- `vars.K3S_SERVER_PUBLIC_IP` (variable de repo)

## URL esperada

```
http://<IP_PUBLICA>:30090/health
```

## Imagen publicada

- Docker Hub: `marcdelrio/demo-api`
- Tags: el del push (`vX.Y.Z`) y `latest`.

## Rollback

```bash
# 1. Volver a la revisión anterior del Deployment
kubectl rollout undo deployment/demo-api
kubectl rollout history deployment/demo-api

# 2. Re-aplicar un tag previo de la imagen
kubectl set image deployment/demo-api api=marcdelrio/demo-api:v1.0.0
kubectl rollout status deployment/demo-api
```

---

## Ejecución local (referencia para desarrollo)

API académica mínima en **Node.js** y **Express**, pensada para practicar contenedores y pruebas con `curl`. Responde siempre en **JSON**.

## Requisitos

- Node.js 20+ (ejecución local)
- Docker (ejecución en contenedor)

## Ejecución local

```bash
npm install
npm start
```

Por defecto escucha en el puerto **3000**: `http://localhost:3000`.

## Docker

Construir la imagen (desde esta carpeta):

```bash
docker build -t auy1104-api-ejemplo .
```

Ejecutar el contenedor:

```bash
docker run --rm -p 3000:3000 -ti auy1104-api-ejemplo
```

Si el puerto **3000** de tu equipo ya está ocupado, usa otro puerto en el host (el primero del mapeo) y deja **3000** como puerto del contenedor:

```bash
docker run --rm -p 8080:3000 -ti auy1104-api-ejemplo
```

En ese caso las URLs de los ejemplos serían `http://localhost:8080/...`.

## Endpoints

| Método | Ruta | Descripción |
|--------|------|-------------|
| `GET` | `/health` | Estado del servicio |
| `GET` | `/api/saludo` | Saludo en JSON; query opcional `nombre` |
| `POST` | `/api/echo` | Devuelve en JSON el cuerpo enviado |

Cualquier otra ruta responde **404** con JSON: `{ "error": "Ruta no encontrada" }`.

## Ejemplos con `curl`

Sustituye `localhost:3000` por `localhost:8080` (u otro) si mapeaste el contenedor distinto, por ejemplo `-p 8080:3000`.

### `GET /health`

```bash
curl -s http://localhost:3000/health
```

### `GET /api/saludo`

Sin parámetros (usa el nombre por defecto `estudiante`):

```bash
curl -s http://localhost:3000/api/saludo
```

Con query `nombre`:

```bash
curl -s "http://localhost:3000/api/saludo?nombre=Duoc"
```

### `POST /api/echo`

Envía JSON en el cuerpo; la API responde con estado **201** y el objeto recibido en `recibido`.

```bash
curl -s -X POST http://localhost:3000/api/echo \
  -H "Content-Type: application/json" \
  -d '{"curso":"AUY1104","modulo":"Docker"}'
```

### Ruta inexistente (404)

```bash
curl -s http://localhost:3000/api/no-existe
```

## Estructura del proyecto

```
Docker de Ejemplo/
├── Dockerfile
├── .dockerignore
├── package.json
├── package-lock.json
├── README.md
└── src/
    └── index.js
```

## Variables de entorno

| Variable | Valor por defecto | Uso |
|----------|-------------------|-----|
| `PORT` | `3000` | Puerto donde escucha la app dentro del contenedor o en local |
