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

## Parte 3 · Decisión bajo restricción
Link al run medido:

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
| Requests por segundo | 53,56 req/s (26,78 iteraciones/s) |

Conclusión sobre el comportamiento bajo carga:
Con 30 usuarios virtuales, Monedero Andino respondió de forma estable: mediana de 13,53 ms, p95 de 20,07 ms, ningún request fallido (los 400 cuentan como esperados en este script) y los tres thresholds en verde. La cola es corta (p90 17,65 ms frente a p95 20,07 ms). El máximo de 202,28 ms es unas 10 veces el p95 y parece un pico aislado, quizá una instancia nueva de Lambda, que no afecta al p95. La carga la fija el diseño de la prueba (30 VUs con `sleep(1)`), así que no se encontró el límite de capacidad: se ve que hay holgura, no cuánta. El umbral actual de 500 ms es unas 25 veces lo medido y no detectaría una degradación real.

SLO propuesto (percentil, valor, ventana) y justificación:
p95 < 100 ms, medido sobre la prueba de k6 de cada run en `main` . Se configura como `SLO_P95: 100` en `ci.yml`. Es 5 veces el p95 medido (20,07 ms): deja margen para la variación normal entre runs y a la vez detecta una regresión seria. Un valor cercano a lo medido (25 a 30 ms) fallaría por ruido, y 500 ms toleraría una degradación de 25 veces.

SLA propuesto y diferencia con el SLO:
SLA: 99,9 % de disponibilidad mensual y p95 < 500 ms medido en producción durante el mes calendario, con crédito de servicio al cliente si no se cumple. El SLO es una meta interna sin penalidad. El SLA es un compromiso contractual con consecuencias. Por eso es más holgado: el equipo recibe la alerta del SLO (100 ms) mucho antes de incumplir el SLA (500 ms), y el SLA debe cubrir lo que el equipo no controla (red del cliente, arranques en frío, picos de tráfico real).

Cambios para cumplir el presupuesto:
| Cambio | Job | Minutos recuperados |
|---|---|---|
| Stages de k6 de 1m + 10m + 1m a 10s de subida + 4m de carga constante (sin rampa de bajada) | load | ⟦≈ 7:50 estimado (12:10 → ≈ 4:20); reemplazar con el valor real del run⟧ |

Total resultante: ⟦7:15 estimado (15:05 − 7:50); reemplazar con el total real⟧ min
Link al run en main con los cambios: ⟦LINK_RUN_MAIN_FINAL⟧

Riesgo aceptado (defecto de la aplicación concreto y efecto en la métrica DORA asignada):
Una fuga de conexiones SQLite en `db.js`, que degrada la latencia solo después de muchos requests. La prueba de 12 minutos hacía unas 19.284 iteraciones (38.568 requests); la de 4 minutos hace unas 6.700 (≈ 13.400 requests). Un defecto que aparece entre los 13.400 y los 38.000 requests antes lo detectaba `load` y ahora llega a producción sin detectarse. Para ese tipo de defecto el time-to-restore sube (empeora): la detección pasa del pipeline a los clientes o al monitoreo, y el tiempo hasta restaurar crece por ese retraso. En general, el recorte baja el time-to-restore: un arreglo o rollback atraviesa un pipeline que pasa de unos 13:46 a unos 5:56 de tiempo de reloj, unos 7:50 menos. Acepto el riesgo porque ese defecto es poco frecuente y las instancias de Lambda se reciclan, lo que limita su efecto.

Decisión sobre sast (se queda / se mueve a otra etapa o evento / informativo) y razón:
Se queda, en pre-build y en todos los eventos. Cuesta 0:21 en la suma de jobs, pero 0 s en tiempo de reloj, porque corre en paralelo con `sca` (0:21) y con `unit` e `integration`, que son más cortos; el camino crítico antes del build es 21 s con o sin `sast`. Por eso quitarlo no mejora el time-to-restore: un arreglo urgente no llega antes a producción. Moverlo solo a pull requests dejaría sin análisis un push directo a `main`, y pasarlo a informativo no ahorra minutos, porque el job se ejecuta igual. Mantenerlo evita que patrones inseguros lleguen a producción, y eso reduce los incidentes que después hay que restaurar. El efecto en time-to-restore es nulo y el costo es de 0:21 sobre un total de 7:15, dentro del presupuesto.

## Declaración de uso de IA
Herramientas usadas y para qué:
