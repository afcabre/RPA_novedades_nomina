# SF_ExpandirConceptosNomina

Documento base del subflujo `SF_ExpandirConceptosNomina` para el RPA de registro de novedades en aplicativo de nomina.

## Objetivo

Tomar cada fila cruda leida desde una hoja valida de nomina y convertirla en uno o varios conceptos candidatos registrables por empleado, preservando tanto el valor del concepto como la novedad original de la fila y la novedad asociada a cada concepto.

## Motivacion

En los archivos de novedades de nomina:

- una fila representa el contexto de un empleado para una quincena
- una fila puede traer cero, uno o varios conceptos con valor
- una misma novedad textual puede aplicar a uno, dos o mas conceptos del empleado

Por tanto, la unidad de procesamiento posterior no debe ser solo la fila Excel sino:

- `empleado + concepto + valor + contexto + novedad asociada`

## Relacion con subflujos previos y posteriores

- `SF_ValidarEstructuraNovedadesNomina` determina hojas validas y metadatos de cabecera
- `SF_LeerNovedadesNominaHojasValidas` produce filas crudas de empleados
- `SF_ExpandirConceptosNomina` convierte filas crudas en conceptos candidatos
- luego subflujos posteriores deben encargarse de:
  - resolver equivalencias de conceptos por empresa
  - resolver empleados
  - decidir registrabilidad
  - registrar en sistema

## Entrada conceptual esperada

La entrada ideal de este subflujo es una coleccion de filas crudas con al menos:

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
- `CabeceraHojaTexto`
- `MapaColumnasCabecera`
- `ValoresFilaPorColumna`

Nota:

En una primera version PAD, esto puede implementarse con listas de texto delimitadas si aun no conviene pasar a estructuras mas ricas.

## Salidas sugeridas

- `gEstadoExpansionConceptosNomina` `Text`
- `gMensajeError` `Text`
- `gListaConceptosNominaCandidatos` `List`
- `gTotalConceptosNominaCandidatos` `Number`
- `gHayConceptosNominaCandidatos` `Boolean`

## Unidad de salida recomendada

Cada concepto candidato deberia representar una posible novedad registrable. Campos sugeridos:

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
- `ConceptoFuente`
- `ColumnaOrigen`
- `ValorTexto`
- `NovedadTextoOriginal`
- `NovedadAsociadaConcepto`
- `MetodoAsociacionNovedad`
- `ConfianzaAsociacion`
- `EsConceptoConValor`
- `RequiereRevision`

## Regla base de expansion

1. tomar una fila cruda de empleado
2. identificar las columnas de conceptos disponibles en esa hoja
3. detectar cuales de esas columnas tienen valor para esa fila
4. generar un concepto candidato por cada columna con valor
5. copiar el contexto del empleado a cada concepto candidato
   - incluyendo `SueldoBaseTexto` y `AuxExtralegalBaseTexto` como contexto fijo de control
6. conservar la novedad original completa a nivel de concepto candidato
7. completar, cuando sea posible, una `NovedadAsociadaConcepto`

## Niveles de novedad que se deben preservar

Para no perder informacion, la novedad debe vivir en dos niveles:

### Nivel fila

- `NovedadTextoOriginal`

Es el texto tal como aparece en la fila del Excel.

### Nivel concepto

- `NovedadAsociadaConcepto`

Es la parte de la novedad que efectivamente se relaciona con ese concepto candidato.

## Estrategia de asociacion de novedad

La asociacion de novedad a concepto debe hacerse por etapas:

### Etapa 1. Reglas deterministicas

Usar primero reglas simples y explicables:

- si solo hay un concepto con valor, asociar toda la novedad a ese concepto
- si la novedad esta vacia, dejar `NovedadAsociadaConcepto` vacia
- si la novedad contiene palabras clave exactas de un concepto, asociarla a ese concepto

### Etapa 2. Reglas por patron

Aplicar patrones de texto cuando existan:

- montos
- fechas
- palabras clave de vacaciones, comision, descuento, viatico, etc.

### Etapa 3. IA como fallback controlado

Si la fila tiene varios conceptos con valor y la novedad es ambigua:

- enviar a IA la novedad original
- enviar la lista cerrada de conceptos activos para esa fila
- enviar los valores de cada concepto
- pedir asignacion entre opciones cerradas, no generacion libre

Campos recomendados para rastreo:

- `MetodoAsociacionNovedad = REGLA | PATRON | IA | MANUAL`
- `ConfianzaAsociacion = ALTA | MEDIA | BAJA`
- `RequiereRevision = True/False`

## Reglas de seguridad para IA

- la IA no debe inventar conceptos
- la IA solo debe elegir entre conceptos activos detectados para esa fila
- si la confianza es baja, el concepto debe quedar marcado para revision
- el texto original de la novedad siempre debe conservarse

## Logging esperado

- inicio del subflujo
- cantidad de filas crudas recibidas
- cantidad de conceptos candidatos generados
- cantidad de filas con un solo concepto
- cantidad de filas con multiples conceptos
- cantidad de casos que requeririan asociacion por IA
- resumen final

## Manejo de errores

Seguir el mismo patron de los demas subflujos:

- `BLOCK ... ON BLOCK ERROR`
- bandera `gEsErrorControlado`
- `THROW ERROR`
- captura final con `ERROR => gMensajeError`
- escritura en `gArchivoLog`

## Nota de implementacion incremental

La primera version no necesita usar IA todavia. Se recomienda implementar por fases:

1. detectar conceptos con valor por fila
2. generar conceptos candidatos
3. copiar `NovedadTextoOriginal` a cada candidato
4. dejar `NovedadAsociadaConcepto` igual al texto original en casos simples
5. incorporar logica de distribucion y luego IA solo para casos ambiguos
