# Tarea para el agente de implementación: granularidad automática en las Tools históricas de `mes-history-mcp` (desafío D3.2)

## Contexto

La tesis define el desafío D3.2 (Capítulo 1): el sistema debe elegir la granularidad de los
datos según el rango temporal de la consulta. Para periodos cortos (última hora, último turno)
conviene el dato fino; para periodos largos (último mes) conviene agregar por hora, turno o día.
Ejemplos del Capítulo 1:

- "Producción en los últimos 30 minutos" → granularidad de minutos.
- "Producción del día de ayer" → agregado por hora.
- "Producción del último mes" → agregado por día o por turno.

Hoy las Tools históricas de `mes-history-mcp` devuelven las filas tal como están en `mes_core`,
sin agregación. Esta tarea agrega un parámetro de granularidad a las Tools que lo admitan y una
regla `auto` decidida **en el servidor** (no por el modelo), de modo que el comportamiento sea
determinista y verificable.

La validación completa (benchmark de preguntas, métricas, repeticiones) se hace después, en el
Capítulo 5 de la tesis. Aquí solo hace falta una verificación funcional y un reporte con la
evidencia que el libro necesita. Este documento es interno y puede nombrar cualquier detalle de
implementación; la tesis no.

## Paso 0. Investigación (solo lectura, antes de escribir código)

Antes de diseñar nada, revisar los datos reales y reportar, por cada tabla que respalda una Tool
histórica (`run`, `statehistory`, `workorder`/`schedule`, `counthistory`):

1. Qué representa una fila (una corrida, un evento de paro, una muestra de conteo...).
2. Resolución temporal real: intervalo típico entre filas, rango de fechas cubierto y número de
   filas por línea.
3. Qué columnas son sumables (conteos, tiempos), cuáles son razones (OEE, disponibilidad,
   rendimiento) y cuáles no se pueden agregar (estados, códigos).
4. Si `mes_core` define turnos en algún lugar (por ejemplo en `schedule` o `stateprocess`). Si los
   define, usar esa definición. Si no, usar el valor por defecto de la sección "Decisiones".
5. Si `counthistory` tiene filas. Hoy se sabe que estaba vacía.

Con esto, decidir **a qué Tools aplica el parámetro**. No todas las Tools tienen sentido
agregadas: `get_standard_rate` y `get_production_orders` describen entidades, no series de tiempo,
y no deben recibirlo. Justificar la decisión en el reporte.

Si la resolución real de una tabla es más gruesa que la granularidad pedida (por ejemplo, la
tabla solo tiene un registro por corrida y se pide minuto), el servidor no debe inventar
resolución. Debe devolver la granularidad más fina disponible e indicarlo en la respuesta.

## Paso 1. Parámetro `granularity`

Agregar a las Tools elegidas en el Paso 0 el parámetro opcional `granularity`, con los valores
`auto` (por defecto), `minute`, `hour`, `shift` y `day`. El contrato actual de cada Tool no debe
romperse: las llamadas existentes, sin el parámetro, deben seguir funcionando.

Reglas generales:

- Todas las consultas siguen siendo `SELECT` parametrizadas con el rol de solo lectura
  `mes_history_reader`. Sin SQL libre ni construcción de consultas con texto del usuario.
- La agregación se hace en SQL (por ejemplo con `date_trunc` o con intervalos), no en Python
  sobre filas ya traídas, para no cargar el servidor con rangos largos.
- Los nombres de área, línea y celda siguen entrando y saliendo con el vocabulario de
  `UNS_Molinera`, como en el resto de las Tools.
- Se mantiene el patrón de devolver el valor con su documentación y unidad.

### Cómo agregar cada tipo de columna

- Conteos y tiempos: suma por intervalo.
- Razones (OEE, disponibilidad, rendimiento, calidad): **no promediar las razones**. Si existen
  las columnas componentes (por ejemplo tiempos de marcha, tiempo planificado, conteos), sumar los
  componentes por intervalo y recalcular la razón. Si no existen, reportar la limitación y usar
  promedio ponderado por duración, indicando el método en la respuesta.
- Eventos de paro: por intervalo, número de eventos y duración total, y la duración máxima.
  Conservar el motivo cuando el intervalo tenga un solo motivo, o la lista de motivos con su
  duración si hay varios.
- Estados y códigos: no se agregan. Si una tabla solo tiene columnas de este tipo, no recibe el
  parámetro.

## Paso 2. Regla `auto`

Cuando `granularity = "auto"`, el servidor elige según la duración del rango
(`date_to - date_from`):

| Duración del rango | Granularidad |
|---|---|
| hasta 3 horas | `minute` |
| más de 3 horas y hasta 48 horas | `hour` |
| más de 48 horas | `day` |

Los umbrales deben ser constantes con nombre en el código, configurables por variable de
entorno, no números sueltos. `shift` solo se usa si el usuario lo pide de forma explícita.

Si la granularidad elegida es más fina que la resolución de la fuente, aplicar lo indicado en el
Paso 0 (usar la más fina disponible e informarlo).

