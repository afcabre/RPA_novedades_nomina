# SF_ResolverCatalogoConceptosNominaIA

Documento base del subflujo `SF_ResolverCatalogoConceptosNominaIA` para el RPA de registro de novedades en aplicativo de nomina.

## Objetivo

Tomar la salida de `SF_ResolverCatalogoConceptosNomina` y solicitar apoyo IA solo para las filas que quedaron en:

- `AMBIGUO`
- `NO_ENCONTRADO`

Este subflujo no debe releer Excel ni reanalizar novedades texto. Su responsabilidad es sugerir un concepto del sistema a partir del catalogo real de la empresa y dejar una salida trazable para validacion posterior.

## Principio rector

La IA aqui no reemplaza el catalogo base ni la resolucion deterministica. Solo actua cuando la fase por reglas ya no pudo cerrar el caso con confianza suficiente.

Por tanto:

- no procesa `RESUELTO`
- no modifica el catalogo base
- no inventa codigos fuera de `conceptosSistema`
- no autoriza por si sola el registro final

## Relacion con subflujos previos y posteriores

- `SF_ResolverCatalogoConceptosNomina` deja la salida base por reglas
- `SF_EnriquecerConciliacionNovedadesNominaConCatalogo` persiste la resolucion base
- `SF_ResolverCatalogoConceptosNominaIA` toma solo casos pendientes de catalogo
- un subflujo posterior debe persistir la sugerencia IA sobre la misma evidencia Excel
- despues vendra la validacion humana y finalmente el registro UI

## Entrada esperada

- `gListaResolucionCatalogoNovedadesNomina` `List`
- `gTotalResolucionCatalogoNovedadesNomina` `Number`
- `gArchivoLog` `Text`
- `gRutaCatalogoConceptosNomina` `Text`

## Filtro de entrada

Este subflujo debe construir una lista de trabajo solo con filas cuya columna `EstadoResolucionCatalogo` venga en:

- `AMBIGUO`
- `NO_ENCONTRADO`

Las filas `RESUELTO` deben quedar por fuera.

## Configuracion independiente requerida

Esta etapa debe usar una configuracion distinta a `analisisNovedadTextoIA`.

Bloque sugerido en `config.json`:

```json
{
  "resolucionCatalogoIA": {
    "proveedorIA": "mistral",
    "modeloIA": "mistral-medium-latest",
    "endpointIA": "https://api.mistral.ai/v1/chat/completions",
    "credentialAliasIA": "Aspraco_Mistral_API",
    "timeoutIAsegundos": 60,
    "debugIA": false
  }
}
```

## Estrategias IA permitidas

### 1. `IA_CANDIDATOS`

Aplica cuando la fila base quedo `AMBIGUO` y ya trae candidatos reales configurados.

La IA recibe:

- datos de la fila
- texto de novedad
- tipo sugerido IA previo, si existe
- lista cerrada de codigos y conceptos candidatos

Y debe escoger solo entre esos candidatos.

### 2. `IA_CATALOGO`

Aplica cuando la fila quedo `NO_ENCONTRADO` o cuando no hay lista cerrada util.

La IA recibe:

- datos de la fila
- texto de novedad
- concepto fuente Excel
- tipo sugerido IA previo
- catalogo completo de la empresa

Y debe sugerir el mejor candidato posible del catalogo, siempre tomado de `conceptosSistema`.

## Regla de uso

Orden sugerido:

1. si `EstadoResolucionCatalogo = AMBIGUO` y hay candidatos configurados:
   - usar `IA_CANDIDATOS`
2. si `EstadoResolucionCatalogo = NO_ENCONTRADO`:
   - usar `IA_CATALOGO`
3. si `EstadoResolucionCatalogo = AMBIGUO` pero no hay lista de candidatos:
   - usar `IA_CATALOGO`

## Datos minimos que debe recibir la IA

