# SF_DescubrirArchivosEntrada

Documento base del subflujo `SF_DescubrirArchivosEntrada` para el RPA de registro de novedades en aplicativo de nomina. Esta version queda armonizada con la implementacion actual de `SF_Init`, que ya realiza una validacion temprana de existencia de archivos Excel en el directorio de entrada.

## Objetivo

Tomar la lista inicial de archivos detectada por `SF_Init`, depurarla, identificar archivos temporales o no procesables, clasificar preliminarmente los archivos por tipo y dejar listas las colecciones que usaran los subflujos posteriores.

## Relacion con SF_Init

`SF_Init` ya hace una deteccion temprana para:

- validar que el directorio exista
- confirmar que haya archivos Excel
- registrar un conteo inicial

Por eso, `SF_DescubrirArchivosEntrada` no debe rehacer esa exploracion en cada corrida. Su comportamiento esperado es:

1. reutilizar `gListaArchivosAProcesar` si ya viene cargada desde `SF_Init`
2. solo volver a consultar el directorio si la lista esta vacia o si se solicita redescubrimiento forzado
3. depurar archivos temporales como `~$*.xlsx`
4. clasificar preliminarmente por nombre de archivo
5. dejar listas separadas para horas extra, novedades de nomina y no clasificados

## Firma sugerida del subflujo

### Entradas

- `pDirectorioEntrada` `Text`
- `pForzarRedescubrimiento` `Boolean`
- `gArchivoLog` `Text`
- `gListaArchivosAProcesar` `List`

### Salidas

- `gEstadoDescubrimiento` `Text`
- `gMensajeError` `Text`
- `gListaArchivosAProcesar` `List`
- `gListaArchivosHorasExtra` `List`
- `gListaArchivosNovedadesNomina` `List`
- `gListaArchivosNoClasificados` `List`
- `gListaArchivosTemporales` `List`
- `gTotalArchivosAProcesar` `Number`
- `gTotalArchivosHorasExtra` `Number`
- `gTotalArchivosNovedadesNomina` `Number`
- `gTotalArchivosNoClasificados` `Number`
- `gTotalArchivosTemporales` `Number`

## Reglas de clasificacion preliminar

La clasificacion de este subflujo es preliminar. No define todavia la logica final de negocio del archivo, solo orienta el procesamiento posterior.

Reglas iniciales propuestas:

- si el nombre contiene `hora`, `horas` o `extra`: `HorasExtra`
- si el nombre contiene `novedad`, `novedades` o `nomina`: `NovedadesNomina`
- en cualquier otro caso: `NoClasificado`

## Flujo propuesto en PAD

1. inicializar estado y listas de salida
2. decidir si se reutiliza la lista de `SF_Init` o si se vuelve a consultar el directorio
3. iterar cada archivo candidato
4. extraer nombre de archivo
5. descartar temporales `~$`
6. agregar archivo a la lista general depurada
7. clasificar preliminarmente el archivo por nombre
8. registrar resumen en log
9. dejar listas listas para procesamiento posterior

## Comportamiento esperado de logging

Debe seguir el mismo patron de `SF_Init`:

- log de inicio del subflujo
- log indicando si se reutilizo lista o se forzo redescubrimiento
- log por cada archivo temporal descartado
- log por cada archivo clasificado
- log resumen de conteos por categoria
- log de finalizacion satisfactoria

## Manejo de errores

Debe seguir el mismo patron estructural que `SF_Init`:

- `BLOCK ... ON BLOCK ERROR`
- bandera `gEsErrorControlado`
- `THROW ERROR`
- bloque final que capture `ERROR => gMensajeError`
- escritura del error controlado en log

## Siguiente subflujo recomendado

Despues de `SF_DescubrirArchivosEntrada`, la propuesta mas consistente es uno de estos dos caminos:

- `SF_AutenticarWeb`, si quieres consolidar la sesion antes de tocar logica por archivo
- `SF_DeterminarTipoArchivo`, si prefieres validar estructura real del workbook antes del login

Con la informacion actual, el siguiente mas robusto seria `SF_DeterminarTipoArchivo`, porque la clasificacion por nombre es solo preliminar.
