# SF_ResolverCatalogoConceptosNomina

Documento base del subflujo `SF_ResolverCatalogoConceptosNomina` para el RPA de registro de novedades en aplicativo de nomina.

## Objetivo

Tomar la lista `gListaConciliacionNovedadesNomina` producida por `SF_ConciliarItemsNomina` y resolver, para cada fila conciliada, su equivalencia operativa dentro del sistema de nomina segun la empresa objetivo.

Este subflujo no debe registrar aun en el sistema. Su responsabilidad es dejar cada concepto candidato enriquecido con resultado de catalogo:

- resuelto
- ambiguo
- no encontrado

## Motivacion

En el Excel ya sabemos:

- quien es el empleado
- cual es el concepto fuente visible en la plantilla
- cual es el valor del concepto
- cual es la novedad textual asociada

Pero todavia no sabemos:

- con que codigo o nombre operativo se registra ese concepto en el sistema
- si una misma etiqueta del Excel cambia por empresa
- si hay conceptos que no deben registrarse aunque tengan valor

Por eso se necesita una capa intermedia de catalogo y equivalencias.

## Relacion con subflujos previos y posteriores

- `SF_ExpandirConceptosNomina` genera `registrosFuenteNovedadesNomina`
- `SF_AnalizarNovedadTextoNomina` segmenta texto libre pendiente
- `SF_ConciliarItemsNomina` deja una salida unificada con origen y estado de conciliacion
- `SF_ResolverCatalogoConceptosNomina` consume toda `gListaConciliacionNovedadesNomina` y resuelve cada fila contra catalogo usando la mejor base disponible
- subflujos posteriores deben encargarse de:
  - resolver empleado en el sistema
  - abrir formulario o pantalla correcta
  - seleccionar codigo real
  - registrar la novedad

## Principio rector

Toda fila de `gListaConciliacionNovedadesNomina` debe pasar por resolucion de catalogo.

No existen filas "fuera" de catalogo en esta fase. Lo que cambia entre filas es:

- la base disponible para resolver
- el metodo de resolucion aplicable
- el nivel de confianza resultante
- si requiere validacion humana posterior

Por tanto, este subflujo no debe preguntar si una fila aplica o no a catalogo. Debe intentar resolver todas las filas usando la mejor base disponible.

## Entrada esperada

- `gListaConciliacionNovedadesNomina` `List`
- `gTotalConciliacionNovedadesNomina` `Number`
- `gNombreEmpresaObjetivo` `Text`
- `gArchivoLog` `Text`

## Campos minimos de entrada

Campos minimos que debe leer de cada fila:

- `OrigenAnalisis`
- `EstadoConciliacion`
- `ArchivoOrigen`
- `HojaOrigen`
- `FilaExcel`
- `NombreEmpleado`
- `CedulaFuente`
- `ConceptoFuenteExcel`
- `ValorConceptoFuente`
- `NovedadTextoOriginal`
- `TextoFragmentoRelacionado`
- `TipoSugeridoIA`
- `MontoMencionadoIA`
- `DiasMencionadosIA`

## Dependencia de configuracion

Este subflujo debe depender de un catalogo externo configurable por empresa, no de valores quemados en el flujo.

Alternativas validas:

- archivo `json`
- archivo `csv`
- hoja Excel de equivalencias
- tabla de base de datos

Para una primera version PAD se adopta `json` como formato base.

## Ubicacion acordada del catalogo

Ruta base operativa:

- `C:\Dev\Aspraco\config`

Recomendacion de nombre inicial:

- `catalogo_conceptos_nomina.json`

Ruta esperada completa:

- `C:\Dev\Aspraco\config\catalogo_conceptos_nomina.json`

## Estructura conceptual minima del catalogo

Se recomienda separar:

- catalogo base del sistema por empresa
- asociaciones configuradas desde `ConceptoFuenteExcel`
- asociaciones configuradas desde `TipoSugeridoIA`, cuando aplique una ruta cerrada por tipo de evento