- `IdEjecucion`
- `LlaveFilaFuente`
- `OrigenAnalisis`
- `ArchivoOrigen`
- `HojaOrigen`
- `FilaExcel`
- `NombreEmpleado`
- `ConceptoFuenteExcel`
- `ValorConceptoFuente`
- `NovedadTextoOriginal`
- `TextoFragmentoRelacionado`
- `TipoSugeridoIA`
- `MontoMencionadoIA`
- `DiasMencionadosIA`
- `BaseResolucionCatalogo`
- `EstadoResolucionCatalogo`
- `CandidatosCodigoSistema`
- `CandidatosConceptoSistema`
- `ObservacionResolucionCatalogo`

## Salida esperada

Se recomienda producir una nueva lista:

- `gListaResolucionCatalogoNovedadesNominaIA`

Cada item debe conservar la fila base y anexar estas columnas:

- `MetodoResolucionCatalogoIA`
- `CodigoSistemaSugeridoIA`
- `ConceptoSistemaSugeridoIA`
- `ConfianzaResolucionCatalogoIA`
- `RequiereValidacionUsuarioIA`
- `ObservacionResolucionCatalogoIA`

Valores sugeridos:

- `MetodoResolucionCatalogoIA = IA_CANDIDATOS | IA_CATALOGO`
- `ConfianzaResolucionCatalogoIA = ALTA | MEDIA | BAJA`
- `RequiereValidacionUsuarioIA = True`

## Salidas sugeridas del subflujo

- `gEstadoResolucionCatalogoNominaIA` `Text`
- `gMensajeError` `Text`
- `gListaResolucionCatalogoNovedadesNominaIA` `List`
- `gTotalResolucionCatalogoNominaIA` `Number`
- `gTotalIAAmbiguosProcesados` `Number`
- `gTotalIANoEncontradosProcesados` `Number`
- `gHayResolucionCatalogoNominaIA` `Boolean`

## Contrato de respuesta IA

La respuesta debe ser un JSON valido con una sola sugerencia por fila.

Estructura sugerida:

```json
{
  "metodoResolucionCatalogoIA": "IA_CANDIDATOS",
  "codigoSistemaSugeridoIA": "118",
  "conceptoSistemaSugeridoIA": "COMISIONES",
  "confianzaResolucionCatalogoIA": "ALTA",
  "requiereValidacionUsuarioIA": true,
  "observacionResolucionCatalogoIA": "La novedad menciona una comision explicita y el mejor candidato del catalogo es COMISIONES."
}
```

## Reglas estrictas para la IA

- nunca sugerir un codigo que no exista en el catalogo entregado
- nunca devolver mas de una sugerencia final por fila
- si no hay certeza, bajar confianza y exigir validacion humana
- si recibe candidatos cerrados, no salir de esa lista
- no inventar equivalencias de negocio no presentes en el catalogo

## Logging esperado

- inicio del subflujo
- total de filas `AMBIGUO`
- total de filas `NO_ENCONTRADO`
- total de filas enviadas a IA
- resumen final

No se recomienda loggear prompts completos salvo debug controlado.

## Implementacion sugerida V1

1. cargar configuracion `resolucionCatalogoIA` desde `config.json`
2. leer credencial IA desde Credential Manager
3. construir lista solo con `AMBIGUO` y `NO_ENCONTRADO`
4. por cada fila:
   - decidir `IA_CANDIDATOS` o `IA_CATALOGO`
   - construir prompt
   - invocar servicio web
   - validar JSON de respuesta
   - agregar salida enriquecida a `gListaResolucionCatalogoNovedadesNominaIA`
5. registrar resumen

## Alcance deliberadamente limitado

Esta etapa:

- no escribe aun en Excel
- no mezcla nuevamente conciliacion
- no modifica `gListaResolucionCatalogoNovedadesNomina`
- no registra nada en UI

Su salida debe ser una capa adicional de sugerencia IA para luego persistirse y validarse.
