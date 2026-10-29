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
  <br>

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
- Job y etapa:
- Problema que causa en el pipeline:
- Link al commit:
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests):
- Evidencia (dato observado, con link al run o fragmento de la salida local):
### Defecto 3
- Job y etapa:
- Problema que causa en el pipeline:
- Link al commit:
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests):
- Evidencia (dato observado, con link al run o fragmento de la salida local):
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