Cada concepto del catalogo base deberia permitir algo como:

- `Empresa`
- `CodigoSistema`
- `ConceptoSistema`
- `DescripcionOperativa`
- `TipoRegistro`
- `Activo`
- `Observaciones`

Cada asociacion deberia permitir algo como:

- `ConceptoFuenteNormalizado`
- `CodigosSistemaCandidato`
- `Activo`
- `Observaciones`

Cada asociacion por tipo de evento deberia permitir algo como:

- `TipoEventoNormalizado`
- `CodigosSistemaCandidato`
- `Activo`
- `Observaciones`

Opcionales utiles:

- `PalabrasClave`
- `RequiereValor`
- `PermiteValorCero`
- `GrupoConcepto`
- `Prioridad`

## Estructura JSON sugerida

Se recomienda una raiz con version y lista de empresas:

```json
{
  "version": "1.0",
  "catalogosNomina": [
    {
      "empresa": "MACOM RENTAL SAS",
      "activo": true,
      "conceptosSistema": [
        {
          "conceptoSistema": "BONO NO PRESTACIONAL OPERADOR",
          "codigoSistema": "BNP001",
          "descripcionOperativa": "Bono no salarial para operador",
          "tipoRegistro": "DEVENGADO",
          "activo": true,
          "observaciones": "Concepto usado para bono operador"
        }
      ],
      "asociacionesFuenteExcel": [
        {
          "conceptoFuenteNormalizado": "bonos operador no prestacional",
          "codigosSistemaCandidato": ["BNP001"],
          "activo": true,
          "observaciones": "Asociacion exacta conocida"
        }
      ],
      "asociacionesTipoEvento": [
        {
          "tipoEventoNormalizado": "incapacidad",
          "codigosSistemaCandidato": ["INC001", "INC002"],
          "activo": true,
          "observaciones": "Candidatos posibles para incapacidades"
        }
      ]
    }
  ]
}
```

## Reglas base de resolucion

1. tomar una fila conciliada
2. determinar la `BaseResolucionCatalogo` segun la informacion disponible
3. filtrar catalogo por empresa objetivo
4. aplicar la estrategia de resolucion segun esa base
5. traducir candidatos a conceptos reales de `conceptosSistema`
6. si hay una sola coincidencia activa:
   - marcar como `RESUELTO`
7. si hay varias coincidencias activas:
   - marcar como `AMBIGUO`
8. si no hay coincidencias:
   - marcar como `NO_ENCONTRADO`

## Bases de resolucion

Cada fila debe etiquetarse con una base principal de resolucion:

- `CONCEPTO_FUENTE`
- `TIPO_EVENTO`
- `TEXTO_NOVEDAD`
- `MIXTA`

Reglas iniciales sugeridas:

- si existe `ConceptoFuenteExcel`, usar `CONCEPTO_FUENTE`
- si no existe `ConceptoFuenteExcel` pero existe `TipoSugeridoIA`, usar `TIPO_EVENTO`
- si no existe concepto fuente y el tipo no es suficiente, usar `TEXTO_NOVEDAD`
- si hay señales estructurales y textuales en conflicto o complementarias, usar `MIXTA`

## Regla de prioridad

La resolucion debe seguir este orden:

1. coincidencia exacta deterministica en asociaciones configuradas desde `ConceptoFuenteExcel`
2. coincidencia deterministica por tipo de evento, si existe mapeo cerrado por empresa en `asociacionesTipoEvento`
3. sugerencia IA sobre lista cerrada de candidatos configurados, solo si el switch correspondiente esta activo
4. sugerencia IA sobre catalogo completo de la empresa, solo si el switch correspondiente esta activo y no existe resolucion deterministica

La IA no debe inventar conceptos del sistema ni autorizar registro por si sola.

## Switches propuestos

