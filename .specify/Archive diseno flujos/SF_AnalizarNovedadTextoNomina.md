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
- `pProveedorIA` `Text`
- `pModeloIA` `Text`
- `pEndpointIA` `Text`
- `pCredentialAliasIA` `Text`
- `pTimeoutIAsegundos` `Number`
- `pDebugIA` `Boolean`

## Contrato exacto de entrada en PAD

Formato esperado por item de `gListaNovedadesTextoPendientes`:

```text
ArchivoOrigen|HojaOrigen|QuincenaHoja|FilaExcel|Ciudad|NombreEmpleado|CedulaFuente|Area|SueldoBaseTexto|AuxExtralegalBaseTexto|NovedadTextoOriginal|TieneRegistrosFuenteMaterializados|MotivoPendienteTexto|RequiereRevision
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
ArchivoOrigen|HojaOrigen|QuincenaHoja|FilaExcel|Ciudad|NombreEmpleado|CedulaFuente|Area|SueldoBaseTexto|AuxExtralegalBaseTexto|NovedadTextoOriginal|TieneRegistrosFuenteMaterializados|MotivoPendienteTexto|IndiceFragmento|TextoFragmento|TipoSugeridoIA|MontoMencionado|DiasMencionados|RequiereCalculo|TieneRespaldoEstructuralSugerido|ConfianzaIA|RequiereRevision|ObservacionTecnica
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

## Persistencia operativa recomendada

El resultado de este subflujo ya no debe vivir solo en memoria ni en el log tecnico. La salida util para operacion debe persistirse en el archivo Excel de evidencias de la corrida, para revision, conciliacion posterior y reutilizacion del analisis.

Formato recomendado para la primera version:

- un unico archivo Excel por `IdEjecucion`
- el archivo debe generarse a partir de una plantilla base y no debe sobreescribir esa plantilla
- la plantilla base aprobada para este frente es `Plantilla_Analisis_Novedades.xlsx`
- el archivo de corrida debe nombrarse `Analisis_Novedades_<MES>_<ANO>_<PERIODO>_<IdEjecucion>.xlsx`
- `PERIODO` debe normalizarse como `1Q`, `2Q` o `MC`
- una hoja por tipo de analisis o salida
- para este subflujo, la primera hoja objetivo sera `novedadesTexto`
- una fila por fragmento analizado
- columnas V1 recomendadas:
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

- columnas propuestas para version posterior:
  - `VersionPromptIA`
  - `HashComparacion` o llave equivalente

El log debe conservar solo resumenes y uno o dos ejemplos debug.

## Validaciones de usuario sugeridas para V1

En la hoja `novedadesTexto`, la plantilla base puede incluir:

- `EstadoRevisionUsuario`: `Pendiente`, `Revisado`
- `DecisionUsuario`: `Aceptar`, `Ajustar`, `Descartar`

## Reutilizacion del analisis

Como el analisis IA es costoso, no conviene repetirlo ciegamente en cada corrida.

Por ello, la evidencia persistida en Excel debe permitir:

- conservar el resultado de IA por registro fuente
- identificar si una fila ya fue analizada anteriormente
- decidir si se reutiliza el analisis previo o si debe recalcularse

La llave minima sugerida para comparar reutilizacion es:

- `ArchivoOrigen`
- `HojaOrigen`
- `FilaExcel`
- `NovedadTextoOriginal`
- `VersionPromptIA`

## Evaluacion actual del prompt

Con la primera corrida exitosa, el prompt actual ya demuestra valor en tres frentes:

- clasifica bien eventos simples como `COMISION` e `INCAPACIDAD`
- extrae montos explicitos cuando aparecen en texto
- marca `RequiereCalculo = True` para eventos como incapacidades sin valor materializado

Sin embargo, para la siguiente fase conviene endurecer el contrato semantico del prompt para hacerlo mas util a la conciliacion posterior.

Se recomienda pedir adicionalmente que el modelo razone dentro de este marco:

- `TEXTO_SIN_RESPALDO`: el texto menciona una novedad, pero no hay evidencia material suficiente para autorregistro
- `EVENTO_CALCULABLE`: el texto describe algo que podria calcularse por regla posterior, como incapacidad o vacaciones sin monto final
- `INDICIO_CONCEPTO_FALTANTE`: el texto sugiere que deberia existir un concepto material que no vino en columnas

Estas etiquetas no deben reemplazar `TipoSugeridoIA`; deben complementar la interpretacion posterior en conciliacion.

## Validacion humana recomendada

No se recomienda resolver la validacion humana con formularios complejos embebidos en PAD. Las limitaciones practicas son:

- baja trazabilidad de decisiones
- dificultad para revisar muchos registros en una corrida
- mantenimiento costoso de interfaces interactivas

La alternativa mas estable para la primera version es:

1. `SF_ExpandirConceptosNomina` genera conceptos materiales y pendientes textuales
2. `SF_AnalizarNovedadTextoNomina` genera fragmentos estructurados con IA
3. `SF_ConciliarItemsNomina` cruza ambos resultados y prepara o actualiza el mismo Excel de evidencias de la corrida
4. el usuario revisa ese Excel y marca decisiones
5. un subflujo posterior consume el archivo validado para resolver catalogo y registrar en UI

Excel ofrece mejor equilibrio entre auditabilidad, volumen, edicion manual y reutilizacion de resultados que una validacion inline en PAD.

## Configuracion tecnica esperada

La configuracion del analisis IA debe vivir como un bloque `analisisNovedadTextoIA` dentro de `config.json`, ubicado en `C:\Dev\Aspraco\config`, y no debe contener la API key en texto plano.

Campos sugeridos:

- `proveedorIA`
- `modeloIA`
- `endpointIA`
- `credentialAliasIA`
- `timeoutIAsegundos`
- `debugIA`

La API key debe obtenerse desde Windows Credential Manager usando el alias configurado. Para la primera implementacion acordada:

- `credentialAliasIA = Aspraco_Mistral_API`
- `username` del credential: valor generico, por ejemplo `Aspraco`
- `password` del credential: API key real

Ejemplo de estructura:

```json
{
  "analisisNovedadTextoIA": {
    "activo": true,
    "proveedorIA": "mistral",
    "modeloIA": "mistral-medium-latest",
    "endpointIA": "https://api.mistral.ai/v1/chat/completions",
    "credentialAliasIA": "Aspraco_Mistral_API",
    "timeoutIAsegundos": 60,
    "debugIA": false
  }
}
```

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
2. recuperar la API key desde Windows Credential Manager
3. invocar Mistral `POST /v1/chat/completions`
4. usar `response_format = { "type": "json_object" }`
5. convertir la respuesta a la salida PAD
6. si la invocacion falla o la respuesta no es usable, emitir un fallback conservador por fila

Version 2:

1. persistir la salida en archivo estructurado de trabajo
2. cruzar automaticamente con conceptos explicitos de la misma fila en `SF_ConciliarItemsNomina`
3. reducir revision manual en casos simples y bien respaldados
4. mejorar prompts y validaciones de salida

## Estado implementado en repo

La version actual implementada en `src/SF_AnalizarNovedadTextoNomina.txt` deja:

- validacion del contrato de entrada
- recuperacion de la API key desde Credential Manager usando `pCredentialAliasIA`
- invocacion real de Mistral con `pModeloIA` y `pEndpointIA`
- solicitud de salida JSON controlada
- conversion de la respuesta al contrato PAD
- fallback conservador por fila si la IA falla, responde vacio o entrega JSON invalido

El fallback actual preserva la fila y emite:

- `TipoSugeridoIA = NO_CLASIFICADO`
- `ConfianzaIA = NO_EJECUTADA`
- `RequiereRevision = True`
- `ObservacionTecnica = Error...` o motivo tecnico equivalente

## Siguiente evolucion prevista

Cuando se active la invocacion real de IA, `SF_AnalizarNovedadTextoNomina` debera:

1. leer la configuracion tecnica desde la llave `analisisNovedadTextoIA` de `config.json`
2. recuperar la API key desde Windows Credential Manager usando `pCredentialAliasIA`
3. invocar el proveedor configurado
4. validar la respuesta contra el contrato de salida PAD

## Nota de implementacion Mistral

La integracion se alineo con la documentacion oficial de Mistral para `POST /v1/chat/completions`, usando `response_format` en modo `json_object` y mensajes `system` + `user` para forzar una salida JSON controlada.

## Propuesta PAD pura

Para esta fase se prefiere una implementacion mayoritariamente con acciones PAD y no con PowerShell, porque:

- el manejo de `config.json` ya demostro ser mas simple en PAD nativo
- la depuracion en UI resulta mas clara
- el mantenimiento del flujo queda menos acoplado a scripts embebidos

### Bloques propuestos

#### 1. Validacion inicial

- `If` `gListaNovedadesTextoPendientes.Count = 0`
- `If` `pProveedorIA <> "mistral"`

#### 2. Lectura de credencial IA

Usar la accion nativa:

- `Get username and password from Windows Credential Manager`
  - `Credential name: %pCredentialAliasIA%`
  - Salidas:
    - `gUsuarioCredentialIA`
    - `gApiKeyIA`

Validar:

- `If` `IsEmpty(gApiKeyIA)`

#### 3. Parseo del item pendiente

Por cada `CurrentItem` en `gListaNovedadesTextoPendientes`:

- `Split text`
  - `Text: CurrentItem`
  - `Delimiter: |`
  - Salida:
    - `gPartesPendienteTextoActual`

Asignar:

- `gArchivoOrigenActual = gPartesPendienteTextoActual[0]`
- `gHojaOrigenActual = gPartesPendienteTextoActual[1]`
- `gQuincenaHojaActual = gPartesPendienteTextoActual[2]`
- `gFilaExcelActual = gPartesPendienteTextoActual[3]`
- `gCiudadActual = gPartesPendienteTextoActual[4]`
- `gNombreEmpleadoActual = gPartesPendienteTextoActual[5]`
- `gCedulaFuenteActual = gPartesPendienteTextoActual[6]`
- `gAreaActual = gPartesPendienteTextoActual[7]`
- `gSueldoBaseTextoActual = gPartesPendienteTextoActual[8]`
- `gAuxExtralegalBaseTextoActual = gPartesPendienteTextoActual[9]`
- `gNovedadTextoOriginalActual = gPartesPendienteTextoActual[10]`
- `gTieneRegistrosFuenteMaterializadosActual = gPartesPendienteTextoActual[11]`
- `gMotivoPendienteTextoActual = gPartesPendienteTextoActual[12]`
- `gRequiereRevisionEntradaActual = gPartesPendienteTextoActual[13]`

#### 4. Construccion del prompt

Definir dos variables de texto:

- `gSystemPromptIA`
- `gUserPromptIA`

`gSystemPromptIA` debe pedir:

- respuesta solo en JSON
- llave raiz `fragments`
- llaves por fragmento:
  - `indiceFragmento`
  - `textoFragmento`
  - `tipoSugeridoIA`
  - `montoMencionado`
  - `diasMencionados`
  - `requiereCalculo`
  - `tieneRespaldoEstructuralSugerido`
  - `confianzaIA`
  - `requiereRevision`
  - `observacionTecnica`

`gUserPromptIA` debe contener el contexto de la fila en JSON o texto estructurado con:

- archivo
- hoja
- quincena
- fila
- ciudad
- nombre
- cedula
- area
- sueldo base
- aux extralegal base
- novedad original
- tiene registros fuente materializados
- motivo del pendiente

#### 5. Construccion del payload JSON

Armar un texto JSON en variable, por ejemplo `gRequestBodyIA`, con esta estructura:

```json
{
  "model": "mistral-medium-latest",
  "response_format": { "type": "json_object" },
  "temperature": 0,
  "max_tokens": 700,
  "messages": [
    { "role": "system", "content": "..." },
    { "role": "user", "content": "..." }
  ]
}
```

Valores variables:

- `model = %pModeloIA%`
- `content system = %gSystemPromptIA%`
- `content user = %gUserPromptIA%`

#### 6. Invocacion HTTP

Usar la accion PAD nativa equivalente a:

- `Invoke web service`
  - `Method: POST`
  - `URL: %pEndpointIA%`
  - Header `Authorization = Bearer %gApiKeyIA%`
  - Header `Content-Type = application/json`
  - Body `gRequestBodyIA`
  - Timeout `pTimeoutIAsegundos`
  - Salida:
    - `gResponseBodyIA`
    - opcionalmente `gStatusCodeIA`

Validar:

- status HTTP exitoso
- `gResponseBodyIA` no vacio

#### 7. Parseo de respuesta

- `Convert JSON to custom object`
  - `Json: gResponseBodyIA`
  - `CustomObject => gResponseJsonIA`

Extraer:

- `gMessageContentIA = %gResponseJsonIA.choices[0].message.content%`

Si `message.content` llega como texto JSON:

- `Convert JSON to custom object`
  - `Json: gMessageContentIA`
  - `CustomObject => gMessageContentJsonIA`

#### 8. Loop de fragmentos

Recorrer:

- `gMessageContentJsonIA.fragments`

Por cada fragmento:

- leer propiedades
- si alguna viene vacia, aplicar defaults:
  - `tipoSugeridoIA = NO_CLASIFICADO`
  - `confianzaIA = BAJA`
  - `requiereRevision = True`
  - `requiereCalculo = False`
  - `tieneRespaldoEstructuralSugerido = False`

Construir una linea PAD del contrato final:

```text
ArchivoOrigen|HojaOrigen|QuincenaHoja|FilaExcel|Ciudad|NombreEmpleado|CedulaFuente|Area|SueldoBaseTexto|AuxExtralegalBaseTexto|NovedadTextoOriginal|TieneRegistrosFuenteMaterializados|MotivoPendienteTexto|IndiceFragmento|TextoFragmento|TipoSugeridoIA|MontoMencionado|DiasMencionados|RequiereCalculo|TieneRespaldoEstructuralSugerido|ConfianzaIA|RequiereRevision|ObservacionTecnica
```

Agregar a:

- `gListaNovedadesTextoAnalizadas`

#### 9. Fallback conservador

Si la llamada falla, el JSON no parsea o no existe `fragments`, generar un unico registro con:

- `IndiceFragmento = 1`
- `TextoFragmento = gNovedadTextoOriginalActual`
- `TipoSugeridoIA = NO_CLASIFICADO`
- `MontoMencionado = ""`
- `DiasMencionados = ""`
- `RequiereCalculo = False`
- `TieneRespaldoEstructuralSugerido = False`
- `ConfianzaIA = NO_EJECUTADA`
- `RequiereRevision = True`
- `ObservacionTecnica = motivo tecnico del fallback`

### Siguiente paso recomendado

Antes de bajar este diseno a `src/SF_AnalizarNovedadTextoNomina.txt`, conviene capturar un ejemplo real de la accion PAD de:

- `Invoke web service`

Con ese ejemplo se aterriza el nombre exacto de los argumentos y se evita inventar sintaxis.
