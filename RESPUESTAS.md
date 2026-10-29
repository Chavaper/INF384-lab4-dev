# Respuestas · Laboratorio 4
Alumno: 20203648
Presupuesto: [11] min  Métrica DORA: [Change failure rate]

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
https://github.com/Chavaper/INF384-lab4-dev/commit/f987c82 <br>

- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests):<br> 
ajustar configuración → quitar `--include-dev-deps` y agregar `--ignore-unfixed` en el gate; si se quiere visibilidad de las dev-deps, un segundo escaneo informativo (`--exit-code 0`) o en un evento programado. Generar el SBOM (`--format cyclonedx`) como artefacto.<br>

- Evidencia (dato observado, con link al run o fragmento de la salida local):<br> 
    - `ci.yml:82` → `--include-dev-deps \`; 
    - `Dockerfile:8` → `RUN npm run build && npm prune --omit=dev`; 
    - `package.json` declara `@playwright/test`, `c8` y `esbuild` solo en `devDependencies`. `npm audit` local reporta `found 0 vulnerabilities`, por lo que el defecto es latente: se manifiesta el día que salga una CVE en una dev-dependency.<br>

### Defecto 4
- Job y etapa:<br> 
`build-push`, paso "Escaneo de la imagen (informativo)" · etapa Publicar.<br>

- Problema que causa en el pipeline:<br> 
el análisis de la imagen corre después del build pero **nunca detiene**: solo escribe un resumen y un `::warning`, y el paso siguiente hace `docker push` igual. Una imagen con hallazgos HIGH/CRITICAL de la base (`public.ecr.aws/lambda/nodejs:20`, sin fijar por digest) o del runtime llega al registro y se despliega. Las capas que no escribió el equipo (SO, runtime, base) quedan sin gate, y no se genera SBOM ni firma asociados al digest.<br>

- Link al commit:<br> 
https://github.com/Chavaper/INF384-lab4-dev/commit/23b5bf9 <br>

- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests):<br> 
ajustar configuración → convertirlo en gate antes del push: `--severity HIGH,CRITICAL --ignore-unfixed --exit-code 1`, de modo que si falla no exista imagen en ECR; generar el SBOM de la imagen junto al digest y fijar la imagen base por digest. <br>

- Evidencia (dato observado, con link al run o fragmento de la salida local):<br> 
    - `ci.yml:130` → `name: Escaneo de la imagen (informativo)`; 
    - `ci.yml:152` → `echo "::warning ...::${altos} hallazgos HIGH o CRITICAL. Informativo, no detiene."`; 
    - `ci.yml:154-157` → `docker push` incondicional a continuación.


### Defecto 5
- Job y etapa: <br>
`integration` · etapa Pruebas (CI).<br>

- Problema que causa en el pipeline:<br>
los tests de integración no son deterministas ni repetibles. Usan una base SQLite persistente en una ruta fija (`DB_PATH: /tmp/wallet.db`, la misma que el valor por defecto de `src/db.js`), no la crean ni la limpian, y cada test depende del estado que dejó el anterior con saldos absolutos (900, 600, 750, 1500). Pasan solo en la primera ejecución sobre una base vacía; cualquier reejecución en un runner reutilizado, en local o con otro orden de ejecución falla, generando fallos que no corresponden a un defecto del código y quitándole confianza al gate.<br>

- Link al commit:<br> 
https://github.com/Chavaper/INF384-lab4-dev/commit/44b0640 <br>

- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests):<br> 
modificar tests + ajustar configuración → crear una base temporal única por ejecución (p. ej. `fs.mkdtempSync` en un `before()` que fije `process.env.DB_PATH`, o `DB_PATH: ${{ runner.temp }}/wallet-${{ github.run_id }}.db`), borrarla en `after()`, y que cada test prepare su propio estado o compare saldos relativos al inicial.<br>

- Evidencia (dato observado, con link al run o fragmento de la salida local):<br> 
ejecución local (`node --test test/integration/transferir.test.js`) varias veces seguidas con la misma `DB_PATH`:
  ```
  === corrida 1
  # pass 12
  # fail 0
  === corrida 2
  not ok 1 - ana transfiere 100 a beto
  not ok 2 - el saldo de ana refleja el debito
  not ok 3 - el saldo de beto refleja el credito
  not ok 4 - el historial de ana registra la transferencia
  not ok 6 - una transferencia rechazada no cambia saldos
  not ok 7 - diego transfiere todo su saldo a elena
  # pass 6
  # fail 6
  === corrida 3
  not ok 1 - ana transfiere 100 a beto
      700 !== 900
  ```
  El saldo de ana se acumula entre corridas (1000 → 900 → 800 → 700) porque la base no se reinicia.



## Parte 2 · Implementación
- Link al run en verde en main:<br> 
https://github.com/Chavaper/INF384-lab4-dev/actions/runs/40964652993
- Link al pull request: https://github.com/Chavaper/INF384-lab4-dev/pull/1
- Defecto de la aplicación introducido (archivo y descripción):<br>
En wallet, condición if en (línea 54): antes `(cuentaOrigen.saldo < centimos) `, ahora `(cuentaOrigen.saldo <= centimos) `. Si un usuario Diego quiere transferir 100 soles (10000 céntimos) a Elena, y Diego solo tiene como saldo 100 soles (10000 céntimos), se supone que debe poder. Antes, la condición daba: saldo (10000 céntimos) < centimos (10000 céntimos) => falso -> se transfiere. Ahora:  saldo (10000 céntimos) < centimos (10000 céntimos) => falso -> lanza ERROR.<br>
- Job que debería detenerlo:<br> unit y integration
- Link al run fallido:<br> https://github.com/Chavaper/INF384-lab4-dev/actions/runs/40966922775

## Parte 3 · Decisión bajo restricción
Link al run medido:

| Job | Duración (min:seg) |
|---|---|
| unit | 15s |
| integration | 15s |
| sast | 21s |
| sca | 17s |
| build-push | 38s |
| deploy | 18s |
| smoke | 6s |
| e2e | 1m 1s |
| load | 12m 10s |
| **Total** | 13m 29s |

Resultado de load en ese run:
| Métrica | Valor |
|---|---|
| p90 | 68.04 ms |
| p95 | 70.03 ms |
| Máximo | 624.83 ms |
| Tasa de error | 0.00 % (0 de 35 162) |
| Requests por segundo | 48.80 req/s |

Conclusión sobre el comportamiento bajo carga:
Con 30 usuarios virtuales sostenidos durante 10 minutos (~48.8 req/s, 35 162 requests entre `POST /transferir` y `GET /saldo/{usuario}`), Monedero Andino se mantuvo estable y sin errores: 0.00 % de `http_req_failed` y 100 % de checks. La latencia es baja y muy concentrada (mín. 55.8 ms, mediana 63.6 ms, p90 68.04 ms, p95 70.03 ms): entre la mediana y el p95 hay menos de 7 ms, lo que indica que el servicio no se degrada a medida que se sostiene la carga, y el p95 queda unas 7 veces por debajo del umbral `SLO_P95` de 500 ms. El único valor fuera de ese patrón es el máximo de 624.83 ms, un caso aislado (no mueve el p95) que es consistente con un arranque en frío de una nueva instancia de Lambda al escalar; es la cola que el percentil deja fuera y que sí sentiría un usuario puntual.

SLO propuesto (percentil, valor, ventana) y justificación:
- percentil: p95 de `http_req_duration` de los requests a la API (`/transferir`, `/saldo`, `/historial`).
- valor: < 200 ms. Configurado en `ci.yml` como `SLO_P95: 200`.
- ventana: 30 días móviles en producción, sobre todos los requests a la Function URL. En el pipeline, el job `load` lo verifica como compuerta temprana sobre la ventana de la prueba (la duración del escenario k6).
- justificación:
    - Umbral de 200 ms: El p95 observado fue 70.03 ms, por lo que 200 ms ofrece un margen de aproximadamente 2.9 veces.
    - Por qué no 80–100 ms: Solo existe un run de referencia y la latencia varía por factores externos, como la ubicación del runner y los arranques en frío de Lambda. Un umbral menor podría generar falsos positivos.
    - Por qué no 500 ms: Es demasiado permisivo (7 veces lo medido) y permitiría que regresiones importantes pasen desapercibidas. Con 200 ms se detectan degradaciones cercanas a 3 veces.



SLA propuesto y diferencia con el SLO:
- SLA propuesto: p95 de latencia menor a 500 ms y disponibilidad mínima de 99.5 % mensual, con créditos de servicio si se incumple. Se excluyen mantenimientos programados y caídas de AWS.
- Diferencia con el SLO: El SLO es interno para el team, y suele ser mayor que el SLA para poder cumplirlo sin problemas.

Cambios para cumplir el presupuesto:
| Cambio | Job | Minutos recuperados |
|---|---|---|
| Carga constante de `test/load/transferir.js` de 10 min a 5 min (≥ 4 min exigidos), con el mismo objetivo de 30 usuarios y el umbral `p(95)<SLO_P95` (200 ms) | load | 5 |
| Rampa de subida de 1 min a 30 s | load | 0.5 |
| Rampa de bajada de 1 min a 30 s | load | 0.5 |

Total resultante: 7m 37s
Link al run en main con los cambios: https://github.com/Chavaper/INF384-lab4-dev/actions/runs/40972314232

Riesgo aceptado (defecto de la aplicación concreto y efecto en la métrica DORA asignada):<br>
Al acortar la fase sostenida de `load` (de 10 min a ~5 min), deja de detectarse una degradación que solo aparece con carga prolongada. Ejemplo concreto: un cambio en `src/handler.js` que guarde cada respuesta de `/saldo` en un objeto global sin límite (fuga de memoria); la latencia sube recién después de varios minutos, cuando la instancia de Lambda se queda sin memoria, por lo que el p95 de una prueba corta sigue bajo 200 ms y el cambio llega a producción. Efecto en Change failure rate: ese despliegue cuenta como fallido (degradación en producción que obliga a revertir), así que la tasa sube; se mitiga corriendo el escenario largo de forma programada fuera del pipeline de cada push.<br>

Decisión sobre sast (se queda / se mueve a otra etapa o evento / informativo) y razón:<br>
Se queda como gate en CI. Dura 21 s y corre en paralelo con `unit`, `integration` y `sca`, por lo que sacarlo no recupera tiempo del camino crítico (que domina `load`). Es rápido, determinista, acotado a `src/` y con `--severity ERROR`, que son las condiciones para que un control detenga en CI; moverlo después del deploy dejaría llegar a producción una inyección en una API pública que maneja dinero.

## Declaración de uso de IA
Herramientas usadas y para qué:

- Claude: para la prueba del defecto 5