- `pHabilitarMapeoConceptosIA` `Boolean`
- `pResolverAmbiguosConIAPremapeados` `Boolean`
- `pResolverAmbiguosConIACatalogoCompleto` `Boolean`
- `pResolverNoEncontradosConIACatalogoCompleto` `Boolean`

Reglas:

- si `pHabilitarMapeoConceptosIA = False`, todo lo no resuelto por regla queda pendiente de validacion
- `AMBIGUO` con candidatos configurados puede pasar por IA solo si `pResolverAmbiguosConIAPremapeados = True`
- `AMBIGUO` sin lista cerrada puede pasar por IA con catalogo completo solo si `pResolverAmbiguosConIACatalogoCompleto = True`
- `NO_ENCONTRADO` puede pasar por IA con catalogo completo solo si `pResolverNoEncontradosConIACatalogoCompleto = True`

## Salidas sugeridas

- `gEstadoResolucionCatalogoNomina` `Text`
- `gMensajeError` `Text`
- `gListaResolucionCatalogoNovedadesNomina` `List`
- `gTotalResolucionCatalogoNovedadesNomina` `Number`
- `gTotalCatalogoResueltos` `Number`
- `gTotalCatalogoAmbiguos` `Number`
- `gTotalCatalogoNoEncontrados` `Number`
- `gHayResolucionCatalogoNovedadesNomina` `Boolean`

## Variables de trabajo recomendadas

- `gRutaCatalogoConceptosNomina` `Text`
- `gCatalogoConceptosNominaTexto` `Text`
- `gCatalogoConceptosNominaJson` `Custom object`
- `gListaCatalogosNomina` `List`
- `gEmpresaCatalogoEncontrada` `Boolean`
- `gEmpresaCatalogoActual` `Custom object`
- `gConceptosSistemaEmpresaActual` `List`
- `gAsociacionesFuenteExcelEmpresaActual` `List`
- `gAsociacionesTipoEventoEmpresaActual` `List`
- `gCodigosSistemaCandidatoActual` `List`
- `gCandidatosSistemaActual` `List`
- `gTotalFilasCatalogoSinBase` `Number`

## Unidad de salida recomendada

Cada registro de salida debe conservar toda la informacion de entrada y agregar al menos:

- `BaseResolucionCatalogo`
- `EmpresaObjetivo`
- `ConceptoFuenteNormalizado`
- `EstadoResolucionCatalogo`
- `ConceptoSistema`
- `CodigoSistema`
- `TipoRegistroSistema`
- `MetodoResolucionCatalogo`
- `ConfianzaResolucionCatalogo`
- `RequiereValidacionUsuario`
- `CandidatosCatalogo`
- `SugerenciaIACodigoSistema`
- `SugerenciaIAConceptoSistema`
- `MetodoSugerenciaIA`

Valores sugeridos:

- `EstadoResolucionCatalogo = RESUELTO | AMBIGUO | NO_ENCONTRADO`
- `MetodoResolucionCatalogo = REGLA_EXACTA | REGLA_CONFIGURADA | REGLA_TIPO_EVENTO | IA_CANDIDATOS | IA_CATALOGO`
- `ConfianzaResolucionCatalogo = ALTA | MEDIA | BAJA`

## Casos que debe manejar

### Caso 1. Un concepto candidato, una coincidencia exacta

Resultado esperado:

- `RESUELTO`
- `REGLA_EXACTA`
- `ALTA`
- `RequiereValidacionUsuario = False`

### Caso 2. Un concepto candidato, varias coincidencias posibles

Resultado esperado:

- `AMBIGUO`
- `IA_CANDIDATOS` si existe lista cerrada y el switch esta activo
- `IA_CATALOGO` si no existe lista cerrada y el switch correspondiente esta activo
- si no hay IA habilitada o no produce salida suficiente, la fila conserva su ultimo metodo efectivo y queda con `RequiereValidacionUsuario = True`
- `BAJA`
- `RequiereValidacionUsuario = True`

