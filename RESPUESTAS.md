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
- Link al commit: (https://github.com/D4rkPatata/INF384-lab4-20211041/commit/69d625d5313a891fbca834558928d530d146b579)
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): ajustar configuración (quitar `--include-dev-deps`).
- Evidencia (dato observado, con link al run o fragmento de la salida local):
  - Antes: 2026-10-29T22:04:48Z    INFO    [npm] Detecting vulnerabilities...
            122
  - Despues: 2026-10-29T22:06:01Z    INFO    [npm] Detecting vulnerabilities...
            38

### Defecto 3
- Job y etapa: `load`, etapa post-deploy. No tiene condición `if`.
- Problema que causa en el pipeline: ejecuta unos 12 minutos de carga (1m + 10m + 1m, 30 VUs) en cada push de cada pull request. Desperdicia tiempo, porque la duración del pipeline es la suma de los jobs, y genera tráfico real sobre la función compartida donde se despliegan todos los PR.
- Link al commit: https://github.com/D4rkPatata/INF384-lab4-20211041/commit/5c507a7280ee979592641950a7f399f7f629ef53
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): mover de etapa o evento (`if: github.event_name != 'pull_request'`; sigue corriendo en `push` a `main` y en `workflow_dispatch`).
- Evidencia (dato observado, con link al run o fragmento de la salida local):
  - Antes el job load duraba 12 minutos
  - Ahora dura 

### Defecto 4
- Job y etapa: nivel del workflow, afecta a `smoke`, `e2e` y `load` (post-deploy). La `concurrency` existe solo dentro de `deploy`.
- Problema que causa en el pipeline: todos los PR despliegan en la misma función. `deploy` se serializa, pero lo que viene después no, así que el run B puede reemplazar la versión mientras el run A la está probando. Un run da verde o rojo por una versión que no es la suya (prueba de forma incorrecta).
- Link al commit: (https://github.com/D4rkPatata/INF384-lab4-20211041/commit/5886d6150fe85cc445d4e9ccea1512f738ab62f2)
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): ajustar configuración (`concurrency` a nivel de workflow, `group: ci-${{ github.repository }}`, `cancel-in-progress: false`).
- Evidencia (dato observado, con link al run o fragmento de la salida local):
  - ⟦pegar `gh run list --workflow ci --json databaseId,status,createdAt` con dos runs `in_progress` a la vez, o el mensaje `/version devuelve X, se esperaba Y` del smoke de uno de ellos, con el link al run⟧

