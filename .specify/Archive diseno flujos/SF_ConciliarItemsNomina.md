# SF_ConciliarItemsNomina

Documento base del subflujo `SF_ConciliarItemsNomina` para el RPA de registro de novedades en aplicativo de nomina.

## Objetivo

Recibir los items candidatos generados desde la lectura y expansion de nomina, y determinar si cada item:

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

## Relacion con subflujos previos y posteriores

- `SF_ExpandirConceptosNomina` genera items candidatos
- `SF_AnalizarNovedadTextoNomina` segmenta y estructura pendientes textuales con apoyo de IA controlada
- `SF_ConciliarItemsNomina` clasifica respaldo y registrabilidad
- `SF_DescomponerConceptosNomina` puede operar antes o despues segun tipo de item
- `SF_ResolverCatalogoConceptosNomina` debe operar sobre items ya conciliados

## Entrada esperada

- `gListaConceptosNominaCandidatos` `List`
- `gTotalConceptosNominaCandidatos` `Number`
- `gListaNovedadesTextoAnalizadas` `List`
- `gTotalNovedadesTextoAnalizadas` `Number`
- `gArchivoLog` `Text`

## Salidas sugeridas

- `gEstadoConciliacionNomina` `Text`
- `gMensajeError` `Text`
- `gListaItemsNominaConciliados` `List`
- `gTotalItemsNominaConciliados` `Number`
- `gTotalItemsNominaDerivadosCalculables` `Number`
- `gTotalItemsNominaTextoSinRespaldo` `Number`
- `gTotalItemsNominaConflicto` `Number`

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

Valores sugeridos:

- `FuenteAutoritativa = COLUMNA | CALCULO | TEXTO | MIXTA`
- `PermiteAutoRegistro = True | False`
- `MetodoConciliacion = REGLA | PATRON | IA | MANUAL`
- `ConfianzaConciliacion = ALTA | MEDIA | BAJA`

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
2. detectar `Sin novedad`
3. detectar incapacidad
4. detectar montos en texto y contrastarlos con columnas
5. marcar como no autorregistrable lo que solo viva en texto

Version 2:

1. integrar descomposicion de descuentos
2. integrar mas eventos calculables
3. usar IA solo para ambiguedades reales
