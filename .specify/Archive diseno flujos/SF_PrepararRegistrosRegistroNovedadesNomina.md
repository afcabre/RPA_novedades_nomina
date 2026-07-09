# SF_PrepararRegistrosRegistroNovedadesNomina

Documento base del subflujo `SF_PrepararRegistrosRegistroNovedadesNomina` para el RPA de registro de novedades en aplicativo de nomina.

## Objetivo

Construir una salida operativa limpia para la etapa de registro en el sistema, separada de la hoja tecnica de conciliacion.

Este subflujo no decide conciliacion, no resuelve catalogo y no llama IA. Solo toma el resultado ya consolidado y prepara una lista para:

- autorregistro posterior
- validacion humana previa al registro

## Motivacion

La hoja `conciliacionNovedadesNomina` ya concentra demasiada trazabilidad:

- fuente Excel
- conciliacion
- materializacion de calculos
- resolucion de catalogo
- sugerencias IA

Esa hoja sirve para auditoria y soporte, pero no es una buena cola operativa para registrar novedades en el sistema.

Por eso se requiere una capa separada con solo los datos clave de operacion.

## Relacion con subflujos previos y posteriores

- `SF_ConciliarItemsNomina` deja el estado funcional del registro
- `SF_MaterializarRegistrosCalculablesNovedadesNomina` completa los casos calculables soportados
- `SF_ResolverCatalogoConceptosNomina` y `SF_ResolverCatalogoConceptosNominaIA` dejan el concepto sistema final o sugerido
- `SF_ClasificarRegistrosListosNovedadesNomina` particiona los registros entre:
  - `LISTO_AUTORREGISTRO`
  - `REQUIERE_VALIDACION_USUARIO`

Este subflujo debe consumir esa clasificacion y producir la cola operativa.

Luego debe existir un subflujo hermano de persistencia, por ejemplo:

- `SF_EscribirRegistrosRegistroNovedadesNomina`

## Entradas esperadas

- `gListaClasificacionRegistrosListosNovedadesNomina`
- `gListaRegistrosListosAutorregistroNovedadesNomina`
- `gListaRegistrosRequierenValidacionNovedadesNomina`
- `gArchivoLog`

## Salidas sugeridas

- `gEstadoPreparacionRegistrosRegistroNovedadesNomina`
- `gMensajeError`
- `gListaRegistrosRegistroNovedadesNomina`
- `gTotalRegistrosRegistroNovedadesNomina`
- `gHayRegistrosRegistroNovedadesNomina`
- `gTotalRegistrosAutorregistroPreparadosNovedadesNomina`
- `gTotalRegistrosValidacionPreparadosNovedadesNomina`

## Principio rector

Esta fase no debe recalcular ni reinterpretar nada.

Debe limitarse a:

- seleccionar los campos utiles
- normalizar el contrato de salida
- marcar la via operativa esperada

## Regla de inclusion

Deben entrar todos los registros clasificados, tanto:

- `LISTO_AUTORREGISTRO`
- `REQUIERE_VALIDACION_USUARIO`

No debe excluirse ninguno en esta fase.

La diferencia debe quedar en el campo operativo de estado.

## Contrato V1 de salida

Formato por item en `gListaRegistrosRegistroNovedadesNomina`:

```text
IdEjecucion|ArchivoOrigen|HojaOrigen|FilaExcel|NombreTercero|IdTercero|Concepto|IdConcepto|Valor|Dias|Observaciones|OrigenResolucionConcepto|EstadoPreparacionRegistro|RequiereValidacionUsuario|MotivoValidacion|__END__
```

## Significado de columnas operativas

- `NombreTercero`
  - nombre del tercero a registrar

- `IdTercero`
  - identificacion del tercero a registrar

- `Concepto`
  - descripcion final del concepto sistema

- `IdConcepto`
  - codigo final del concepto sistema

- `Valor`
  - valor monetario final del registro
  - debe salir de `ValorConceptoFuente`

- `Dias`
  - cantidad de dias relevante para el registro cuando venga disponible
  - si no existe soporte, debe quedar vacio

- `Observaciones`
  - texto operativo corto para apoyar el registro
  - debe salir preferiblemente de `TextoFragmentoRelacionado`
  - si no existe, usar `NovedadTextoOriginal`

