# SF_ConciliarItemsNomina

Documento base del subflujo `SF_ConciliarItemsNomina` para el RPA de registro de novedades en aplicativo de nomina.

## Objetivo

Recibir los registros fuente materializados y los hallazgos textuales de nomina, y determinar si cada registro:

- tiene respaldo estructural suficiente para autorregistro
- requiere calculo previo
- solo existe como referencia textual
- presenta conflicto entre texto y columnas

## Motivacion

La fuente Excel puede contener varios patrones distintos:

- conceptos con valor visible en columna
- eventos descritos en la observacion pero sin valor porque se calculan despues
- montos mencionados en texto sin respaldo estructural
- conceptos que en Excel aparecen agregados y en sistema deben dividirse

Por eso antes de resolver catalogo y antes de registrar en la UI debe existir una capa de conciliacion.

## Alcance especifico de conciliacion

Este subflujo si debe encargarse de los casos mixtos. Es decir:

- cuando una fila trae uno o varios valores en columnas que ya generan registros fuente
- y esa misma fila trae texto adicional con novedades que no quedaron materializadas

La conciliacion debe:

- conservar los conceptos ya respaldados por columna
- identificar si el texto adicional complementa un concepto existente, describe un evento calculable o introduce un pendiente sin respaldo
- dejar trazabilidad comun por fila para que el usuario revise el conjunto y no piezas aisladas

## Relacion con subflujos previos y posteriores

- `SF_ExpandirConceptosNomina` genera registros fuente materializados
- `SF_AnalizarNovedadTextoNomina` segmenta y estructura pendientes textuales con apoyo de IA controlada
- `SF_ConciliarItemsNomina` clasifica respaldo y registrabilidad, incluyendo filas mixtas con conceptos en columnas y texto adicional no materializado
- `SF_ResolverCatalogoConceptosNomina` debe operar sobre registros ya conciliados

## Entrada esperada

- `gListaRegistrosFuenteNovedadesNomina` `List`
- `gTotalRegistrosFuenteNovedadesNomina` `Number`
- `gListaNovedadesTextoAnalizadas` `List`
- `gTotalNovedadesTextoAnalizadas` `Number`
- `gRutaArchivoAnalisisNovedades` `Text`
- `gIdEjecucion` `Text`
- `gArchivoLog` `Text`

## Salidas sugeridas

- `gEstadoConciliacionNomina` `Text`
- `gMensajeError` `Text`
- `gListaConciliacionNovedadesNomina` `List`
- `gTotalConciliacionNovedadesNomina` `Number`
- `gHayConciliacionNovedadesNomina` `Boolean`
- `gTotalConciliadosNomina` `Number`
- `gTotalDerivadosCalculablesNomina` `Number`
- `gTotalTextoSinRespaldoNomina` `Number`
- `gTotalConflictosNomina` `Number`

## Estados de conciliacion

- `CONCILIADO`
  - hay respaldo por columna y el item puede seguir al catalogo

- `DERIVADO_CALCULABLE`
  - no hay valor autoritativo en columna, pero existe una regla formal de calculo
  - ejemplo: incapacidad por dias a partir de sueldo base

- `TEXTO_SIN_RESPALDO`
  - el texto menciona concepto o monto, pero no existe respaldo en columna ni regla de calculo
  - no debe autorregistrarse

- `CONFLICTO`
  - texto y estructura discrepan o no cuadran entre si
  - ejemplo: texto menciona dos montos y la suma no coincide con la columna

## Regla principal de operacion

El texto de novedad no autoriza por si solo un valor monetario.

Solo se permite autorregistro si el item queda como:

- `CONCILIADO`
- `DERIVADO_CALCULABLE`

Si queda como:

- `TEXTO_SIN_RESPALDO`
- `CONFLICTO`

debe marcarse para revision manual.

## Campos sugeridos de salida

Cada item conciliado debe conservar los campos originales y agregar al menos:

- `EstadoConciliacion`
- `FuenteAutoritativa`
- `PermiteAutoRegistro`
- `MotivoRevision`
- `RequiereCalculo`
- `MetodoConciliacion`
- `ConfianzaConciliacion`

