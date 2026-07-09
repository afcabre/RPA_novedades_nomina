# SF_EscribirRegistrosRegistroNovedadesNomina

Documento base del subflujo `SF_EscribirRegistrosRegistroNovedadesNomina` para el RPA de registro de novedades en aplicativo de nomina.

## Objetivo

Persistir en el archivo de analisis de la corrida la cola operativa generada por:

- `SF_PrepararRegistrosRegistroNovedadesNomina`

La salida debe escribirse en una hoja independiente, orientada a operacion y validacion, no a trazabilidad tecnica.

## Hoja objetivo

- `registroNovedadesNomina`

## Entrada esperada

- `gRutaArchivoAnalisisNovedades`
- `gListaRegistrosRegistroNovedadesNomina`
- `gTotalRegistrosRegistroNovedadesNomina`
- `gArchivoLog`

## Salidas sugeridas

- `gEstadoEscrituraRegistrosRegistroNovedadesNomina`
- `gMensajeError`
- `gTotalFilasRegistrosRegistroNovedadesNominaEscritas`
- `gTotalFilasRegistrosRegistroNovedadesNominaConErrorFormato`
- `gHayFilasRegistrosRegistroNovedadesNominaEscritas`

## Contrato V1 esperado por fila

```text
IdEjecucion|ArchivoOrigen|HojaOrigen|FilaExcel|NombreTercero|IdTercero|Concepto|IdConcepto|Valor|Dias|Observaciones|OrigenResolucionConcepto|EstadoPreparacionRegistro|RequiereValidacionUsuario|MotivoValidacion|__END__
```

## Cabeceras sugeridas en Excel

- `IdEjecucion`
- `ArchivoOrigen`
- `HojaOrigen`
- `FilaExcel`
- `NombreTercero`
- `IdTercero`
- `Concepto`
- `IdConcepto`
- `Valor`
- `Dias`
- `Observaciones`
- `OrigenResolucionConcepto`
- `EstadoPreparacionRegistro`
- `RequiereValidacionUsuario`
- `MotivoValidacion`

## Regla de escritura

Este subflujo no debe recalcular ni reinterpretar datos.

Solo debe:

- abrir el archivo de analisis
- crear la hoja si no existe a partir de `Sheet1`
- escribir cabeceras si la hoja esta vacia
- ubicar la primera fila libre
- escribir cada registro valido

## Criterio operativo

Si una persona abre esta hoja, debe poder usarla como cola de trabajo para:

- dejar correr el registro automatico
- o revisar y aprobar antes del registro

No debe necesitar abrir `conciliacionNovedadesNomina` para entender que hacer con cada fila.
