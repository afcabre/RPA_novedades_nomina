# SF_ResolverCatalogoConceptosNomina

Documento base del subflujo `SF_ResolverCatalogoConceptosNomina` para el RPA de registro de novedades en aplicativo de nomina.

## Objetivo

Tomar la lista `gListaConciliacionNovedadesNomina` producida por `SF_ConciliarItemsNomina` y resolver, para cada fila conciliada de tipo `REGISTRO_FUENTE`, su equivalencia operativa dentro del sistema de nomina segun la empresa objetivo.

Este subflujo no debe registrar aun en el sistema. Su responsabilidad es dejar cada concepto candidato enriquecido con resultado de catalogo:

- resuelto
- ambiguo
- no encontrado
- no aplica a catalogo

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
- `SF_ResolverCatalogoConceptosNomina` solo toma la parte material conciliada y la cruza contra un catalogo por empresa
- subflujos posteriores deben encargarse de:
  - resolver empleado en el sistema
  - abrir formulario o pantalla correcta
  - seleccionar codigo real
  - registrar la novedad

## Regla de particion obligatoria

Este subflujo no debe procesar toda la conciliacion indiscriminadamente.

Solo entran filas que cumplan:

- `OrigenAnalisis = REGISTRO_FUENTE`
- `EstadoConciliacion = CONCILIADO`
- `ConceptoFuenteExcel` con valor

No entran aqui:

- `NOVEDAD_TEXTO`
- `DERIVADO_CALCULABLE`
- `TEXTO_SIN_RESPALDO`
- `CONFLICTO`

Esos casos deben quedar en una ruta separada, por ejemplo:

- `SF_PrepararPendientesTextoNomina`
- o un resolvedor especializado posterior, cuando exista base funcional para ello

## Entrada esperada

- `gListaConciliacionNovedadesNomina` `List`
- `gTotalConciliacionNovedadesNomina` `Number`
- `gNombreEmpresaObjetivo` `Text`
- `gArchivoLog` `Text`

## Filtro de entrada esperado

