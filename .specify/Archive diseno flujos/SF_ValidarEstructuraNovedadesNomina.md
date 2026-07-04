# SF_ValidarEstructuraNovedadesNomina

Documento base del subflujo `SF_ValidarEstructuraNovedadesNomina` para el RPA de registro de novedades en aplicativo de nomina.

## Objetivo

Validar que un archivo clasificado preliminarmente como `NovedadesNomina` tenga la estructura esperada y contenga hojas compatibles con el periodo objetivo de la corrida antes de pasar al procesamiento detallado.

## Contexto de negocio

Los archivos de novedades de nomina tienen una estructura definida:

- contienen una o varias hojas
- cada hoja representa un periodo de una quincena del mismo mes
- en la hoja existe una cabecera conocida del template
- cada hoja debe poder asociarse al menos a `1RA QUINCENA` o `2DA QUINCENA`

A diferencia del flujo de horas extra, aqui los registros no traen una fecha por fila para validar periodo. La validacion de pertenencia al periodo se hace a nivel de hoja.

## Relacion con subflujos previos y posteriores

- `SF_TomarSiguienteArchivoPendiente` entrega el archivo actual y su tipo preliminar
- `SF_ValidarEstructuraNovedadesNomina` confirma estructura y determina hojas elegibles
- luego un subflujo de lectura detallada debe consumir solo las hojas validadas para el periodo objetivo

## Entradas sugeridas

- `pRutaArchivoActual` `Text`
- `pMesNominaObjetivo` `Text`
- `pAnoNominaObjetivo` `Text`
- `pPeriodoNominaObjetivo` `Text`
- `gArchivoLog` `Text`

Valores esperados para `pPeriodoNominaObjetivo`:

- `1RA QUINCENA`
- `2DA QUINCENA`
- `MES_COMPLETO`

## Salidas sugeridas

- `gEstadoValidacionEstructuraNomina` `Text`
- `gMensajeError` `Text`
- `gArchivoNominaValido` `Boolean`
- `gCantidadHojasArchivoActual` `Number`
- `gListaHojasNominaValidas` `List`
- `gListaHojasNominaDescartadas` `List`
- `gCabeceraNominaDetectada` `Boolean`

## Regla de validacion propuesta

1. abrir el workbook en modo lectura
2. obtener la lista de hojas
3. para cada hoja:
   - leer una muestra controlada de celdas
   - validar presencia de cabecera esperada del template
   - identificar si la hoja corresponde a `1RA QUINCENA` o `2DA QUINCENA`
4. comparar la quincena detectada con `pPeriodoNominaObjetivo`
5. agregar la hoja a validas o descartadas
6. si no queda ninguna hoja valida, el archivo no debe pasar a procesamiento

## Cabecera esperada

La version inicial debe manejar una lista configurable de encabezados clave del template de novedades. Ejemplos:

- identificacion del empleado
- nombre del empleado
- concepto o novedad
- valor o cantidad

La lista exacta debe ajustarse al template real y mantenerse centralizada cuando el proceso madure.

## Validacion de periodo

La validacion de periodo en novedades de nomina debe hacerse por hoja:

- si la corrida es `1RA QUINCENA`, solo son validas hojas detectadas como `1RA QUINCENA`
- si la corrida es `2DA QUINCENA`, solo son validas hojas detectadas como `2DA QUINCENA`
- si la corrida es `MES_COMPLETO`, son validas ambas

El mes y el ano objetivo deben quedar registrados en logs aunque en la primera version la deteccion sobre la hoja se concentre en quincena y cabecera.

## Logging esperado

- inicio del subflujo
- archivo y periodo objetivo
- cantidad de hojas detectadas
- resultado por hoja:
  - cabecera valida o invalida
  - quincena detectada
  - decision final sobre la hoja
- resumen final del archivo

## Manejo de errores

Seguir el mismo patron de los subflujos anteriores:

- `BLOCK ... ON BLOCK ERROR`
- bandera `gEsErrorControlado`
- `THROW ERROR`
- captura final de `ERROR => gMensajeError`
- escritura del error controlado en log

## Nota de evolucion

La primera version puede usar heuristicas de texto sobre una muestra de la hoja. La lista definitiva de encabezados y la logica exacta de deteccion de quincena deben ajustarse contra el template real del cliente.
