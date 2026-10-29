# Respuestas · Laboratorio 4
Alumno: 20203648
Presupuesto: [ ] min  Métrica DORA: [ ]

## Parte 1 · Diagnóstico
### Defecto 1
- Job y etapa: <br>
`smoke` (y por arrastre `e2e` y `load`) · etapa Desplegar / verificación post-despliegue.

- Problema que causa en el pipeline:<br> 
`smoke` declara `needs: build-push` en lugar de `needs: deploy`, así que corre en paralelo con el despliegue. Como la Function URL ya existe desde la primera ejecución, el paso `URL` la obtiene de inmediato y `smoke.sh` golpea la versión **anterior** de la Lambda mientras `deploy` todavía hace `update-function-code`. Además el workflow no le pasa `EXPECTED_COMMIT`, por lo que el script se salta la verificación de `/version` contra `GITHUB_SHA`. Resultado: smoke, e2e y load pueden quedar en verde validando un artefacto que no es el que se acaba de construir (falso verde), y si `deploy` falla las pruebas post-despliegue igual se ejecutan.<br>

- Link al commit:<br> 
  https://github.com/Chavaper/INF384-lab4-dev/commit/62cdaf2<br>

- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests):<br> 
ajustar configuración → `smoke` con `needs: deploy` y en el paso `Smoke` agregar `EXPECTED_COMMIT: ${{ github.sha }}` para que el script espere hasta que `/version` devuelva el commit desplegado.<br>

- Evidencia (dato observado, con link al run o fragmento de la salida local):<br> 
Link al run: https://github.com/Chavaper/INF384-lab4-dev/actions/runs/40951759725
Se ve que `smoke` corre en paralelo que `deploy`.
Esto porque sucede lo siguiente:
    - `ci.yml: línea 233` → `needs: build-push` (igual que `deploy` en la línea 173, por lo que ambos arrancan a la vez); 
    - `ci.yml: líneas 259-262` define solo `BASE_URL`; 
    - `test/smoke/smoke.sh:20` → `if [ -n "${EXPECTED_COMMIT:-}" ]; then` (bloque de verificación de commit que nunca se ejecuta).


### Defecto 2
- Job y etapa: <br>
`build-push` → `deploy` · etapas Publicar / Desplegar, disparadas por el evento `pull_request`.<br>

- Problema que causa en el pipeline: <br>
el workflow se dispara en `pull_request` y ningún job de entrega tiene condición de rama/evento, así que **cada PR construye, publica en ECR y despliega a la única Lambda (producción)** con los secretos de AWS antes de ser revisado o mergeado. Un cambio no aprobado llega a la URL pública, pisa lo que hay en `main` y las pruebas post-despliegue de un PR y de `main` compiten por el mismo entorno (el `concurrency` de `deploy` solo los encola, no los separa).<br>

- Link al commit:<br> 
 https://github.com/Chavaper/INF384-lab4-dev/commit/ee937f2 <br>

- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests):
<br> ajustar configuración → como `deploy` no se modifica, condicionar `build-push` con `if: github.event_name != 'pull_request'` (o `github.ref == 'refs/heads/main'`); `deploy`, `smoke`, `e2e` y `load` quedan saltados en PR por dependencia y el PR solo corre `unit`, `integration`, `sast` y `sca`.<br>

- Evidencia (dato observado, con link al run o fragmento de la salida local):<br> 
    - `ci.yml:4` → `pull_request:` sin filtro de ramas; 
    - `ci.yml:88` → `build-push` solo tiene `needs: [unit, integration, sast, sca]`, sin `if:`.<br>

### Defecto 3
- Job y etapa:<br> 
`sca` · etapa Seguridad (CI, antes del build).<br>

- Problema que causa en el pipeline:<br> 
Trivy se ejecuta con `--include-dev-deps`, `--severity HIGH,CRITICAL` y `--exit-code 1`, sin `--ignore-unfixed`. El gate detiene el pipeline por vulnerabilidades en `@playwright/test`, `c8` y `esbuild`, que **no viajan en el artefacto** (el Dockerfile hace `npm prune --omit=dev`): hallazgos no alcanzables ni expuestos bloquean la entrega. Tampoco distingue si hay parche, así que una CVE sin corrección también detiene sin acción posible. Es un gate que prioriza por severidad y no por alcanzabilidad/exposición/parche, y además no genera SBOM.<br>

- Link al commit:<br> 
 <br>

- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests):<br> 
ajustar configuración → quitar `--include-dev-deps` y agregar `--ignore-unfixed` en el gate; si se quiere visibilidad de las dev-deps, un segundo escaneo informativo (`--exit-code 0`) o en un evento programado. Generar el SBOM (`--format cyclonedx`) como artefacto.<br>

- Evidencia (dato observado, con link al run o fragmento de la salida local):<br> 
    - `ci.yml:82` → `--include-dev-deps \`; 
    - `Dockerfile:8` → `RUN npm run build && npm prune --omit=dev`; 
    - `package.json` declara `@playwright/test`, `c8` y `esbuild` solo en `devDependencies`. `npm audit` local reporta `found 0 vulnerabilities`, por lo que el defecto es latente: se manifiesta el día que salga una CVE en una dev-dependency.<br>

### Defecto 4
- Job y etapa:
- Problema que causa en el pipeline:
- Link al commit:
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests):
- Evidencia (dato observado, con link al run o fragmento de la salida local):
### Defecto 5
- Job y etapa:
- Problema que causa en el pipeline:
- Link al commit:
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests):
- Evidencia (dato observado, con link al run o fragmento de la salida local):


## Parte 2 · Implementación
- Link al run en verde en main:
- Link al pull request:
- Defecto de la aplicación introducido (archivo y descripción):
- Job que debería detenerlo:
- Link al run fallido:

## Parte 3 · Decisión bajo restricción
Link al run medido:

| Job | Duración (min:seg) |
|---|---|
| unit | |
| integration | |
| sast | |
| sca | |
| build-push | |
| deploy | |
| smoke | |
| e2e | |
| load | |
| **Total** | |

Resultado de load en ese run:
| Métrica | Valor |
|---|---|
| p90 | |
| p95 | |
| Máximo | |
| Tasa de error | |
| Requests por segundo | |

Conclusión sobre el comportamiento bajo carga:

SLO propuesto (percentil, valor, ventana) y justificación:

SLA propuesto y diferencia con el SLO:

Cambios para cumplir el presupuesto:
| Cambio | Job | Minutos recuperados |
|---|---|---|

Total resultante: [ ] min
Link al run en main con los cambios:

Riesgo aceptado (defecto de la aplicación concreto y efecto en la métrica DORA asignada):

Decisión sobre sast (se queda / se mueve a otra etapa o evento / informativo) y razón:

## Declaración de uso de IA
Herramientas usadas y para qué:
