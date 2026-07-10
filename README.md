# TechMarket Orders: EFT AUY1104 (Ciclo de Vida del Software II)

Microservicio **TechMarket Orders** con pipeline CI/CD robustecido para el Examen Final Transversal: plantillas reutilizables de GitHub Actions, despliegue **Blue-Green** sobre Kubernetes (K3s en EC2, AWS Learner Lab) con validación de salud previa al cambio de tráfico y **rollback automático** ante fallos ("Ingeniería del Caos").

> Nota de infraestructura: el encargo referencia Amazon EKS y Amazon ECR. Según la precisión técnica del docente, se utilizan sus equivalentes trabajados durante el semestre: **K3s sobre EC2** (misma API estándar de Kubernetes) y **Docker Hub** como registry.

## 1. Arquitectura del despliegue

```
                      GitHub (push tag v*)
                              │
              ┌───────────────▼────────────────┐
              │ client.yaml (este repositorio) │
              └───────────────┬────────────────┘
                              │ uses
              ┌───────────────▼─────────────────┐
              │ deploy-api.yaml (SharedWorkflows)│
              │  Build:                          │
              │   · npm test (fail fast)         │
              │   · docker build + push DockerHub│
              │  Deploy Blue-Green:              │
              │   · detecta color activo         │
              │   · despliega candidato          │
              │   · valida salud (pre-switch)    │
              │   · conmuta tráfico              │
              │   · rollback automático          │
              └──────────────┬───────────────────┘
                                         │ SSH
                       ┌─────────────────▼──────────────────┐
                       │        EC2 + K3s (Learner Lab)     │
                       │                                    │
                       │  Deployment techmarket-orders-blue │
                       │  Deployment techmarket-orders-green│
                       │                                    │
                       │  Service techmarket-orders   :30090│ ← producción (selector color)
                       │  Service …-preview-blue      :30091│ ← validación blue
                       │  Service …-preview-green     :30092│ ← validación green
                       └────────────────────────────────────┘
```