La primera responsabilidad del subflujo es depurar `gListaConciliacionNovedadesNomina` y construir una lista operativa interna solo con filas aptas para resolucion de catalogo.

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
- asociaciones deterministicas desde `ConceptoFuenteExcel`

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
      ]
    }
  ]
}
```

## Reglas base de resolucion

1. tomar una fila conciliada apta
2. validar que `OrigenAnalisis = REGISTRO_FUENTE`
3. validar que `EstadoConciliacion = CONCILIADO`
4. normalizar `ConceptoFuenteExcel`
5. filtrar catalogo por empresa objetivo
6. buscar coincidencia exacta en `asociacionesFuenteExcel`
7. traducir candidatos a conceptos reales de `conceptosSistema`
8. si hay una sola coincidencia activa:
   - marcar como `RESUELTO`
9. si hay varias coincidencias activas:
   - marcar como `AMBIGUO`
10. si no hay coincidencias:
   - marcar como `NO_ENCONTRADO`

## Regla de prioridad

La resolucion debe seguir este orden:

1. coincidencia exacta deterministica en asociaciones configuradas
2. sugerencia IA sobre lista cerrada de candidatos premapeados, solo si el switch correspondiente esta activo
3. sugerencia IA sobre catalogo completo de la empresa, solo si el switch correspondiente esta activo y no existe resolucion deterministica

La IA no debe inventar conceptos del sistema ni autorizar registro por si sola.

## Switches propuestos

- `pHabilitarMapeoConceptosIA` `Boolean`
- `pResolverAmbiguosConIAPremapeados` `Boolean`
- `pResolverAmbiguosConIACatalogoCompleto` `Boolean`
- `pResolverNoEncontradosConIACatalogoCompleto` `Boolean`

Reglas:

- si `pHabilitarMapeoConceptosIA = False`, todo lo no resuelto por regla queda para revision
- `AMBIGUO` con candidatos premapeados puede pasar por IA solo si `pResolverAmbiguosConIAPremapeados = True`
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
- `gTotalCatalogoNoAplicaCatalogo` `Number`
- `gHayResolucionCatalogoNovedadesNomina` `Boolean`

## Unidad de salida recomendada

Cada registro de salida debe conservar toda la informacion de entrada y agregar al menos:

- `EmpresaObjetivo`
- `ConceptoFuenteNormalizado`
- `EstadoResolucionCatalogo`
- `ConceptoSistema`
- `CodigoSistema`
- `TipoRegistroSistema`
- `MetodoResolucionCatalogo`
- `ConfianzaResolucionCatalogo`
- `RequiereRevisionCatalogo`
- `CandidatosCatalogo`
- `SugerenciaIACodigoSistema`
- `SugerenciaIAConceptoSistema`
- `MetodoSugerenciaIA`

Valores sugeridos:

- `EstadoResolucionCatalogo = RESUELTO | AMBIGUO | NO_ENCONTRADO | NO_APLICA_CATALOGO`
- `MetodoResolucionCatalogo = EXACTO | IA_PREMAPEADOS | IA_CATALOGO_COMPLETO | MANUAL`
- `ConfianzaResolucionCatalogo = ALTA | MEDIA | BAJA`

## Casos que debe manejar

### Caso 1. Un concepto candidato, una coincidencia exacta

Resultado esperado:

- `RESUELTO`
- `EXACTO`
- `ALTA`
- `RequiereRevisionCatalogo = False`

### Caso 2. Un concepto candidato, varias coincidencias posibles

Resultado esperado:

- `AMBIGUO`
- `MANUAL` o sugerencia IA segun switch activo
- `BAJA`
- `RequiereRevisionCatalogo = True`

### Caso 3. Concepto candidato sin equivalencia cargada

Resultado esperado:

- `NO_ENCONTRADO`
- `MANUAL` o sugerencia IA con catalogo completo segun switch activo
- `BAJA`
- `RequiereRevisionCatalogo = True`

### Caso 4. Fila de conciliacion no apta para catalogo

Resultado esperado:

- `NO_APLICA_CATALOGO`
- `MetodoResolucionCatalogo = MANUAL`
- `ConfianzaResolucionCatalogo = BAJA`
- `RequiereRevisionCatalogo = False`
- `MotivoDescartado = No aplica a resolucion de catalogo en esta fase`

## Que no debe hacer este subflujo

- no releer el Excel
- no volver a expandir conceptos
- no buscar empleados en el sistema
- no registrar datos en la UI
- no resolver desde texto libre si no existe `ConceptoFuenteExcel`
- no decidir valores finales del negocio por IA sin candidatos reales o catalogo real de empresa

## Logging esperado

- inicio del subflujo
- empresa objetivo
- total de filas conciliadas recibidas
- total aptas para catalogo
- total no aplica catalogo
- total resueltos
- total ambiguos
- total no encontrados
- resumen final

No se recomienda loggear todas las lineas resueltas salvo modo debug.

## Flujo propuesto en PAD

1. inicializar estado, contadores y listas de salida
2. validar que `gListaConciliacionNovedadesNomina` tenga elementos
3. recorrer conciliacion y depurar solo filas aptas para catalogo
4. cargar catalogo de conceptos de la empresa objetivo
5. recorrer cada fila apta
6. normalizar `ConceptoFuenteExcel`
7. buscar coincidencias activas en asociaciones configuradas
8. si aplica, invocar sugerencia IA segun switches
9. construir registro enriquecido de salida
10. agregar a lista final
11. actualizar contadores por estado
12. registrar resumen final

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
   - consumir solo `REGISTRO_FUENTE` conciliado
   - resolucion exacta deterministica por empresa
   - sin IA
2. version 2:
   - sugerencia IA sobre ambiguos con lista cerrada
   - sugerencia IA sobre catalogo completo para ambiguos o no encontrados
   - persistencia de candidatos y sugerencias en evidencia Excel
