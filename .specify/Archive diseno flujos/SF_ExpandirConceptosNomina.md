# SF_ExpandirConceptosNomina

Documento base del subflujo `SF_ExpandirConceptosNomina` para el RPA de registro de novedades en aplicativo de nomina.

## Objetivo

Tomar cada fila cruda leida desde una hoja valida de nomina y:

- convertir en registros fuente unicamente los conceptos materializados en columnas con valor
- preservar en una salida separada las novedades textuales relevantes que no deban perderse

## Motivacion

En los archivos de novedades de nomina:

- una fila representa el contexto de un empleado para una quincena
- una fila puede traer cero, uno o varios conceptos con valor
- una misma novedad textual puede aplicar a uno, dos o mas conceptos del empleado

Por tanto, la unidad de procesamiento posterior no debe ser solo la fila Excel sino:

- `empleado + concepto + valor + contexto + novedad asociada`

Adicionalmente:

- no toda novedad viene materializada en una columna con valor
- no todo valor mencionado en texto debe registrarse automaticamente
- el subflujo no debe inventar conceptos a partir del texto
- pero tampoco debe dejar perder filas con novedad relevante

## Relacion con subflujos previos y posteriores

- `SF_ValidarEstructuraNovedadesNomina` determina hojas validas y metadatos de cabecera
- `SF_LeerNovedadesNominaHojasValidas` produce filas crudas de empleados
- `SF_ExpandirConceptosNomina` convierte filas crudas en registros fuente materializados y preserva pendientes textuales
- `SF_AnalizarNovedadTextoNomina` analiza las pendientes textuales en una fase separada apoyada en IA controlada
- `SF_ConciliarItemsNomina` valida respaldo estructural, calculable o textual
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
- `gListaRegistrosFuenteNovedadesNomina` `List`
- `gTotalRegistrosFuenteNovedadesNomina` `Number`
- `gHayRegistrosFuenteNovedadesNomina` `Boolean`
- `gListaNovedadesTextoPendientes` `List`
- `gTotalNovedadesTextoPendientes` `Number`
- `gHayNovedadesTextoPendientes` `Boolean`

Nota:

El contrato principal del subflujo sigue siendo material y estable:

- la lista de registros fuente solo contiene conceptos explicitos con valor en columna
- la lista de pendientes textuales solo preserva evidencia para fases posteriores

## Contrato exacto de salida en PAD

Mientras la implementacion siga basada en listas de texto delimitadas, se recomienda fijar desde ya el orden exacto de campos.

### Contrato de `gListaRegistrosFuenteNovedadesNomina`

Formato por item:

```text
ArchivoOrigen|HojaOrigen|QuincenaHoja|FilaExcel|Ciudad|NombreEmpleado|CedulaFuente|Area|SueldoBaseTexto|AuxExtralegalBaseTexto|ConceptoFuente|ColumnaOrigen|ValorTexto|NovedadTextoOriginal|NovedadAsociadaConcepto|MetodoAsociacionNovedad|ConfianzaAsociacion|RequiereRevision
```

### Contrato de `gListaNovedadesTextoPendientes`

Formato por item:

```text
ArchivoOrigen|HojaOrigen|QuincenaHoja|FilaExcel|Ciudad|NombreEmpleado|CedulaFuente|Area|SueldoBaseTexto|AuxExtralegalBaseTexto|NovedadTextoOriginal|TieneConceptosExplicitos|MotivoPendienteTexto|RequiereRevision
```

### Semantica de los ultimos campos de `gListaNovedadesTextoPendientes`

- `TieneConceptosExplicitos = True | False`
- `MotivoPendienteTexto = TEXTO_SIN_CONCEPTO_MATERIALIZADO | TEXTO_ADICIONAL_NO_RESPALDADO | TEXTO_REQUIERE_ANALISIS`
- `RequiereRevision = True | False`

## Unidad de salida recomendada

### Salida 1. Registros fuente materializados

Cada registro fuente deberia representar una posible novedad estructuralmente visible. Campos sugeridos:

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

Valores sugeridos para esta salida:

- `EsConceptoConValor = True`

### Salida 2. Novedades texto pendientes

Cada novedad pendiente debe preservar la fila para analisis posterior, sin convertirla todavia en concepto. Campos sugeridos:

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
- `TieneConceptosExplicitos`
- `MotivoPendienteTexto`
- `RequiereRevision`

Valores sugeridos para `MotivoPendienteTexto`:

- `TEXTO_SIN_CONCEPTO_MATERIALIZADO`
- `TEXTO_ADICIONAL_NO_RESPALDADO`
- `TEXTO_REQUIERE_ANALISIS`

## Regla exacta de emision de `gListaNovedadesTextoPendientes`

La emision de pendientes textuales debe ser cerrada y explicable.

1. tomar `NovedadTextoOriginal`
2. aplicar solo limpieza tecnica minima:
   - trim
   - reemplazo de saltos de linea por espacio
3. comparar contra una lista cerrada de valores no relevantes:
   - vacio
   - `-`
   - `sin novedad`
   - `sin novedad.`
4. si cae en esa lista:
   - no generar pendiente textual
5. si no cae en esa lista:
   - generar pendiente textual

## Regla de convivencia entre ambas salidas

Una misma fila puede producir:

- solo conceptos explicitos
- solo pendiente textual
- ambos

Ejemplos:

- fila con `DESCUENTOS = 30700` y texto `Descontar 30.700 plan exequial`
  - genera concepto explicito
  - no necesita pendiente adicional si el texto solo describe ese mismo respaldo

- fila con `DESCUENTOS = 30700` y texto `Descontar 30.700 plan exequial. Bono de cumplimiento no salarial 100.000`
  - genera concepto explicito por descuentos
  - genera pendiente textual por informacion adicional no respaldada

- fila sin valores y texto `Incapacidad 15 dias`
  - no genera concepto explicito
  - si genera pendiente textual

## Criterio minimo para `TieneConceptosExplicitos`

- `True` si la fila genero al menos un item en `gListaRegistrosFuenteNovedadesNomina`
- `False` si no genero ninguno

## Regla base de expansion

1. tomar una fila cruda de empleado
2. identificar las columnas de conceptos disponibles en esa hoja
3. detectar cuales de esas columnas tienen valor para esa fila
4. generar un concepto candidato por cada columna con valor
5. copiar el contexto del empleado a cada concepto candidato
   - incluyendo `SueldoBaseTexto` y `AuxExtralegalBaseTexto` como contexto fijo de control
6. conservar la novedad original completa a nivel de concepto candidato
7. completar, cuando sea posible, una `NovedadAsociadaConcepto`
8. evaluar si la fila debe agregarse tambien a `gListaNovedadesTextoPendientes`

## Tipos de salida que debe contemplar

### Conceptos explicitos

Casos en que una columna trae un valor concreto usable:

- descuentos
- bonos
- viaticos
- transporte

### Pendientes textuales

Casos en que el texto debe preservarse para analisis posterior:

- no hay columnas con valor, pero la novedad es distinta de `Sin novedad`
- hay conceptos explicitos detectados, pero el texto menciona informacion adicional no respaldada
- la fila requiere analisis posterior por negocio

Estos casos no deben convertirse aqui en conceptos derivados ni autorregistrarse.

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

## Regla de seguridad sobre montos en texto

Si la observacion contiene un monto monetario que no esta respaldado por:

- una columna con valor
- una regla formal de calculo
- una excepcion parametrizada

entonces la fila debe preservarse como pendiente textual o quedar preparada para conciliacion posterior, pero no como registro fuente directo a autorregistro.

## Regla minima para pendientes textuales

Sin introducir heuristica amplia, la regla minima sugerida es:

- si `NovedadTextoOriginal` esta vacia: no generar pendiente textual
- si `NovedadTextoOriginal` equivale a `Sin novedad`: no generar pendiente textual
- en cualquier otro caso: preservar pendiente textual

Una fase posterior definira si ese texto corresponde a evento derivable, conflicto, observacion informativa o revision manual.

## Reglas de seguridad para IA

- la IA no debe inventar conceptos
- la IA solo debe elegir entre conceptos activos detectados para esa fila
- si la confianza es baja, el concepto debe quedar marcado para revision
- el texto original de la novedad siempre debe conservarse

## Logging esperado

- inicio del subflujo
- cantidad de filas crudas recibidas
- cantidad de registros fuente generados
- cantidad de filas con un solo concepto
- cantidad de filas con multiples conceptos
- cantidad de pendientes textuales preservadas
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
2. generar registros fuente explicitos
3. copiar `NovedadTextoOriginal` a cada candidato
4. preservar pendientes textuales en una lista separada
5. separar lo conciliado de lo no respaldado en subflujos posteriores
6. dejar `NovedadAsociadaConcepto` igual al texto original en casos simples
7. incorporar logica de distribucion y luego IA solo para casos ambiguos
