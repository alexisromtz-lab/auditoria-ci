# Informe de auditoría de CI · Nivel 1 · muestra

<!-- HUELLA-A3-INFORME-MUESTRA-TT-METAL-20261021 -->

**Esto es una muestra, no un entregable vendido.** Es la forma exacta que tendría un informe de Nivel 1,
aplicada a un repositorio público real. No se cobró, no se reservó nada y no se abrió ninguna ventana comercial
al producirla.

## 1 · Cabecera del objeto auditado

| Campo | Valor de esta muestra | Qué significa para tu repositorio |
|---|---|---|
| `TARGET` | `tenstorrent/tt-metal` (público) | el repositorio auditado, sin acceso privado ni credenciales |
| Ancla | `main` en `d0ef2f5d` + un `head.sha` por PR | el commit exacto; un veredicto describe el SHA medido, no «el PR» |
| `LÍNEAS_PRE` | no aplica (repo de terceros) | en tu repo, líneas declaradas antes de correr |
| `BLOB_PRE` | no aplica (repo de terceros) | en tu repo, blob declarado antes de correr |
| Ventana de medición | 2026-10-21, ~19:46–19:55Z | la fotografía tiene fecha; se re-corre para datarla |
| Instrumento | `tools/observador_tt_metal_20261020.py --sweep` · `--rojo` · `--blame` | el mismo binario público, sin versión privada |
| Errores de API | **0** | una corrida con errores no se publica |

En un asiento de cliente, `BLOB_PRE` y `LÍNEAS_PRE` van en esta cabecera y son lo que se certifica.
Aquí no existen porque el repositorio no es tuyo ni mío: se sustituyen por el `head.sha` de cada PR, y se dice.

## 2 · Resumen ejecutivo

| Medida | Valor | Denominador |
|---|---|---|
| PRs abiertos medidos | **1.608** | límite tomado en un instante; **no** es censo |
| Inventariados con shard de simulación visible | **236** | del universo anterior |
| Con más de 100 check-runs en el SHA | **235** | de 236 |
| Falso verde estricto | **33** | **14,0%** de 236 · **28,7%** de los rojos |
| Rojo visible en la ventana | **115** | de 236 inventariados |
| Sin shard (el pipeline no corre para ese diff) | **1.372** | de 1.608 |
| Logs de job clasificados | **114** | de 115 rojos |

**Lectura.** El rollup de GitHub entrega una ventana de 100 check-runs. Treinta y tres PRs leían verde en esa
ventana mientras el shard rojo estaba más allá del corte. El número es menor que el declarado en una corrida
anterior («143»), y ese 143 **no reproduce**: se publica 33 porque es lo que se puede re-auditar hoy.
Un barrido que solo lee la primera ventana reporta un **cero falso**, y ese cero falso se detectó primero en casa.

## 3 · Metodología (lo que se hizo, no lo que se promete)

1. **Paginación completa.** Se paginan PRs y check-runs por encima de cada corte; el corte se declara (`100`) en
   cada fila, no se esconde. Un flujo que se detiene en la primera página mide la primera página.
2. **Un veredicto por PR, anclado.** Cada veredicto se ata al `head.sha` medido. Si la rama se mueve, la fila no
   se vuelve falsa: se vuelve histórica. Esto es lo que permite re-auditar sin discutir de memoria.
3. **Control negativo declarado.** El régimen «leer la ventana» es a la vez el objeto de la medición y su control:
   compararlo con «leer todo» es lo que hace visible el falso verde. Sin esa comparación, el instrumento mediría
   su propia ceguera y la llamaría salud.
4. **Un log por PR rojo.** `--rojo` descarga el log del job y clasifica el **mecanismo**. Un check-run nunca trae
   el texto del fallo: `output.summary` es nulo en todas las corridas muestreadas.
5. **Inspección de fuentes con referencia fija.** `--blame` inspecciona test y kernel en un snapshot declarado y
   entrega **observaciones**, no un veredicto de culpabilidad.

## 4 · Hallazgos

### 4.1 · El rojo no es una cosa

| Mecanismo (censo sobre 114 logs de 115 rojos) | PRs |
|---|---|
| `assert` — el test corre y compara; el rojo habla del objeto | **37** |
| `tt_fatal` — el runtime C++ aborta; el test no llega a comparar | **37** |
| `timeout` — el job muere por el límite de GitHub; no hay veredicto | **35** |
| otros errores / sin log | **6** |

### 4.2 · Tres familias, no 115 bugs

| Familia (archivo, o mecanismo cuando no hay archivo) | PRs |
|---|---|
| `test_prefetcher_worker_pipes.py` · `TT_FATAL sub_device_manager_tracker.cpp:118` | **36** |
| `mecanismo:timeout` · grupo *fused* del SKU sim | **35** |
| `test_unary_category6_float32.py` · `clamp`/`clip`/`threshold`, signed-zero | **27** |
| cola · 10 archivos distintos | **16** |

Concentración top-3 = **86%**: tres mecanismos explican **98 de 114** PRs rojos. Con el criterio declarado antes
de correr, el veredicto es *sistémico*.

Falsadores aplicados, y ambos fallan a favor de lo declarado:

- **F-CULP-1** (*≥80% en un solo archivo ⇒ una sola rotura*): **falso**. El archivo mayor es 36/114 = **31,6%**.
- **F-CULP-2** (*distribución plana ⇒ roturas independientes*): **falso**. Tres familias retienen el 86%.