### Defecto 5
- Job y etapa: `unit` e `integration`, etapa pre-build. Scripts `test:unit` y `test:integration` de `package.json`.
- Problema que causa en el pipeline: el gate de cobertura de `unit` (`--include src/wallet.js --include src/util.js`, umbral 95%) mide solo 2 de los 4 archivos de `src/`. `handler.js` y `db.js` no se miden en ningún job, porque `integration` corre sin c8. Medidos localmente, `handler.js` tiene 78,57% de líneas y 66,66% de ramas. Ese código puede perder cobertura sin que el pipeline lo detecte (deja pasar).
- Link al commit: (https://github.com/D4rkPatata/INF384-lab4-20211041/commit/7978e3a2a37a1f6f426ff416e71e730373cc42cc)
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
- Link al run en verde en main: https://github.com/D4rkPatata/INF384-lab4-20211041/actions/runs/40964082039
- Link al pull request: https://github.com/D4rkPatata/INF384-lab4-20211041/pull/1
- Defecto de la aplicación introducido (archivo y descripción): en util.js se cambio:
        if (monto =< 0) {
            return 'monto debe ser mayor que cero';
        }
        ->
        if (monto < 0) {
            return 'monto debe ser mayor que cero';
        }
- Job que debería detenerlo: unit. el defecto fallan en validarMonto rechaza cero (util.test.js) y transferir con monto invalido es 400 (wallet.test.js), y ambos están en test/unit/.
- Link al run fallido: https://github.com/D4rkPatata/INF384-lab4-20211041/actions/runs/40965804074

Alumno: [20211041]
Presupuesto: [ 8 ] min  Métrica DORA: [ time-to-restore]

## Parte 3 · Decisión bajo restricción
Link al run medido: ⟦LINK_RUN_PARTE_2⟧

| Job | Duración (min:seg) |
|---|---|
| unit | 0:10 |
| integration | 0:12 |
| sast | 0:21 |
| sca | 0:21 |
| build-push | 0:50 |
| deploy | 0:18 |
| smoke | 0:07 |
| e2e | 0:36 |
| load | 12:10 |
| **Total** | **15:05** |

Resultado de load en ese run:
| Métrica | Valor |
|---|---|
| p90 | 17,65 ms |
| p95 | 20,07 ms |
| Máximo | 202,28 ms |
| Tasa de error | 0,00 % (0 de 38.568 requests) |
| Requests por segundo | 53,56 req/s |

Conclusión sobre el comportamiento bajo carga:
Con 30 usuarios virtuales, Monedero Andino fue rápido y estable. La mitad de los requests respondió en 13,53 ms o menos, el p95 fue 20,07 ms y no falló ninguno de los 38.568 requests (los 400 cuentan como esperados en el script). Los tres umbrales pasaron. Los tiempos fueron muy parejos (p90 17,65 ms, p95 20,07 ms) y el máximo de 202,28 ms es un pico aislado. Esta prueba no busca el límite del sistema: solo muestra que, con esta carga, hay mucha holgura. El umbral original de 500 ms era unas 25 veces lo medido, así que no habría detectado un problema real.

SLO propuesto (percentil, valor, ventana) y justificación:
p95 < 100 ms, medido en la prueba de k6 de cada run en `main` (4 minutos de carga constante, unos 14.000 requests). Se configura como `SLO_P95: 100` en `ci.yml`. En los dos runs medidos el p95 fue 20,07 ms y 33,57 ms. 100 ms es unas 3 veces el valor más alto: deja margen para la variación normal entre runs y aun así detecta una degradación seria. Un valor muy cerca de lo medido (30 a 40 ms) fallaría por simple ruido, y 500 ms casi no detectaría nada.

SLA propuesto y diferencia con el SLO:
SLA: 99,9 % de disponibilidad al mes y p95 < 500 ms en producción, con una compensación (crédito) al cliente si no se cumple. El SLO es una meta interna sin castigo; el SLA es una promesa al cliente con consecuencias. El SLA es más flojo (500 ms frente a 100 ms) para que el equipo reciba la alerta con tiempo antes de incumplir lo prometido, y porque en producción hay factores que no controla (la red del cliente, picos de tráfico, arranques en frío).

Cambios para cumplir el presupuesto:
| Cambio | Job | Minutos recuperados |
|---|---|---|
| Stages de k6: de 1m + 10m + 1m a 10s de subida + 4m de carga constante (sin rampa de bajada) | load | 7:52 (de 12:10 a 4:18) |

Total resultante: 7:01 min
Link al run en main con los cambios: https://github.com/D4rkPatata/INF384-lab4-20211041/actions/runs/40967645943

Riesgo aceptado (defecto de la aplicación concreto y efecto en la métrica DORA asignada):
Una fuga de conexiones SQLite en `db.js` (cada request abre una conexión y no la cierra). Solo se nota después de muchos requests. La prueba larga hacía 38.568 requests y la nueva hace 13.934. Si el problema aparece entre unos 14.000 y 38.000 requests, antes lo detectaba `load` y ahora se queda en producción sin detectarse. Para este defecto el time-to-restore sube (empeora), porque lo descubren los clientes o el monitoreo y no el pipeline. En general el time-to-restore baja: un arreglo urgente pasa de esperar unos 13:46 en el pipeline a unos 5:38. Acepto el riesgo porque es un defecto poco frecuente y Lambda recicla sus instancias, lo que limita el daño.

Decisión sobre sast (se queda / se mueve a otra etapa o evento / informativo) y razón:
Se queda, en pre-build y en todos los eventos. Cuesta 23 s en la suma de jobs, pero en tiempo real solo agrega 5 s, porque corre en paralelo con `sca` (18 s), `unit` e `integration`. Quitarlo ahorraría 5 s de 5:38, casi nada para el time-to-restore. Moverlo solo a pull requests dejaría sin revisar un push directo a `main`, y pasarlo a informativo no ahorra minutos, porque el job se ejecuta igual. Mantenerlo evita que código inseguro llegue a producción, y así hay menos incidentes que restaurar. Efecto en time-to-restore: prácticamente nulo. Costo: 23 s dentro de un total de 7:01.

## Declaración de uso de IA
Herramientas usadas y para qué: Claude (Anthropic), para analizar el pipeline, identificar los defectos y estructurar RESPUESTAS.md. Los comandos, los runs y las mediciones los ejecuté y verifiqué yo.