Adicionalmente, para trabajo operativo en Excel de revision, se recomienda agregar:

- `IdEjecucion`
- `LlaveFilaFuente`
- `OrigenAnalisis`
- `EstadoRevisionUsuario`
- `DecisionUsuario`
- `ObservacionUsuario`

Valores sugeridos:

- `FuenteAutoritativa = COLUMNA | CALCULO | TEXTO | MIXTA`
- `PermiteAutoRegistro = True | False`
- `MetodoConciliacion = REGLA | PATRON | IA | MANUAL`
- `ConfianzaConciliacion = ALTA | MEDIA | BAJA`

## Contrato V1 de salida en PAD

Para la primera version conviene dejar un contrato tabular, plano y trazable.

Formato por item en `gListaConciliacionNovedadesNomina`:

```text
IdEjecucion|LlaveFilaFuente|OrigenAnalisis|ArchivoOrigen|HojaOrigen|FilaExcel|Ciudad|NombreEmpleado|CedulaFuente|Area|SueldoBaseTexto|AuxExtralegalBaseTexto|ConceptoFuenteExcel|ColumnaOrigenExcel|ValorConceptoFuente|NovedadTextoOriginal|TextoFragmentoRelacionado|TipoSugeridoIA|MontoMencionadoIA|DiasMencionadosIA|EstadoConciliacion|FuenteAutoritativa|PermiteAutoRegistro|RequiereCalculo|MetodoConciliacion|ConfianzaConciliacion|MotivoRevision|EstadoRevisionUsuario|DecisionUsuario|ObservacionUsuario
```

## Llave minima de cruce

La conciliacion debe agrupar por una llave comun de fila fuente:

```text
ArchivoOrigen|HojaOrigen|FilaExcel
```

Sobre esa llave:

- uno o varios registros fuente pueden venir de `gListaRegistrosFuenteNovedadesNomina`
- cero, uno o varios fragmentos textuales pueden venir de `gListaNovedadesTextoAnalizadas`

## Reglas V1 de conciliacion

La primera version no debe sobredisenarse. Debe operar con reglas cerradas.

### Regla 1. Registro fuente sin texto analizado asociado

Si una llave de fila tiene registros fuente y no tiene fragmentos IA:

- emitir una fila por cada registro fuente
- `EstadoConciliacion = CONCILIADO`
- `FuenteAutoritativa = COLUMNA`
- `PermiteAutoRegistro = True`
- `RequiereCalculo = False`
- `MetodoConciliacion = REGLA`
- `ConfianzaConciliacion = ALTA`

### Regla 2. Texto analizado sin registros fuente

Si una llave de fila no tiene registros fuente y si tiene fragmentos IA:

- emitir una fila por cada fragmento IA
- si `RequiereCalculo = True` y el tipo es calculable, por ejemplo `INCAPACIDAD`, marcar:
  - `EstadoConciliacion = DERIVADO_CALCULABLE`
  - `FuenteAutoritativa = CALCULO`
  - `PermiteAutoRegistro = True`
- en los demas casos marcar:
  - `EstadoConciliacion = TEXTO_SIN_RESPALDO`
  - `FuenteAutoritativa = TEXTO`
  - `PermiteAutoRegistro = False`

### Regla 3. Fila mixta con registros fuente y fragmentos IA

Si una misma llave tiene ambos:

- emitir primero una fila por cada registro fuente
- si existen fragmentos con `TieneRespaldoEstructuralSugerido = True` y claramente describen el mismo registro, dejar:
  - `EstadoConciliacion = CONCILIADO`
  - `FuenteAutoritativa = MIXTA`
- si existen fragmentos adicionales no respaldados por columna:
  - emitir fila adicional por fragmento
  - `EstadoConciliacion = TEXTO_SIN_RESPALDO`
  - `PermiteAutoRegistro = False`

### Regla 4. Conflicto

Si el texto menciona un monto explicito y existe un registro fuente en la misma fila, pero el monto no coincide materialmente:

- `EstadoConciliacion = CONFLICTO`
- `FuenteAutoritativa = MIXTA`
- `PermiteAutoRegistro = False`
- `MetodoConciliacion = REGLA`
- `ConfianzaConciliacion = MEDIA` o `BAJA`

## Hoja Excel objetivo

La salida conciliada no debe escribirse en `novedadesTexto` ni en `registrosFuenteNovedadesNomina`.

Debe persistirse en una hoja dedicada:

- `conciliacionNovedadesNomina`

## Subflujo hermano de persistencia

Para mantener el patron ya adoptado, `SF_ConciliarItemsNomina` no debe escribir directamente al Excel.

Debe existir un subflujo separado:

- `SF_EscribirConciliacionNovedadesNominaExcel`

Ese subflujo debe:

- tomar `gListaConciliacionNovedadesNomina`
- escribirla en la hoja `conciliacionNovedadesNomina`
- crear la hoja desde `Sheet1` si aun no existe
- escribir cabeceras V1
- dejar columnas de revision humana inicializadas

## Casos iniciales a cubrir

### Caso 1. Descuento con valor conciliado

- columna `DESCUENTOS = 30700`
- texto: `Descontar 30.700 plan exequial`

Resultado:

- `CONCILIADO`
- `FuenteAutoritativa = COLUMNA`
- `PermiteAutoRegistro = True`

### Caso 2. Bono mencionado solo en texto

- columna bono vacia
- texto: `Bono de cumplimiento no salarial 100.000`

Resultado:

- `TEXTO_SIN_RESPALDO`
- `FuenteAutoritativa = TEXTO`
- `PermiteAutoRegistro = False`

### Caso 3. Incapacidad sin valor en columnas

- no hay columna con valor
- texto: `Incapacidad 15 dias`
- existe sueldo base

Resultado:

- `DERIVADO_CALCULABLE`
- `FuenteAutoritativa = CALCULO`
- `PermiteAutoRegistro = True`
- `RequiereCalculo = True`

### Caso 4. Monto en texto que no cuadra con columna

Resultado:

- `CONFLICTO`
- `PermiteAutoRegistro = False`

## Uso de IA

La IA en esta fase solo debe actuar como fallback controlado:

- para clasificar texto ambiguo ya estructurado
- para proponer asociacion entre una lista cerrada de candidatos
- para sugerir si hay conflicto o falta respaldo

La IA no debe autorizar montos no respaldados.

## Logging esperado

- inicio del subflujo
- total de items recibidos
- total conciliados
- total derivados calculables
- total texto sin respaldo
- total conflictos
- resumen final

## Persistencia operativa recomendada

`SF_ConciliarItemsNomina` no debe crear un archivo aislado aparte. Debe escribir en el mismo archivo Excel de evidencias de la corrida, generado por `IdEjecucion` a partir de una plantilla base.

Hojas sugeridas para ese archivo:

- `registrosFuenteNovedadesNomina`
- `novedadesTexto`
- `conciliacionNovedadesNomina`
- `catalogoNomina`
- `revisionUI`

La hoja `conciliacionNovedadesNomina` debe quedar orientada a validacion humana, con listas o validaciones de datos definidas desde la plantilla base.

Su estructura definitiva debe ajustarse despues de estabilizar la hoja `novedadesTexto`, para no sobredisenar la primera version.

## Manejo de errores

Seguir el patron ya usado:

- `BLOCK ... ON BLOCK ERROR`
- bandera `gEsErrorControlado`
- `THROW ERROR`
- captura final con `ERROR => gMensajeError`
- escritura en `gArchivoLog`

## Recomendacion de implementacion

Version 1:

1. clasificar por reglas deterministicas
2. agrupar por `ArchivoOrigen|HojaOrigen|FilaExcel`
3. emitir conciliacion base para registros fuente
4. detectar incapacidad y otros calculables cerrados
5. detectar montos en texto y contrastarlos con columnas
6. marcar como no autorregistrable lo que solo viva en texto

Version 2:

1. integrar descomposicion de descuentos
2. integrar mas eventos calculables
3. usar IA solo para ambiguedades reales
