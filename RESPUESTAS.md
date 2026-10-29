# Respuestas · Laboratorio 4
Alumno: [20211041]
Presupuesto: [ 8 ] min  Métrica DORA: [ time-to-restore]

## Parte 1 · Diagnóstico

### Defecto 1
- Job y etapa: smoke, etapa deploy. Tiene needs: build-push, así que arranca al mismo nivel que `deploy` en vez de después (post-deploy).
- Problema que causa en el pipeline: smoke corre en paralelo con deploy y llama a la Function URL mientras la Lambda todavía sirve la versión anterior, así que pasa en verde sin haber probado el código nuevo (deja pasar un despliegue roto o que nunca ocurrió). e2e y load dependen de smoke y heredan el problema
- Link al commit: ⟦LINK_COMMIT_1⟧
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): mover de etapa (needs: deploy) y ajustar configuración (EXPECTED_COMMIT: ${{ github.sha }} en el step Smoke).
- Evidencia (dato observado, con link al run o fragmento de la salida local):
  - Run base: ⟦LINK_RUN_BASE⟧
  - Antes: ⟦pegar filas de `smoke` y `deploy` de `gh run view <RUN_ID> --json jobs --jq '.jobs[] | [.name,.startedAt,.completedAt] | @tsv'`: smoke empieza antes de que deploy termine⟧
  - Antes: ⟦pegar la línea `salud: {...}` del log de smoke con su `commit`, comparado con el SHA del run; y que no aparece la línea `commit:`⟧
  - Después: ⟦link al run corregido, con smoke empezando después de deploy y la línea `commit: <sha>`⟧

### Defecto 2
- Job y etapa: `sca`, etapa pre-build. Step `Trivy`, flag `--include-dev-deps`.
- Problema que causa en el pipeline: Trivy escanea también `@playwright/test`, `c8` y `esbuild`, que no van en la imagen. Con `--exit-code 1`, un HIGH o CRITICAL en esos paquetes detiene el build por algo que nunca se despliega (prueba lo incorrecto). Lo que sí se despliega lo cubre el escaneo de la imagen.
- Link al commit: ⟦LINK_COMMIT_2⟧
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): ajustar configuración (quitar `--include-dev-deps`).
- Evidencia (dato observado, con link al run o fragmento de la salida local):
  - ⟦pegar la salida de `docker run --rm -v "$PWD:/src" -w /src ghcr.io/aquasecurity/trivy:0.76.1 fs --scanners vuln --list-all-pkgs --format json --include-dev-deps package-lock.json | jq '[.Results[].Packages[]?] | length'`⟧
  - ⟦pegar la misma salida sin `--include-dev-deps`; la diferencia son los paquetes de desarrollo escaneados de más⟧

### Defecto 3
- Job y etapa: `load`, etapa post-deploy. No tiene condición `if`.
- Problema que causa en el pipeline: ejecuta unos 12 minutos de carga (1m + 10m + 1m, 30 VUs) en cada push de cada pull request. Desperdicia tiempo, porque la duración del pipeline es la suma de los jobs, y genera tráfico real sobre la función compartida donde se despliegan todos los PR.
- Link al commit: ⟦LINK_COMMIT_3⟧
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): mover de etapa o evento (`if: github.event_name != 'pull_request'`; sigue corriendo en `push` a `main` y en `workflow_dispatch`).
- Evidencia (dato observado, con link al run o fragmento de la salida local):
  - ⟦duración del job `load` en el run base: link y valor, frente a la de los demás jobs⟧
  - ⟦después: link al run de PR donde `load` aparece como *skipped*⟧

### Defecto 4
- Job y etapa: nivel del workflow, afecta a `smoke`, `e2e` y `load` (post-deploy). La `concurrency` existe solo dentro de `deploy`.
- Problema que causa en el pipeline: todos los PR despliegan en la misma función. `deploy` se serializa, pero lo que viene después no, así que el run B puede reemplazar la versión mientras el run A la está probando. Un run da verde o rojo por una versión que no es la suya (prueba de forma incorrecta).
- Link al commit: ⟦LINK_COMMIT_4⟧
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): ajustar configuración (`concurrency` a nivel de workflow, `group: ci-${{ github.repository }}`, `cancel-in-progress: false`).
- Evidencia (dato observado, con link al run o fragmento de la salida local):
  - ⟦pegar `gh run list --workflow ci --json databaseId,status,createdAt` con dos runs `in_progress` a la vez, o el mensaje `/version devuelve X, se esperaba Y` del smoke de uno de ellos, con el link al run⟧

### Defecto 5
- Job y etapa: `build-push`, etapa build. Step `Escaneo de la imagen (informativo)`.
- Problema que causa en el pipeline: es lo único que revisa lo que realmente se despliega (sistema operativo, runtime y dependencias de la imagen), y no detiene nada. Una imagen con HIGH o CRITICAL corregibles se sube a ECR y llega a `deploy`, que no se puede modificar. Deja pasar vulnerabilidades.
- Link al commit: ⟦LINK_COMMIT_5⟧
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): ajustar configuración (falla si hay HIGH o CRITICAL con versión corregida disponible, conservando la tabla del resumen).
- Evidencia (dato observado, con link al run o fragmento de la salida local):
  - ⟦pegar la anotación `N hallazgos HIGH o CRITICAL` y la tabla "Escaneo de la imagen" del resumen del run base, con el link; y que el run siguió a `deploy`⟧
  - ⟦después: link al run con el step en rojo o en verde, según el caso⟧

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
