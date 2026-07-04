# SF_TomarSiguienteArchivoPendiente

Documento base del subflujo `SF_TomarSiguienteArchivoPendiente` para el RPA de registro de novedades en aplicativo de nomina.

## Objetivo

Tomar la cola generada por `SF_DescubrirArchivosEntrada`, obtener el siguiente archivo pendiente y dejar listas las variables de contexto para el procesamiento posterior.

Este subflujo no valida estructura interna del Excel. Su responsabilidad es solo orquestar la cola y entregar el siguiente candidato.

## Relacion con subflujos previos y posteriores

- `SF_Init` prepara el entorno.
- `SF_DescubrirArchivosEntrada` depura y clasifica preliminarmente los archivos.
- `SF_TomarSiguienteArchivoPendiente` toma uno de esos archivos y lo marca como actual.
- Luego el flujo se desvía al validador estructural especializado:
  - `SF_ValidarEstructuraNovedadesNomina`
  - `SF_ValidarEstructuraHorasExtra`

## Firma sugerida del subflujo

### Entradas

- `gListaArchivosHorasExtra` `List`
- `gListaArchivosNovedadesNomina` `List`
- `gListaArchivosNoClasificados` `List`
- `gArchivoLog` `Text`
- `pPrioridadTipoArchivo` `Text`
- `gIndiceNovedadesNominaActual` `Number`
- `gIndiceHorasExtraActual` `Number`
- `gIndiceNoClasificadosActual` `Number`

### Salidas

- `gEstadoSeleccionArchivo` `Text`
- `gMensajeError` `Text`
- `gRutaArchivoActual` `Text`
- `gNombreArchivoActual` `Text`
- `gTipoPreliminarArchivo` `Text`
- `gHayArchivoPendiente` `Boolean`
- `gIndiceNovedadesNominaActual` `Number`
- `gIndiceHorasExtraActual` `Number`
- `gIndiceNoClasificadosActual` `Number`

## Regla de seleccion propuesta

1. si `pPrioridadTipoArchivo = NovedadesNomina`, intentar primero esa lista
2. luego intentar `HorasExtra`
3. luego intentar `NoClasificados`
4. si la prioridad es `HorasExtra`, invertir el orden de las dos primeras listas
5. si todas las listas estan vacias, no hay archivo pendiente
6. la primera version no elimina elementos de las listas; avanza con cursores por tipo

## Logging esperado

- inicio del subflujo
- prioridad solicitada
- archivo seleccionado y tipo preliminar
- mensaje cuando no quedan archivos pendientes

## Manejo de errores

Mantener el mismo patron estructural:

- `BLOCK ... ON BLOCK ERROR`
- bandera `gEsErrorControlado`
- `THROW ERROR`
- captura final de `ERROR => gMensajeError`

## Nota de evolucion

La primera version usa cursores por tipo de lista en lugar de eliminar elementos. Esto evita depender de acciones de listas que pueden variar entre versiones de PAD y deja el avance del procesamiento de forma mas estable.
