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

- `SF_TomarSiguienteArchivoPendiente` entrega `gRutaArchivoActual`, el nombre del archivo actual y su tipo preliminar
- `SF_ValidarEstructuraNovedadesNomina` confirma estructura y determina hojas elegibles
- luego un subflujo de lectura detallada debe consumir solo las hojas validadas para el periodo objetivo

## Entradas sugeridas

- `gRutaArchivoActual` `Text`
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
- `gListaFilasCabeceraNominaValidas` `List`
- `gListaFilasInicioDatosNominaValidas` `List`
- `gListaQuincenasNominaValidas` `List`
- `gListaHojasNominaDescartadas` `List`
- `gCabeceraNominaDetectada` `Boolean`

## Regla de validacion propuesta

1. abrir el workbook en modo lectura
2. obtener la lista de hojas
3. para cada hoja:
   - leer una muestra controlada de celdas
   - validar presencia de cabecera esperada del template
   - detectar si la cabecera real esta en fila 1 o fila 2
   - identificar si la hoja corresponde a `1RA QUINCENA` o `2DA QUINCENA`
4. comparar la quincena detectada con `pPeriodoNominaObjetivo`
5. para cada hoja valida, guardar en listas paralelas:
   - nombre de hoja
   - fila de cabecera
   - fila inicial de datos
   - quincena detectada
6. agregar la hoja a validas o descartadas
6. si no queda ninguna hoja valida, el archivo no debe pasar a procesamiento

## Cabecera esperada

La validacion debe inspeccionar primero fila 1 y luego fila 2, porque la cabecera puede venir en cualquiera de esas dos filas. La cabecera base esperada para novedades de nomina es:

- `CIUDAD`
- `NOMBRE`
- `CEDULA DE CIUDADANIA`
- `AREA`
- `SUELDO <MES>/<AA>`
- `AUX. EXTRALEGAL <MES>/<AA>`
- `AUX. TRASLADO CIUDAD/ AUX. RODAMIENTO <DD> <MES>/<AA>`
- `HORAS OPERADOR`
- `BONOS OPERADOR NO PRESTACIONAL`
- `HOSPEDAJE`
- `GASTOS REPRESENTACION`
- `TRANSPORTE`
- `VIATICOS NO SALARIALES`
- `DESCUENTOS`
- `NOVEDADES`

Notas de implementacion:

- no se debe quemar `ENE/26` ni ningun otro mes o ano
- las columnas dinamicas deben validarse por patron, no por literal exacto
- la validacion debe normalizar tildes y mayusculas/minusculas
- se acepta cabecera valida si los encabezados fijos esperados aparecen y las columnas dinamicas cumplen el patron

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
  - fila de cabecera detectada
  - fila de inicio de datos
  - quincena detectada
  - decision final sobre la hoja
- resumen final del archivo

## Contrato con el siguiente subflujo

`SF_LeerNovedadesNominaHojasValidas` no debe volver a validar estructura. Debe consumir directamente las listas generadas por este subflujo usando el mismo indice en cada una.

Ejemplo conceptual:

- `gListaHojasNominaValidas[0] = 1RA QUINCENA`
- `gListaFilasCabeceraNominaValidas[0] = 2`
- `gListaFilasInicioDatosNominaValidas[0] = 3`
- `gListaQuincenasNominaValidas[0] = 1RA QUINCENA`

## Manejo de errores

Seguir el mismo patron de los subflujos anteriores:

- `BLOCK ... ON BLOCK ERROR`
- bandera `gEsErrorControlado`
- `THROW ERROR`
- captura final de `ERROR => gMensajeError`
- escritura del error controlado en log

## Nota de evolucion

La primera version ya no debe usar solo heuristicas genericas como `empleado` o `concepto`. Debe apoyarse en la cabecera real del template, aceptando variacion dinamica en las columnas que dependen de mes, ano y fecha de referencia.
