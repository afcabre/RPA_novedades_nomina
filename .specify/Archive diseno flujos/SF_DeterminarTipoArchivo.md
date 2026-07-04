# SF_DeterminarTipoArchivo

Documento base del subflujo `SF_DeterminarTipoArchivo` para el RPA de registro de novedades en aplicativo de nomina. Este subflujo toma un archivo Excel puntual y valida su tipo real a partir de la estructura del workbook, para no depender unicamente de la clasificacion preliminar por nombre realizada en `SF_DescubrirArchivosEntrada`.

## Objetivo

Abrir un archivo Excel puntual, inspeccionar una parte minima de su estructura y determinar si corresponde a:

- `HorasExtra`
- `NovedadesNomina`
- `NoIdentificado`

## Relacion con subflujos previos

- `SF_Init` valida que exista carga de trabajo y deja el entorno listo.
- `SF_DescubrirArchivosEntrada` depura y clasifica preliminarmente los archivos por nombre.
- `SF_DeterminarTipoArchivo` confirma el tipo real del archivo usando contenido del workbook.

## Firma sugerida del subflujo

### Entradas

- `pRutaArchivoActual` `Text`
- `gArchivoLog` `Text`
- `pTipoPreliminarArchivo` `Text`

### Salidas

- `gEstadoDeterminacionTipo` `Text`
- `gMensajeError` `Text`
- `gTipoArchivoDeterminado` `Text`
- `gNombreArchivoActual` `Text`
- `gCantidadHojasArchivoActual` `Number`
- `gCoincidenciasHorasExtra` `Number`
- `gCoincidenciasNovedadesNomina` `Number`

## Estrategia de validacion

La validacion debe ser minima y barata. No se busca procesar el archivo todavia, solo reconocer patron general.

### Regla propuesta

1. abrir el workbook en modo lectura
2. obtener el listado de hojas
3. leer una muestra controlada del contenido, por ejemplo primeras filas y columnas relevantes
4. buscar palabras clave asociadas a cada tipo
5. decidir tipo final

### Palabras clave iniciales sugeridas

`HorasExtra`

- `hora`
- `horas`
- `extra`
- `recargo`
- `nocturno`

`NovedadesNomina`

- `novedad`
- `concepto`
- `nomina`
- `incapacidad`
- `vacaciones`
- `licencia`

## Regla de decision

- si predominan coincidencias de `HorasExtra`: `HorasExtra`
- si predominan coincidencias de `NovedadesNomina`: `NovedadesNomina`
- si no hay evidencia suficiente: `NoIdentificado`

## Logging esperado

- inicio del analisis del archivo
- tipo preliminar recibido
- total de hojas detectadas
- resumen de coincidencias por categoria
- tipo final determinado
- cierre del subflujo

## Manejo de errores

Seguir el mismo patron estructural que en los otros subflujos:

- `BLOCK ... ON BLOCK ERROR`
- bandera `gEsErrorControlado`
- `THROW ERROR`
- captura final de `ERROR => gMensajeError`
- escritura en log

## Notas de implementacion

- abrir Excel en modo invisible y solo lectura
- cerrar el archivo al final, tanto en flujo normal como en manejo de error
- esta version puede comenzar con una heuristica simple y luego refinarse cuando conozcamos mejor la estructura real de los archivos
