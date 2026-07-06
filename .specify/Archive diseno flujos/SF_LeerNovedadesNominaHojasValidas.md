# SF_LeerNovedadesNominaHojasValidas

Documento base del subflujo `SF_LeerNovedadesNominaHojasValidas` para el RPA de registro de novedades en aplicativo de nomina.

## Objetivo

Leer unicamente las hojas ya aprobadas en `gListaHojasNominaValidas` y construir una lista intermedia de registros crudos de novedades de nomina, sin registrar aun en el sistema y sin resolver todavia conceptos ni empleados.

## Contexto funcional

La seleccion del periodo objetivo no se hace en este subflujo. Esa decision ya viene resuelta desde:

- `SF_Init`, que define `pMesNominaObjetivo`, `pAnoNominaObjetivo` y `pPeriodoNominaObjetivo`
- `SF_ValidarEstructuraNovedadesNomina`, que deja depuradas `gListaHojasNominaValidas` y sus listas paralelas de metadatos

Por tanto:

- si el periodo objetivo es `1RA QUINCENA`, este subflujo debe leer solo esa hoja valida
- si el periodo objetivo es `2DA QUINCENA`, debe leer solo esa hoja valida
- si el periodo objetivo es `MES_COMPLETO`, debe leer todas las hojas validas del archivo

## Relacion con subflujos previos y posteriores

- `SF_TomarSiguienteArchivoPendiente` entrega `gRutaArchivoActual`, `gNombreArchivoActual` y `gTipoPreliminarArchivo`
- `SF_ValidarEstructuraNovedadesNomina` valida el archivo y deja `gListaHojasNominaValidas`
- `SF_LeerNovedadesNominaHojasValidas` construye una lista intermedia de registros crudos
- luego un subflujo posterior debe encargarse de normalizacion, resolucion de conceptos, resolucion de empleados y registro en el sistema

## Entradas sugeridas

- `gRutaArchivoActual` `Text`
- `gNombreArchivoActual` `Text`
- `gListaHojasNominaValidas` `List`
- `gListaFilasCabeceraNominaValidas` `List`
- `gListaFilasInicioDatosNominaValidas` `List`
- `gListaQuincenasNominaValidas` `List`
- `gArchivoLog` `Text`
- `pMesNominaObjetivo` `Text`
- `pAnoNominaObjetivo` `Text`
- `pPeriodoNominaObjetivo` `Text`

## Salidas sugeridas

- `gEstadoLecturaNovedadesNomina` `Text`
- `gMensajeError` `Text`
- `gTotalHojasLeidasNomina` `Number`
- `gTotalRegistrosNominaLeidos` `Number`
- `gListaRegistrosNominaCrudos` `List`
- `gHayRegistrosNominaLeidos` `Boolean`

## Variables internas sugeridas

- `ExcelInstance` `Excel instance`
- `gHojaNominaActual` `Text`
- `gFilaCabeceraActual` `Number`
- `gFilaInicioDatosActual` `Number`
- `gQuincenaHojaActual` `Text`
- `gIndiceHojaNominaActual` `Number`
- `gFilaActual` `Number`
- `gConsecutivoFilasVacias` `Number`
- `gRegistroNominaActual` `Custom object o Dictionary si PAD lo soporta en la implementacion`
- `gEsFilaUtil` `Boolean`

## Flujo propuesto

1. inicializar estado del subflujo y contadores
2. validar que `gListaHojasNominaValidas` contenga al menos una hoja
3. abrir `gRutaArchivoActual` en modo lectura
4. recorrer `gListaHojasNominaValidas` usando el mismo indice para sus listas paralelas
5. recuperar para cada indice:
   - hoja actual
   - fila de cabecera
   - fila de inicio de datos
   - quincena detectada