- **Plantilla reutilizable** (`workflow_call`, diseñada en la EP1 y refactorizada para el EFT) en el repo central [`AUY1104-SharedWorkflows`](https://github.com/AUY1104-MDR/AUY1104-SharedWorkflows): `deploy-api.yaml`, que consolida las etapas de Build y Deploy Blue-Green en jobs encadenados (`deps-and-test` → `build-and-push` → `blue-green-deploy`).
- **Inyección de variables de entorno dinámicas al clúster:** el bloque `env-vars` de `client.yaml` define variables `CLAVE=VALOR` por release (versión, entorno, nombre); la plantilla de Deploy las aplica al Deployment candidato con `kubectl set env` antes del rollout. El endpoint `/health` las expone (`version`, `color`, `entorno`) como evidencia verificable.

## 2. Estrategia de despliegue: Blue-Green

Coexisten dos Deployments (`techmarket-orders-blue` y `techmarket-orders-green`). El Service de producción (`:30090`) selecciona pods por las etiquetas `app` **y** `color`: ese selector es el interruptor del tráfico.

Flujo de cada release (tag `v*`):

1. **Detección del color activo:** se lee `spec.selector.color` del Service de producción; el candidato se despliega en el color opuesto (primer despliegue: `blue`).
2. **Despliegue del candidato:** se renderiza `k8s/deployment.template.yml` (imagen y color) con `envsubst`, se aplica en el clúster y se inyectan las variables dinámicas; producción sigue sirviendo la versión estable, intacta.
3. **Validación de salud pre-switch:** se consulta `/health` del candidato por su Service de preview (`:30091`/`:30092`), con reintentos, verificando `"ok":true` **y** que el `color` del payload coincida con el candidato. El tráfico real aún no se ha movido.
4. **Switch de tráfico 100%:** solo si la validación pasa, se conmuta el selector del Service de producción con `kubectl patch` y se re-valida `/health` en `:30090`.
5. La versión anterior queda corriendo en su color, disponible para una reversión inmediata.

**Por qué Blue-Green (y no Canary):** el cambio de tráfico es determinista y atómico (un patch del selector), la validación se ejecuta contra el entorno candidato completo antes de exponer usuarios, y la remediación es instantánea porque la versión estable nunca deja de correr. Este comportamiento resulta adecuado frente a un error desconocido inyectado en vivo.

## 3. Remediación automática (rollback)

El paso `Rollback automático` de `deploy-api.yaml` se ejecuta con `if: failure()`, es decir, se activa solo si falló alguno de estos puntos de control:

| Punto de control | Falla detectada |
|---|---|
| `rollout status --timeout=120s` del candidato | Imagen rota, crash loop, probes que no pasan |
| Validación de salud pre-switch (preview) | `/health` no responde, responde error o responde el color equivocado |
| Validación post-switch (producción) | El servicio degradado tras la conmutación |

Acciones del rollback, en orden:

1. **Repone el selector** del Service de producción en el color estable (`kubectl patch`). Si el fallo fue pre-switch, el tráfico nunca llegó a moverse.
2. **Apaga el candidato** defectuoso (`kubectl scale --replicas=0`), dejando su Deployment disponible para diagnóstico.
3. El job termina en fallo, dejando registro visible en GitHub Actions.

Resultado: ante el error inyectado durante la "Prueba de Fuego", producción sigue (o vuelve a quedar) servida por la última versión sana sin intervención manual.

## 4. Estructura del repositorio

| Ruta | Propósito |
|---|---|
| `src/index.js` | Express app: `/health`, `/api/saludo`, `/api/echo`, `/api/suma`. |
| `src/lib/ejemplo.js` | Lógica pura; `/health` expone `version`, `color` y `entorno` inyectados por el pipeline. |
| `tests/*.test.js` + `test.js` | Suite Jest ejecutada por la plantilla de Build (fail fast). |
| `Dockerfile` | Imagen `node:20-alpine`, expone `3000`. |
| `k8s/deployment.template.yml` | Template Blue-Green (placeholders `${APP_NAME}`, `${COLOR}`, `${IMAGE}`). |
| `k8s/service.yml` | Service de producción `:30090` (selector `app`+`color`; se aplica solo la primera vez). |
| `k8s/service-preview-*.yml` | Services de preview `:30091` (blue) y `:30092` (green) para la validación. |
| `.github/workflows/client.yaml` | Pipeline de release: Build → Deploy Blue-Green (tags `v*`). |
| `.github/workflows/ci.yaml` | Validación en ramas (tests + build sin publicar). |

## 5. Configuración requerida

Secrets/variables en este repositorio (o en la organización):

- `secrets.DOCKER_USERNAME` / `secrets.DOCKER_PASSWORD`: publicación en Docker Hub.
- `secrets.EA2_SSH_PRIVATE_KEY`: acceso SSH al nodo K3s.
- `vars.K3S_SERVER_PUBLIC_IP`: IP pública de la EC2 con K3s (cambia en cada sesión del Learner Lab).

Imagen publicada: `docker.io/<DOCKER_USERNAME>/techmarket-orders:<tag>` (+ `latest`).

## 6. Operación

Publicar una nueva versión:

```bash
git tag v2.0.0
git push origin v2.0.0
```

Verificación manual post-despliegue:

```bash
curl -s http://<K3S_SERVER_PUBLIC_IP>:30090/health   # producción (color activo)
curl -s http://<K3S_SERVER_PUBLIC_IP>:30091/health   # preview blue
curl -s http://<K3S_SERVER_PUBLIC_IP>:30092/health   # preview green
```

Ejecución local (desarrollo):

```bash
npm install
npm test
npm start          # http://localhost:3000/health
```

Variables de entorno de la app:

| Variable | Valor por defecto | Uso |
|---|---|---|
| `PORT` | `3000` | Puerto de escucha |
| `APP_NAME` | `techmarket-orders` | Nombre reportado en `/health` |
| `APP_VERSION` | `dev` | Versión reportada en `/health` (inyectada por el pipeline) |
| `DEPLOY_COLOR` | `sin-color` | Color Blue-Green (inyectada por el manifiesto/pipeline) |
| `APP_ENVIRONMENT` | `local` | Entorno reportado en `/health` (inyectada por el pipeline) |
