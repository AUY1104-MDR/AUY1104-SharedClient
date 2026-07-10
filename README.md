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

## 4. Desarrollo realizado (bitácora del EFT)

### 4.1 Punto de partida (estado al cierre del semestre)

| Componente | Estado previo al EFT |
|---|---|
| Pipeline de despliegue | `deploy-api.yaml` (SharedWorkflows): estrategia **Rolling**, con `kubectl apply` de toda la carpeta `k8s/` y `rollout status` con `rollout undo` ante fallo. |
| Riesgo del enfoque | El tráfico productivo alcanzaba la versión nueva **antes** de validar su salud; el rollback actuaba después de la exposición al usuario. |
| Manifiestos | Un único Deployment `demo-api` (1 réplica, probes `/health`) y un Service NodePort 30090. |
| Variables de entorno | Sin mecanismo de inyección dinámica hacia el clúster. |

### 4.2 Cambios implementados por ítem del encargo

| Ítem de la pauta | Implementación | Archivos |
|---|---|---|
| **Ítem 1** · Plantilla reutilizable de Build y Deploy | El workflow Rolling del semestre se refactorizó en la plantilla `workflow_call` consolidada del repo central, con las etapas de Build y Deploy como jobs encadenados, invocada desde este repo. | `SharedWorkflows: deploy-api.yaml`; `client.yaml` |
| **Ítem 1** · Variables de entorno dinámicas al clúster | Input `env-vars` (líneas `CLAVE=VALOR`) definido por release en `client.yaml`; la plantilla las aplica con `kubectl set env` al Deployment candidato antes del rollout. `/health` las expone como evidencia. | `client.yaml`, `deploy-api.yaml` (paso 6), `src/lib/ejemplo.js` |
| **Ítem 2** · Despliegue Blue-Green | Dos Deployments por color generados desde un template (`envsubst`); el Service de producción selecciona por `app`+`color` y el tráfico se conmuta manipulando ese selector con `kubectl patch` (nunca con `apply`, para no pisar el color activo). | `k8s/deployment.template.yml`, `k8s/service.yml`, `deploy-api.yaml` (pasos 2-5 y 9) |
| **Ítem 2** · Validación de salud antes del 100% | Services de preview por color (30091/30092) permiten consultar `/health` del candidato con reintentos, verificando `"ok":true` y que responde el color esperado, con el tráfico productivo intacto. | `k8s/service-preview-*.yml`, `deploy-api.yaml` (paso 8) |
| **Ítem 3** · Rollback automático | Paso `if: failure()` que repone el selector en el color estable y escala el candidato a 0 réplicas; cubre fallo de rollout, de validación pre-switch y post-switch. | `deploy-api.yaml` (paso Rollback) |
| Entregable · README técnico | Este documento: arquitectura, estrategia, remediación, bitácora y guía de demostración. | `README.md` |
| Calidad · CI en ramas | Validación de tests + build Docker sin publicar en cada push a ramas de trabajo. | `.github/workflows/ci.yaml` |

### 4.3 Decisiones técnicas registradas

1. **Repositorios del semestre, no repositorios nuevos:** la pauta exige entregar el repositorio trabajado durante el semestre con la evolución documentada en los commits; el EFT se implementó como nuevos commits sobre `AUY1104-SharedClient` y `AUY1104-SharedWorkflows`.
2. **Blue-Green sobre Canary:** switch de tráfico determinista y atómico (patch del selector), validación contra el entorno candidato completo antes de exponer usuarios y reversión instantánea, dado que la versión estable nunca deja de correr.
3. **Service de producción aplicado solo la primera vez:** re-aplicar `service.yml` en cada despliegue restauraría el selector `color: blue` del manifiesto y movería el tráfico de forma no controlada; tras el primer despliegue solo se gestiona vía `kubectl patch`.
4. **Candidato defectuoso escalado a 0 (no eliminado):** su Deployment queda disponible para diagnóstico posterior al rollback.
5. **K3s + Docker Hub en lugar de EKS + ECR:** conforme a la precisión técnica del docente; K3s implementa la misma API estándar de Kubernetes.
6. **`deploy-api.yaml` se conserva** en el repo central como evidencia de la evolución del pipeline (Rolling → Blue-Green).

### 4.4 Guía de demostración ("Prueba de Fuego")

1. **Provisionar el clúster:** ejecutar `ea2-lab-dispatch-main.yml` en `SharedWorkflows` (Terraform → EC2 + K3s) y registrar la IP pública en `vars.K3S_SERVER_PUBLIC_IP` de este repo.
2. **Despliegue inicial:** publicar un tag `v*`; el pipeline construye la imagen y despliega el color `blue`, aplica el Service de producción y valida salud. Verificar con `curl http://<IP>:30090/health` → responde `"color":"blue"`.
3. **Release sana:** publicar un nuevo tag; el pipeline despliega `green`, lo valida por `:30092` y conmuta el tráfico. `:30090/health` pasa a responder `"color":"green"` con la nueva `version`.
4. **Release con error (simulación del caos):** publicar un tag con un fallo (por ejemplo, `/health` retornando 500 o una imagen que no arranca). El pipeline despliega el candidato, la validación falla, el selector **no** se conmuta (o se repone), el candidato se escala a 0 y el job termina en fallo visible en Actions. `:30090/health` sigue respondiendo la versión estable durante todo el proceso.
5. **Evidencias:** logs del run en GitHub Actions (pasos 8-10 y Rollback), `kubectl get deploy,svc,pods -l app=techmarket-orders` del paso final, y salidas de `curl` antes/durante/después.

## 5. Estructura del repositorio

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

## 6. Configuración requerida

Secrets/variables en este repositorio (o en la organización):

- `secrets.DOCKER_USERNAME` / `secrets.DOCKER_PASSWORD`: publicación en Docker Hub.
- `secrets.EA2_SSH_PRIVATE_KEY`: acceso SSH al nodo K3s.
- `vars.K3S_SERVER_PUBLIC_IP`: IP pública de la EC2 con K3s (cambia en cada sesión del Learner Lab).

Imagen publicada: `docker.io/<DOCKER_USERNAME>/techmarket-orders:<tag>` (+ `latest`).

## 7. Operación

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

## 8. Declaración de uso de IA

Durante el desarrollo de esta evaluación se utilizó una herramienta de inteligencia artificial generativa como apoyo para la revisión de la estructura de los workflows y la redacción de documentación técnica. Todas las decisiones de diseño (estrategia Blue-Green, puntos de control de salud y mecanismo de rollback), la configuración de la infraestructura y la validación del funcionamiento fueron realizadas y verificadas por el estudiante.

## 9. Referencias

- Kubernetes. (2025). *Deployments*. https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- GitHub. (2025). *Reusing workflows*. https://docs.github.com/en/actions/using-workflows/reusing-workflows
- Humble, J., & Farley, D. (2010). *Continuous Delivery: Reliable Software Releases through Build, Test, and Deployment Automation*. Addison-Wesley.
- Fowler, M. (2010). *BlueGreenDeployment*. https://martinfowler.com/bliki/BlueGreenDeployment.html