6. activar la hoja actual
7. iniciar lectura desde `gFilaInicioDatosActual`
8. leer filas de datos una a una
9. evaluar si la fila contiene informacion util
10. construir un registro crudo por cada fila util
11. agregar el registro a `gListaRegistrosNominaCrudos`
12. detener lectura de la hoja cuando se acumulen varias filas vacias consecutivas
13. continuar con la siguiente hoja valida
14. cerrar el archivo Excel
15. registrar resumen del subflujo en log

## Contrato con el subflujo previo

Este subflujo no debe volver a detectar cabecera ni quincena. Debe confiar en lo que ya resolvio `SF_ValidarEstructuraNovedadesNomina`.

Se espera recibir listas paralelas por indice:

- `gListaHojasNominaValidas`
- `gListaFilasCabeceraNominaValidas`
- `gListaFilasInicioDatosNominaValidas`
- `gListaQuincenasNominaValidas`

Ejemplo:

- `gListaHojasNominaValidas[0] = 1RA QUINCENA`
- `gListaFilasCabeceraNominaValidas[0] = 2`
- `gListaFilasInicioDatosNominaValidas[0] = 3`
- `gListaQuincenasNominaValidas[0] = 1RA QUINCENA`

## Regla de lectura de filas

La lectura debe iniciar en la fila indicada por `gFilaInicioDatosActual`, sin recalcular cabecera.

### Criterio de fila util

Una fila puede considerarse util si al menos una de estas columnas contiene valor:

- `NOMBRE`
- `CEDULA DE CIUDADANIA`
- `NOVEDADES`

### Criterio de fin de bloque

Para no depender de una fila final fija, se recomienda detener la lectura cuando se detecten `3` filas consecutivas vacias.

## Estructura sugerida del registro crudo

Cada fila util debe convertirse en un registro intermedio que preserve el contenido original del archivo. Aun no debe hacerse transformacion de negocio final.

Campos sugeridos:

- `ArchivoOrigen`
- `HojaOrigen`
- `PeriodoObjetivo`
- `FilaExcel`
- `Ciudad`
- `NombreEmpleado`
- `CedulaFuente`
- `Area`
- `SueldoTexto`
- `AuxExtralegalTexto`
- `AuxTrasladoORodamientoTexto`
- `HorasOperadorTexto`
- `BonosTexto`
- `HospedajeTexto`
- `GastosRepresentacionTexto`
- `TransporteTexto`
- `ViaticosNoSalarialesTexto`
- `DescuentosTexto`
- `NovedadesTexto`
- `FilaCrudaCompleta`

## Que no debe hacer este subflujo

- no registrar en el sistema
- no decidir codigos de nomina
- no resolver equivalencias de conceptos
- no consultar IA
- no resolver empleados por nombre
- no buscar cedulas faltantes

## Logging esperado

- inicio del subflujo
- archivo actual
- hoja actual
- fila de cabecera recibida
- fila inicial de datos recibida
- quincena recibida
- cantidad de filas utiles leidas por hoja
- cantidad de filas descartadas por estar vacias
- resumen final del archivo

## Manejo de errores

Seguir el mismo patron de los subflujos anteriores:

- `BLOCK ... ON BLOCK ERROR`
- bandera `gEsErrorControlado`
- `THROW ERROR`
- captura posterior con `ERROR => gMensajeError`
- registro del error en `gArchivoLog`
- cierre de Excel si la instancia fue abierta

## Criterio de salida

- si no se leen registros utiles, `gHayRegistrosNominaLeidos = False`
- si se leen uno o mas registros, `gHayRegistrosNominaLeidos = True`
- `gEstadoLecturaNovedadesNomina` debe diferenciar entre:
  - `OK`
  - `SIN_REGISTROS`
  - `ERROR`

## Nota de evolucion

Esta fase debe preservar la lectura del template con la menor interpretacion posible. La resolucion de reglas de negocio debe quedar separada en subflujos posteriores, particularmente:

- `SF_NormalizarRegistrosNomina`
- `SF_ResolverConceptoNomina`
- `SF_ResolverEmpleadoNomina`
- `SF_RegistrarNovedadNominaEnSistema`