- `OrigenResolucionConcepto`
  - indica si el concepto final quedo por regla o por IA
  - valores V1:
    - `REGLA`
    - `IA`

- `EstadoPreparacionRegistro`
  - `LISTO_AUTORREGISTRO`
  - `REQUIERE_VALIDACION_USUARIO`

- `RequiereValidacionUsuario`
  - `True` o `False`

- `MotivoValidacion`
  - motivo corto y operativo

## Reglas de armado V1

### Regla 1. Registro listo para autorregistro

Si el item clasificado viene como `LISTO_AUTORREGISTRO`:

- `EstadoPreparacionRegistro = LISTO_AUTORREGISTRO`
- `RequiereValidacionUsuario = False`
- `MotivoValidacion = ""`
- `ObservacionOperativa = "Registro listo para carga automatica"`

### Regla 2. Registro que requiere validacion

Si el item clasificado viene como `REQUIERE_VALIDACION_USUARIO`:

- `EstadoPreparacionRegistro = REQUIERE_VALIDACION_USUARIO`
- `RequiereValidacionUsuario = True`
- `MotivoValidacion = MotivoClasificacionRegistro`
- `ObservacionOperativa = "Validar antes de registrar en sistema"`

### Regla 3. Valor

`Valor` debe salir directamente de:

- `ValorConceptoFuente`

No debe recalcularse en esta fase.

### Regla 4. Concepto final

El concepto a exponer debe ser el final ya consolidado:

- `IdConcepto = CodigoSistemaFinal`
- `Concepto = ConceptoSistemaFinal`

### Regla 5. Dias

`Dias` debe salir de:

- `DiasMencionadosIA`

Si no existe dato, dejar vacio.

### Regla 6. Observaciones

`Observaciones` debe poblarse asi:

1. si existe `TextoFragmentoRelacionado`, usarlo
2. si no existe, usar `NovedadTextoOriginal`
3. si tampoco existe, dejar vacio

### Regla 7. Origen de resolucion del concepto

`OrigenResolucionConcepto` debe poblarse asi:

- `IA` si el estado final de resolucion de catalogo es `SUGERIDO_IA`
- `REGLA` en cualquier otro caso

No debe exponerse aqui el detalle de candidatos, catalogo base o sugerencia IA.

## Campos que deliberadamente no deben pasar a la salida operativa V1

Estos campos si sirven en conciliacion, pero no deben salir a la cola operativa inicial:

- `BaseResolucionCatalogo`
- `MetodoResolucionCatalogoFinal`
- `ConfianzaResolucionCatalogoFinal`
- `ObservacionResolucionCatalogoFinal`
- `MetodoConciliacion`
- `ConfianzaConciliacion`
- `FuenteAutoritativa`
- `PermiteAutoRegistro`
- `MotivoRevision`
- campos crudos de catalogo IA
- `LlaveRegistroAnalisis`
- `Ciudad`
- `Area`
- `TipoRegistroSistema`
- `RequiereCalculo`
- `OrigenAnalisis`
- `ObservacionOperativa`

Si luego hacen falta, se agregan por decision puntual.

## Hoja Excel objetivo sugerida

La persistencia de esta salida debe ir en una hoja nueva, por ejemplo:

- `registroNovedadesNomina`

No debe escribirse encima de:

- `conciliacionNovedadesNomina`
- `novedadesTexto`
- `registrosFuenteNovedadesNomina`

## Columnas sugeridas en Excel V1

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

## Variante posterior esperada

Mas adelante esta hoja puede crecer con columnas de ejecucion real en sistema, por ejemplo:

- `EstadoRegistroSistema`
- `FechaHoraRegistroSistema`
- `ResultadoRegistroSistema`
- `ObservacionRegistroSistema`

Pero eso no debe entrar en la V1.

## Criterio de calidad

Si un usuario abre esta hoja, debe poder entender rapido:

- quien es el tercero
- que concepto se pretende registrar
- por que valor
- cuantos dias aplica si corresponde
- si el concepto quedo por regla o por IA
- si puede dejar correr el robot o si debe validar antes

Si la hoja requiere leer conciliacion para entenderla, entonces la salida operativa quedo mal.