Limitar el tamaño de la respuesta. Si el resultado supera un máximo de filas (constante
configurable, por ejemplo 500), devolver las primeras filas e indicar `truncated: true` con el
total real. Si se trunca con `auto`, sugerir en el mensaje una granularidad más gruesa.

## Paso 3. Qué debe decir la respuesta

Cada respuesta de una Tool con `granularity` debe incluir, además de las filas:

- `granularity_requested` y `granularity_used`.
- `granularity_reason`: texto corto del motivo (por ejemplo "rango de 26 h, regla auto: hour").
- `source_resolution`: la resolución de la fuente, si fue la que limitó el resultado.
- `row_count` y, si aplica, `truncated`.
- `aggregation`: cómo se agregó cada columna (suma, recálculo, máximo...).

## Paso 4. Contexto para el modelo

Actualizar la parte fija del Resource `mes-history://schema` (la `note`) para que el LLM sepa que
existe `granularity`, cuáles son sus valores y cuál es la regla `auto`. Hacerlo en un párrafo
corto. Mantener la restricción vigente: la `note` no debe nombrar `report_ava_masterdata` ni
hablar de idiomas, simulador ni de `UNS_Molinera` más allá de lo que ya dice. Actualizar también la
descripción de cada Tool afectada en `list_tools()`.

## Qué NO cambiar

- Las 13 tablas en alcance, los permisos de `mes_history_reader` y la exclusión de `audit_events`
  y `report_ava_masterdata`.
- La URI y el MIME type del Resource, y su generación dinámica.
- El número de Tools (seis) y de Resources (uno). Esta tarea no agrega Tools nuevas.
- `query_table`: no recibe `granularity`. Sigue siendo la Tool de consulta genérica acotada.
- Las unidades de cada fuente (OEE en fracción 0-1, tiempos en segundos).
- `uns-mcp`.
- Los datos almacenados. **No ejecutar INSERT, UPDATE ni DELETE.** Si faltan datos para probar
  (por ejemplo `counthistory` vacía o poco histórico), reportar qué falta y, si ayuda, entregar un
  script de carga como archivo aparte sin ejecutarlo. Los datos los carga el autor.

## Decisiones por confirmar (valores por defecto si no se contradicen)

- Turnos: si `mes_core` no los define, usar tres turnos de 8 horas (06:00 a 14:00, 14:00 a 22:00 y
  22:00 a 06:00), como constantes configurables.
- Zona horaria: `America/Bogota`. Verificar si las columnas de tiempo son con o sin zona horaria
  y reportarlo, porque afecta el corte de días y turnos.
- Umbrales de `auto`: los de la tabla del Paso 2.

## Verificación funcional (la validación completa va en el Capítulo 5)

Para cada Tool que recibió `granularity`, con una línea que tenga datos:

1. Un rango de 30 minutos, uno de un día y uno de un mes, con `auto`: la granularidad elegida
   coincide con la regla y el número de filas es coherente.
2. Cada valor explícito (`minute`, `hour`, `shift`, `day`) en un mismo rango: la suma de los
   conteos agregados coincide con la suma de las filas originales del mismo rango.
3. OEE agregado: coincide con el recálculo manual a partir de los componentes.
4. Una granularidad más fina que la fuente: se devuelve la más fina disponible, con
   `source_resolution` informado.
5. Un rango que excede el máximo de filas: `truncated: true` y el mensaje sugerido.
6. Entradas inválidas (granularidad inexistente, `date_from` posterior a `date_to`, línea
   inexistente): mensaje de error claro, sin traza.
7. Compatibilidad: llamadas sin `granularity` y `test_server_live.py` completo siguen sin error
   (6 Tools y 1 Resource).
8. `resources/read` sobre `mes-history://schema` incluye el párrafo nuevo y mantiene la `note`
   sin las cadenas prohibidas (buscar `report_ava`, `inglés`, `english`: cero resultados).

## Qué reportar de vuelta

Guardar el reporte como `reporte_implementacion_granularidad.md` y mostrarlo completo. Debe
incluir:

1. **Hallazgos del Paso 0**: por tabla, qué es una fila, resolución real, rango de fechas, número
   de filas y qué columnas son sumables, razones o no agregables. Si `counthistory` sigue vacía.
2. **A qué Tools se aplicó `granularity` y a cuáles no**, con la justificación.
3. **El esquema final de parámetros** de cada Tool afectada, en una tabla lista para pasar al
   Capítulo 4 (Tool, parámetros nuevos, tablas consultadas).
4. **Cómo se agregó cada columna** y cómo se manejaron las razones.
5. **La regla `auto`**, con los umbrales finales y cómo se configuran.
6. **El texto final** del párrafo agregado al Resource y de las descripciones de las Tools.
7. **Un ejemplo de respuesta real** por cada uno de los tres rangos (30 minutos, un día, un mes)
   con los campos nuevos, abreviando las filas.
8. **El resultado de las ocho verificaciones**, con lo que se ejecutó y lo que devolvió.
9. **Zona horaria y turnos**: qué se encontró o qué valor por defecto se usó.
10. **Limitaciones o decisiones** que el autor deba conocer antes de escribir el libro, y los
    archivos modificados.