## 5 · Honestidad obligatoria: lo que este informe **no** adjudica

Esta sección es el producto. Cualquiera puede listar rojos; lo que se compra es saber **dónde termina el dato**.

1. **Cluster `assert` signed-zero (27) — `INDECIDIBLE`.** Veinticinco de las 27 ramas contienen el commit del test
   (`cb0eabe0`, `behind_by = 0`) y siguen fallando el 2026-10-21. El log reporta diferencias **solo de signo**
   (32.512 de 65.536 elementos; cero diferencias por valor). Decidir entre defecto de kernel y comparación
   demasiado estricta **no se puede** desde el comparador: hace falta un contrato normativo de signed-zero o un
   diferencial controlado. Sin contrato localizado, la atribución queda bloqueada. El veredicto es `INDECIDIBLE`,
   no `INDECIDIBLE_CON_MATRIZ`. Hay un plan de contraste propuesto y **no ejecutado**; no cambia el veredicto.
2. **Cluster `tt_fatal` (36) — base rancia, no defecto sin arreglar.** Las 36 ramas llevan la misma firma
   (`Using sub device managers is unsupported with slow dispatch`). El arreglo existe: #61470 (`78957cf0`,
   2026-10-20T12:47Z). **Ninguna de las 36 ramas lo contiene**: están `diverged`, 13–65 commits atrás.
   Predicción testeable: rebasar sobre `78957cf0` y el `TT_FATAL` desaparece. Llamarlo «bug sin arreglar» sería
   un falso positivo del propio informe.
3. **Cluster `timeout` (35) — timeout no es cuelgue.** El job muere por el límite de GitHub. Los puntos listados
   son el último test **iniciado**, no un cuelgue probado. La forma es un grupo que no termina, no un test que falla.
4. **78 de 79 diffs no tocan el archivo que falla.** De los **121** rojos del censo, **42** no nombran archivo
   (38 `timeout`, 3 `otro_error`, 1 `sin_log`: `NO_EVALUABLE`); de los **79** restantes, **78** no lo tocan y
   **1** sí (#58101). Es una auditoría de diffs **separada**, y `--rojo` no la calcula. Muestra **desacople**
   entre síntoma y diff; **no prueba** que esos PRs sean víctimas ni que la base esté rota. El `114 de 115`
   queda **retirado**: esa corrida no conservó crudo. Portador:
   [`crudos/atribucion_diff_79_20261021.json`](../../crudos/atribucion_diff_79_20261021.json).
   <!-- HUELLA-SAMPLE-ATRIBUCION-DIFF-79-20261022 · gate A2.1 RC 0 · BLOB_PRE 5b57af1cd7c13af7ff170e6ee4c6b4e7b508f65f · LINEAS_PRE 133 -->
5. **Hueco estructural: los shards sim no corren en `main`.** `main` en `d0ef2f5d` lleva **1.317** check-runs; solo
   **2** tienen un shard sim en el nombre, y ambos están verdes — pero son **otros grupos**. Los 115 rojos viven en
   `ttnn-sanity-tests`, que solo corre en PRs. «¿Está roto `main`?» **no se responde** desde el rollup: falta el
   control negativo. Es el mismo error del §2, en dirección contraria.
6. **Una fotografía.** Tomada el 2026-10-21. Los PRs cierran, las ramas rebasan, las familias se disuelven.
   Se re-corre para datarla; no se cita como estado permanente.

## 6 · Qué recibirías (Nivel 1)

**Un repositorio y un snapshot de commit. Precio propuesto: USD 500, pago único. Sin cobro ni reserva en esta etapa.**

- El **JSON por PR** con esquema estable: `pr` · `head_sha` · `ancla` · `checks_declarados` · `checks_vistos` ·
  veredicto · `completado` · y, para los rojos, `job` · `tipo` (mecanismo) · `archivo` · `test` · `mensaje` ·
  `firma_ttfatal` · `archivo_ultimo` · `fallos` · `kb`.
- El **censo rojo** clasificado por mecanismo y agrupado por familia, con la concentración y sus falsadores.
- La **cabecera del asiento**: `TARGET` · `BLOB_PRE` · `HUELLA` · `LÍNEAS_PRE`, y los **límites declarados** de
  cada cifra (universo, corte, ventana, fecha).
- Los veredictos **`INDECIDIBLE`** escritos como tales, con lo que haría falta para resolverlos.

**No incluye, y se dice aquí para que nadie lo compre por error:** «matriz de solución» (no existe), ejecución en
hardware ni runners de terceros, atribución de culpabilidad donde el dato no alcanza, ni acceso a código privado.

Redacción en tu repositorio: el mismo informe, con `PR` → `[PR-nn]`, rutas internas → `[ruta interna]`,
fragmentos de log recortados, y las cifras intactas. Lo que se censura es la identidad; la aritmética no se toca.

**Contacto:** [alexis_romtz@hotmail.com](mailto:alexis_romtz@hotmail.com?subject=Auditor%C3%ADa%20CI) · asunto
**«Auditoría CI»** · rol, organización, confirmación de que viste el precio propuesto de USD 500 y enlace a un
**repositorio público**. No envíes enlaces privados ni credenciales. El correo solo expresa interés; no reserva
un cupo.

<!-- HUELLA-A3-INFORME-MUESTRA-TT-METAL-20261021 -->