### Caso 3. Concepto candidato sin equivalencia cargada

Resultado esperado:

- `NO_ENCONTRADO`
- `IA_CATALOGO` o permanencia sin resolucion automatica segun switch activo
- `BAJA`
- `RequiereValidacionUsuario = True`

### Caso 4. Derivado calculable sin concepto fuente visible

Ejemplo:

- incapacidad
- vacaciones
- licencia

Resultado esperado:

- `BaseResolucionCatalogo = TIPO_EVENTO`
- `MetodoResolucionCatalogo = REGLA_TIPO_EVENTO` o IA segun switch
- `EstadoResolucionCatalogo = RESUELTO | AMBIGUO | NO_ENCONTRADO`
- `RequiereValidacionUsuario` segun confianza

### Caso 5. Texto sin respaldo estructural directo

Resultado esperado:

- `BaseResolucionCatalogo = TEXTO_NOVEDAD`
- `MetodoResolucionCatalogo = IA_CATALOGO` o permanencia sin resolucion automatica
- `EstadoResolucionCatalogo = AMBIGUO | NO_ENCONTRADO | RESUELTO`
- `RequiereValidacionUsuario = True` salvo una regla cerrada extremadamente clara

### Caso 6. Conflicto entre estructura y texto

Resultado esperado:

- `BaseResolucionCatalogo = MIXTA`
- `EstadoResolucionCatalogo = RESUELTO | AMBIGUO | NO_ENCONTRADO`
- `RequiereValidacionUsuario = True`
- `MotivoRevision` debe explicar la tension entre texto y estructura

## Lectura PAD del JSON

### 1. Cargar archivo

Acciones sugeridas:

- `Read text from file`
  - `File: gRutaCatalogoConceptosNomina`
  - `Content => gCatalogoConceptosNominaTexto`

- `Convert JSON to custom object`
  - `Json: gCatalogoConceptosNominaTexto`
  - `CustomObject => gCatalogoConceptosNominaJson`

### 2. Tomar lista de empresas

- `Set variable`
  - `gListaCatalogosNomina = gCatalogoConceptosNominaJson['catalogosNomina']`

### 3. Encontrar empresa objetivo

Recorrer `gListaCatalogosNomina`.

Por cada `CurrentEmpresaCatalogo` leer:

- `CurrentEmpresaCatalogo['empresaId']`
- `CurrentEmpresaCatalogo['empresaNombre']`
- `CurrentEmpresaCatalogo['activo']`

Cuando coincida con la empresa objetivo:

- `gEmpresaCatalogoActual = CurrentEmpresaCatalogo`
- `gConceptosSistemaEmpresaActual = CurrentEmpresaCatalogo['conceptosSistema']`
- `gAsociacionesFuenteExcelEmpresaActual = CurrentEmpresaCatalogo['asociacionesFuenteExcel']`
- `gAsociacionesTipoEventoEmpresaActual = CurrentEmpresaCatalogo['asociacionesTipoEvento']`

## Estrategia de resolucion por base

### Ruta 1. `CONCEPTO_FUENTE`

1. normalizar `ConceptoFuenteExcel`
2. recorrer `gAsociacionesFuenteExcelEmpresaActual`
3. comparar contra `conceptoFuenteNormalizado`
4. si hay match:
   - tomar `codigosSistemaCandidato`
5. traducir esos codigos contra `gConceptosSistemaEmpresaActual`

### Ruta 2. `TIPO_EVENTO`

1. normalizar `TipoSugeridoIA`
2. recorrer `gAsociacionesTipoEventoEmpresaActual`
3. comparar contra `tipoEventoNormalizado`
4. si hay match:
   - tomar `codigosSistemaCandidato`
5. traducir esos codigos contra `gConceptosSistemaEmpresaActual`

### Ruta 3. `TEXTO_NOVEDAD`

