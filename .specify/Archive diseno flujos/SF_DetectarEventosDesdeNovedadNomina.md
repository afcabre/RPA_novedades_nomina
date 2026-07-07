# SF_DetectarEventosDesdeNovedadNomina

Documento base del subflujo `SF_DetectarEventosDesdeNovedadNomina` para el RPA de registro de novedades en aplicativo de nomina.

## Objetivo

Consumir `gListaNovedadesTextoPendientes` y producir candidatos de evento textual para fases posteriores, sin alterar el contrato material de `SF_ExpandirConceptosNomina`.

## Motivacion

Hay filas del Excel que no traen concepto materializado en columnas, pero si contienen informacion relevante en `NOVEDADES`, por ejemplo:

- incapacidad
- vacaciones
- licencia
- observaciones con monto no respaldado

Esa informacion no debe perderse, pero tampoco debe convertirse prematuramente en concepto dentro de `SF_ExpandirConceptosNomina`.

## Entrada esperada

- `gListaNovedadesTextoPendientes` `List`
- `gTotalNovedadesTextoPendientes` `Number`
- `gArchivoLog` `Text`

## Contrato exacto de entrada en PAD

Formato esperado por item de `gListaNovedadesTextoPendientes`:

```text
ArchivoOrigen|HojaOrigen|QuincenaHoja|FilaExcel|Ciudad|NombreEmpleado|CedulaFuente|Area|SueldoBaseTexto|AuxExtralegalBaseTexto|NovedadTextoOriginal|TieneConceptosExplicitos|MotivoPendienteTexto|RequiereRevision
```

## Salidas sugeridas

- `gEstadoDeteccionEventosNomina` `Text`
- `gMensajeError` `Text`
- `gListaEventosTextoNominaCandidatos` `List`
- `gTotalEventosTextoNominaCandidatos` `Number`
- `gHayEventosTextoNominaCandidatos` `Boolean`

## Naturaleza del subflujo

Este subflujo no debe:

- registrar en UI
- resolver catalogo final
- autorizar montos solo por texto
- reemplazar la conciliacion

Su funcion es solo clasificar y preservar posibles eventos textuales para analisis posterior.

## Regla de entrada

Solo debe consumir filas que ya fueron preservadas como pendientes textuales por `SF_ExpandirConceptosNomina`.

## Unidad de salida sugerida

Cada evento textual candidato deberia contener:

- `ArchivoOrigen`
- `HojaOrigen`
- `QuincenaHoja`
- `FilaExcel`
- `Ciudad`
- `NombreEmpleado`
- `CedulaFuente`
- `Area`
- `SueldoBaseTexto`
- `AuxExtralegalBaseTexto`
- `NovedadTextoOriginal`
- `TipoEventoTextoDetectado`
- `MetodoDeteccion`
- `ConfianzaDeteccion`
- `RequiereCalculo`
- `PermiteAutoRegistroInicial`
- `RequiereRevision`
- `ObservacionTecnica`

## Contrato exacto de salida en PAD

Formato recomendado por item de `gListaEventosTextoNominaCandidatos`:

```text
ArchivoOrigen|HojaOrigen|QuincenaHoja|FilaExcel|Ciudad|NombreEmpleado|CedulaFuente|Area|SueldoBaseTexto|AuxExtralegalBaseTexto|NovedadTextoOriginal|TieneConceptosExplicitos|MotivoPendienteTexto|TipoEventoTextoDetectado|MetodoDeteccion|ConfianzaDeteccion|RequiereCalculo|PermiteAutoRegistroInicial|RequiereRevision|ObservacionTecnica
```

## Valores sugeridos

- `TipoEventoTextoDetectado = INCAPACIDAD | VACACIONES | LICENCIA | TEXTO_MONETARIO_SIN_RESPALDO | TEXTO_NO_CLASIFICADO`
- `MetodoDeteccion = REGLA | PATRON | IA | MANUAL`
- `ConfianzaDeteccion = ALTA | MEDIA | BAJA`

## Criterio de diseno

La heuristica debe ser minima y cerrada.

Version inicial recomendada:

- detectar solo un conjunto reducido de tipos de evento aprobados
- todo lo demas sale como `TEXTO_NO_CLASIFICADO`
- los montos que aparezcan solo en texto deben salir como `TEXTO_MONETARIO_SIN_RESPALDO`

## Regla de alcance para version 1

Este subflujo no debe intentar resolver si el texto adicional ya quedo suficientemente cubierto por un concepto explicito.

Version 1 recomendada:

- si la fila llega como pendiente textual, se analiza
- si no se puede clasificar en la whitelist cerrada, sale como `TEXTO_NO_CLASIFICADO`
- la decision final de descarte o autorregistro queda fuera de este subflujo

## Logging esperado

- inicio del subflujo
- total de pendientes textuales recibidas
- total de eventos detectados
- total no clasificados
- total de montos sin respaldo
- resumen final

## Manejo de errores

Seguir el patron ya usado:

- `BLOCK ... ON BLOCK ERROR`
- bandera `gEsErrorControlado`
- `THROW ERROR`
- captura final con `ERROR => gMensajeError`
- escritura en `gArchivoLog`

## Relacion con fases posteriores

La salida de este subflujo debe alimentar:

- `SF_ConciliarItemsNomina`
- o una fase equivalente de clasificacion y autorizacion

La decision sobre calculo, autorregistro o revision manual no debe cerrarse aqui.
