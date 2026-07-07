# SF_AnalizarNovedadTextoNomina

Documento base del subflujo `SF_AnalizarNovedadTextoNomina` para el RPA de registro de novedades en aplicativo de nomina.

## Objetivo

Consumir `gListaNovedadesTextoPendientes` y producir una salida estructurada de novedades textuales analizadas, apoyandose en IA controlada para:

- segmentar multiples novedades contenidas en un mismo texto
- extraer fragmentos relevantes
- identificar montos, dias u otras señales utiles
- marcar si un fragmento parece calculable o requiere revision

Este subflujo no debe autorizar registro automatico ni resolver codigos finales del sistema.

## Motivacion

Las observaciones de nomina pueden contener:

- una sola novedad simple
- varias novedades mezcladas en el mismo texto
- combinaciones de conceptos respaldados y no respaldados
- montos monetarios sin columna asociada
- eventos calculables como incapacidades o vacaciones

Intentar resolver esto con heuristica amplia en PAD seria fragil y costoso de mantener. Por eso esta fase se desplaza a IA controlada con alcance acotado.

## Relacion con subflujos previos y posteriores

- `SF_ExpandirConceptosNomina` preserva `gListaNovedadesTextoPendientes`
- `SF_AnalizarNovedadTextoNomina` segmenta y estructura esas pendientes textuales
- `SF_ConciliarItemsNomina` cruza el analisis textual con los conceptos explicitos y decide respaldo, conflicto o revision
- `SF_ResolverCatalogoConceptosNomina` opera despues sobre items ya conciliados

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

- `gEstadoAnalisisNovedadTextoNomina` `Text`
- `gMensajeError` `Text`
- `gListaNovedadesTextoAnalizadas` `List`
- `gTotalNovedadesTextoAnalizadas` `Number`
- `gHayNovedadesTextoAnalizadas` `Boolean`

## Contrato exacto de salida en PAD

Formato recomendado por item de `gListaNovedadesTextoAnalizadas`:

```text
ArchivoOrigen|HojaOrigen|QuincenaHoja|FilaExcel|Ciudad|NombreEmpleado|CedulaFuente|Area|SueldoBaseTexto|AuxExtralegalBaseTexto|NovedadTextoOriginal|TieneConceptosExplicitos|MotivoPendienteTexto|IndiceFragmento|TextoFragmento|TipoSugeridoIA|MontoMencionado|DiasMencionados|RequiereCalculo|TieneRespaldoEstructuralSugerido|ConfianzaIA|RequiereRevision|ObservacionTecnica
```

## Rol exacto de la IA

La IA en este subflujo solo debe:

- separar el texto libre en uno o varios fragmentos de novedad
- sugerir un tipo de fragmento
- extraer montos mencionados
- extraer dias mencionados
- sugerir si el fragmento parece calculable
- sugerir si el fragmento parece tener respaldo estructural o no

La IA no debe:

- autorizar registro automatico
- inventar codigos del sistema
- decidir conciliacion final
- aprobar montos sin respaldo

## Tipos de salida sugeridos por IA

No se deben tratar como conceptos finales del sistema. Son solo etiquetas intermedias de analisis, por ejemplo:

- `INCAPACIDAD`
- `VACACIONES`
- `LICENCIA`
- `COMISION`
- `BONO`
- `DEDUCCION`
- `OBSERVACION_GENERAL`
- `NO_CLASIFICADO`

## Criterio de implementacion

### Fase 1. Filtro minimo previo

Antes de invocar IA, solo aplicar reglas tecnicas minimas:

- validar que la lista no este vacia
- validar formato de entrada
- omitir filas con texto vacio si aun llegaran por error

### Fase 2. Analisis IA controlado

Para cada pendiente textual:

- enviar prompt con contexto de fila
- pedir salida estructurada
- no permitir generacion libre fuera del esquema solicitado

### Fase 3. Persistencia de resultado

- agregar cada fragmento analizado a `gListaNovedadesTextoAnalizadas`
- conservar siempre el texto original completo
- marcar `RequiereRevision = True` por defecto en version 1

## Prompting esperado

La solicitud a IA debe estar cerrada a un esquema. Por ejemplo:

- texto original
- nombre empleado
- fila Excel
- si la fila ya tenia conceptos explicitos
- monto o conceptos explicitos detectados en columnas
- pedir respuesta estructurada en JSON o formato tabular controlado

## Logging esperado

- inicio del subflujo
- total de pendientes textuales recibidas
- total de filas enviadas a IA
- total de fragmentos estructurados recibidos
- total de filas que requirieron revision
- resumen final

No se recomienda loggear la respuesta completa de IA salvo modo debug controlado.

## Manejo de errores

Seguir el patron ya usado:

- `BLOCK ... ON BLOCK ERROR`
- bandera `gEsErrorControlado`
- `THROW ERROR`
- captura final con `ERROR => gMensajeError`
- escritura en `gArchivoLog`

## Recomendacion de implementacion

Version 1:

1. consumir pendientes textuales
2. llamar IA con esquema cerrado
3. devolver fragmentos estructurados
4. marcar todos como `RequiereRevision = True`
5. dejar la conciliacion final fuera de este subflujo

Version 2:

1. cruzar automaticamente con conceptos explicitos de la misma fila
2. reducir revision manual en casos simples
3. mejorar prompts y validaciones de salida