1. si no hubo resolucion por `ConceptoFuenteExcel` ni por `TipoSugeridoIA`
2. intentar IA sobre catalogo completo si el switch correspondiente esta activo
3. si no hay IA o no hay salida suficiente:
   - conservar `EstadoResolucionCatalogo = NO_ENCONTRADO`
   - `RequiereValidacionUsuario = True`

### Ruta 4. `MIXTA`

1. conservar `ConceptoFuenteExcel`, `TipoSugeridoIA` y `NovedadTextoOriginal` como contexto
2. intentar primero por `ConceptoFuenteExcel`
3. si eso no resuelve suficientemente, usar `TipoSugeridoIA`
4. si persiste ambiguedad:
   - IA sobre candidatos o catalogo segun switch

## Uso esperado de IA en resolucion de catalogo

La IA no entra de una sola forma. Deben existir dos rutas diferenciadas:

### Ruta 1. IA sobre candidatos premapeados

Aplica cuando:

- ya existe una lista cerrada de conceptos posibles para esa empresa
- la resolucion por regla no fue univoca

Metodo esperado:

- `IA_CANDIDATOS`

Salida esperada:

- sugerencia entre candidatos reales ya conocidos
- no debe inventar nuevos conceptos
- normalmente debe conservar `RequiereValidacionUsuario = True`

### Ruta 2. IA sobre catalogo completo

Aplica cuando:

- no existe premapeo suficiente
- o el caso no quedo resuelto con candidatos cerrados

Metodo esperado:

- `IA_CATALOGO`

Salida esperada:

- sugerencia sobre conceptos reales del catalogo de la empresa
- debe quedar marcada como sugerencia no autoritativa
- normalmente debe conservar `RequiereValidacionUsuario = True`

## Que no debe hacer este subflujo

- no releer el Excel
- no volver a expandir conceptos
- no buscar empleados en el sistema
- no registrar datos en la UI
- no decidir valores finales del negocio por IA sin candidatos reales o catalogo real de empresa

## Logging esperado

- inicio del subflujo
- empresa objetivo
- ruta del catalogo cargado
- total de filas conciliadas recibidas
- total por base de resolucion
- total resueltos
- total ambiguos
- total no encontrados
- resumen final

No se recomienda loggear todas las lineas resueltas salvo modo debug.

## Flujo propuesto en PAD

1. inicializar estado, contadores y listas de salida
2. validar que `gListaConciliacionNovedadesNomina` tenga elementos
3. construir `gRutaCatalogoConceptosNomina`
4. leer archivo JSON de catalogo
5. convertir JSON a custom object
6. ubicar empresa objetivo y capturar:
   - `conceptosSistema`
   - `asociacionesFuenteExcel`
   - `asociacionesTipoEvento`
7. validar que la empresa exista y este activa
8. recorrer cada fila recibida
9. determinar `BaseResolucionCatalogo`
10. ejecutar la ruta de resolucion correspondiente
11. traducir codigos candidatos a conceptos reales del sistema
12. si aplica, invocar sugerencia IA segun switches
13. construir registro enriquecido de salida
14. agregar a lista final
15. actualizar contadores por estado
16. registrar resumen final

## Manejo de errores

Seguir el patron ya usado:

- `BLOCK ... ON BLOCK ERROR`
- bandera `gEsErrorControlado`
- `THROW ERROR`
- captura final con `ERROR => gMensajeError`
- escritura en `gArchivoLog`

## Recomendacion de implementacion

Implementarlo en dos pasos:

1. version 1:
   - consumir toda `gListaConciliacionNovedadesNomina`
   - resolucion exacta deterministica por `ConceptoFuenteExcel`
   - resolucion deterministica cerrada por `TipoSugeridoIA` para eventos como incapacidad, vacaciones o licencia
   - sin IA
2. version 2:
   - sugerencia IA sobre ambiguos con lista cerrada
   - sugerencia IA sobre catalogo completo para ambiguos o no encontrados
   - persistencia de candidatos y sugerencias en evidencia Excel
