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
  - Antes: Run test/smoke/smoke.sh
        salud: {"ok":true,"version":"1.0.0","commit":"49ba49ebee84f4c9f8b6943031595970311a3bb6"}
        saldo: {"usuario":"ana","saldo":1000}
  - Después: Run test/smoke/smoke.sh
        salud: {"ok":true,"version":"1.0.0","commit":"3afcf155391c88b02ff017637276e429b42539fd"}
        commit: 3afcf155391c88b02ff017637276e429b42539fd
        saldo: {"usuario":"ana","saldo":1000}

### Defecto 2
- Job y etapa: `sca`, etapa pre-build. Step `Trivy`, flag `--include-dev-deps`.
- Problema que causa en el pipeline: Trivy escanea también `@playwright/test`, `c8` y `esbuild`, que no van en la imagen. Con `--exit-code 1`, un HIGH o CRITICAL en esos paquetes detiene el build por algo que nunca se despliega (prueba lo incorrecto). Lo que sí se despliega lo cubre el escaneo de la imagen.
- Link al commit: ⟦LINK_COMMIT_2⟧
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): ajustar configuración (quitar `--include-dev-deps`).
- Evidencia (dato observado, con link al run o fragmento de la salida local):
  - Antes: 2026-10-29T22:04:48Z    INFO    [npm] Detecting vulnerabilities...
            122
  - Despues: 2026-10-29T22:06:01Z    INFO    [npm] Detecting vulnerabilities...
            38

### Defecto 3
- Job y etapa: `load`, etapa post-deploy. No tiene condición `if`.
- Problema que causa en el pipeline: ejecuta unos 12 minutos de carga (1m + 10m + 1m, 30 VUs) en cada push de cada pull request. Desperdicia tiempo, porque la duración del pipeline es la suma de los jobs, y genera tráfico real sobre la función compartida donde se despliegan todos los PR.
- Link al commit: ⟦LINK_COMMIT_3⟧
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): mover de etapa o evento (`if: github.event_name != 'pull_request'`; sigue corriendo en `push` a `main` y en `workflow_dispatch`).
- Evidencia (dato observado, con link al run o fragmento de la salida local):
  - Antes el job load duraba 12 minutos
  - Ahora dura 

### Defecto 4
- Job y etapa: nivel del workflow, afecta a `smoke`, `e2e` y `load` (post-deploy). La `concurrency` existe solo dentro de `deploy`.
- Problema que causa en el pipeline: todos los PR despliegan en la misma función. `deploy` se serializa, pero lo que viene después no, así que el run B puede reemplazar la versión mientras el run A la está probando. Un run da verde o rojo por una versión que no es la suya (prueba de forma incorrecta).
- Link al commit: ⟦LINK_COMMIT_4⟧
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): ajustar configuración (`concurrency` a nivel de workflow, `group: ci-${{ github.repository }}`, `cancel-in-progress: false`).
- Evidencia (dato observado, con link al run o fragmento de la salida local):
  - ⟦pegar `gh run list --workflow ci --json databaseId,status,createdAt` con dos runs `in_progress` a la vez, o el mensaje `/version devuelve X, se esperaba Y` del smoke de uno de ellos, con el link al run⟧

### Defecto 5
- Job y etapa: `unit` e `integration`, etapa pre-build. Scripts `test:unit` y `test:integration` de `package.json`.
- Problema que causa en el pipeline: el gate de cobertura de `unit` (`--include src/wallet.js --include src/util.js`, umbral 95%) mide solo 2 de los 4 archivos de `src/`. `handler.js` y `db.js` no se miden en ningún job, porque `integration` corre sin c8. Medidos localmente, `handler.js` tiene 78,57% de líneas y 66,66% de ramas. Ese código puede perder cobertura sin que el pipeline lo detecte (deja pasar).
- Link al commit: ⟦LINK_COMMIT_5⟧
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): ajustar configuración (c8 con `--check-coverage --lines 85 --branches 70` en `test:integration`).
- Evidencia (dato observado, con link al run o fragmento de la salida local):
  - Antes: ------------|---------|----------|---------|---------|----------------------------------------
            File        | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s                      
            ------------|---------|----------|---------|---------|----------------------------------------
            All files   |   86.39 |    69.04 |     100 |   86.39 |                                        
            db.js      |    93.5 |       75 |     100 |    93.5 | 33-35,69-70                            
            handler.js |   78.57 |    66.66 |     100 |   78.57 | 10-11,17-18,26-27,32-34,36-37,53,65-67 
            ------------|---------|----------|---------|---------|----------------------------------------
  - Después: ------------|---------|----------|---------|---------|-------------------
                File        | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s 
                ------------|---------|----------|---------|---------|-------------------
                All files   |   47.12 |    95.34 |    87.5 |   47.12 |                   
                db.js      |       0 |        0 |       0 |       0 | 1-77              
                handler.js |       0 |        0 |       0 |       0 | 1-70              
                util.js    |     100 |      100 |     100 |     100 |                   
                wallet.js  |     100 |      100 |     100 |     100 |                   
                ------------|---------|----------|---------|---------|-------------------
            

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
