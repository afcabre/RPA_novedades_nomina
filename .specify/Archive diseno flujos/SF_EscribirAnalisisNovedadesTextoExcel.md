# SF_EscribirAnalisisNovedadesTextoExcel

Documento base del subflujo `SF_EscribirAnalisisNovedadesTextoExcel` para el RPA de registro de novedades en aplicativo de nomina.

## Objetivo

Persistir la salida de `SF_AnalizarNovedadTextoNomina` en la hoja `novedadesTexto` del archivo Excel de evidencia de la corrida.

Este subflujo no analiza texto ni reconsulta IA. Solo escribe evidencia estructurada y deja el archivo listo para revision posterior.

## Relacion con subflujos previos y posteriores

- `SF_Init` prepara la ruta del archivo de evidencia de la corrida
- `SF_ExpandirConceptosNomina` preserva novedades textuales pendientes
- `SF_AnalizarNovedadTextoNomina` produce `gListaNovedadesTextoAnalizadas`
- `SF_EscribirAnalisisNovedadesTextoExcel` persiste esa lista en Excel
- `SF_ConciliarItemsNomina` debe consumir despues la hoja `novedadesTexto` junto con otras salidas de evidencia

## Entradas esperadas

- `gRutaArchivoAnalisisNovedades` `Text`
- `gIdEjecucion` `Text`
- `gArchivoLog` `Text`
- `gListaNovedadesTextoAnalizadas` `List`

## Salidas sugeridas

- `gEstadoEscrituraAnalisisNovedadesExcel` `Text`
- `gMensajeError` `Text`
- `gTotalFilasAnalisisNovedadesEscritas` `Number`
- `gTotalFilasAnalisisNovedadesConErrorFormato` `Number`
- `gHayFilasAnalisisNovedadesEscritas` `Boolean`

## Hoja objetivo

- nombre de hoja: `novedadesTexto`

Si la hoja no existe y el archivo plantilla conserva una hoja base `Sheet1`, el subflujo puede copiar `Sheet1` para crear `novedadesTexto`.

No se recomienda renombrar destructivamente la hoja base si luego puede reutilizarse para otras salidas de evidencia.

## Contrato de entrada esperado

Cada item de `gListaNovedadesTextoAnalizadas` debe venir con este orden:

```text
ArchivoOrigen|HojaOrigen|QuincenaHoja|FilaExcel|Ciudad|NombreEmpleado|CedulaFuente|Area|SueldoBaseTexto|AuxExtralegalBaseTexto|NovedadTextoOriginal|TieneRegistrosFuenteMaterializados|MotivoPendienteTexto|IndiceFragmento|TextoFragmento|TipoSugeridoIA|MontoMencionado|DiasMencionados|RequiereCalculo|TieneRespaldoEstructuralSugerido|ConfianzaIA|RequiereRevision|ObservacionTecnica
```

## Columnas V1 de la hoja `novedadesTexto`

- `IdEjecucion`
- `ArchivoOrigen`
- `HojaOrigen`
- `FilaExcel`
- `Ciudad`
- `NombreEmpleado`
- `CedulaFuente`
- `Area`
- `SueldoBaseTexto`
- `AuxExtralegalBaseTexto`
- `NovedadTextoOriginal`
- `IndiceFragmento`
- `TextoFragmento`
- `TipoSugeridoIA`
- `MontoMencionado`
- `DiasMencionados`
- `RequiereCalculo`
- `TieneRespaldoEstructuralSugerido`
- `ConfianzaIA`
- `RequiereRevision`
- `ObservacionTecnica`
- `EstadoRevisionUsuario`
- `DecisionUsuario`
- `ObservacionRevisionUsuario`

## Valores iniciales de validacion humana

Al momento de escribir:

- `EstadoRevisionUsuario = Pendiente`
- `DecisionUsuario = ""`
- `ObservacionRevisionUsuario = ""`

## Reglas de escritura

1. validar que `gListaNovedadesTextoAnalizadas` tenga elementos
2. validar que exista `gRutaArchivoAnalisisNovedades`
3. abrir el archivo Excel en modo escritura
4. asegurar la existencia de la hoja `novedadesTexto`
5. si la fila 1 esta vacia, escribir cabeceras V1
6. encontrar la primera fila libre
7. recorrer `gListaNovedadesTextoAnalizadas`
8. validar formato minimo del item
9. escribir una fila por fragmento analizado
10. guardar y cerrar Excel

## Logging esperado

- inicio del subflujo
- archivo destino
- total de registros recibidos
- errores de formato detectados
- total de filas escritas
- resumen final

No se recomienda loggear todas las filas escritas salvo modo debug.

## Manejo de errores

Seguir el patron ya usado:

- `BLOCK ... ON BLOCK ERROR`
- bandera `gEsErrorControlado`
- `THROW ERROR`
- captura final con `ERROR => gMensajeError`
- cierre de Excel si quedo abierto
- escritura en `gArchivoLog`

## Alcance deliberadamente limitado

Este subflujo:

- no reanaliza respuestas IA
- no recalcula montos
- no decide aprobacion final
- no cruza aun contra registros materiales de nomina

Su responsabilidad es solo persistencia consistente de evidencia.
