# SF_ResolverCatalogoConceptosNomina

Documento base del subflujo `SF_ResolverCatalogoConceptosNomina` para el RPA de registro de novedades en aplicativo de nomina.

## Objetivo

Tomar la lista `gListaConceptosNominaCandidatos` producida por `SF_ExpandirConceptosNomina` y resolver, para cada concepto fuente detectado en Excel, su equivalencia operativa dentro del sistema de nomina segun la empresa objetivo.

Este subflujo no debe registrar aun en el sistema. Su responsabilidad es dejar cada concepto candidato enriquecido con resultado de catalogo:

- resuelto
- ambiguo
- no encontrado
- descartado

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

- `SF_ExpandirConceptosNomina` genera conceptos candidatos con contexto
- `SF_ResolverCatalogoConceptosNomina` cruza esos conceptos contra un catalogo por empresa
- luego subflujos posteriores deben encargarse de:
  - resolver empleado en el sistema
  - abrir formulario o pantalla correcta
  - seleccionar codigo real
  - registrar la novedad

## Entrada esperada

- `gListaConceptosNominaCandidatos` `List`
- `gTotalConceptosNominaCandidatos` `Number`
- `gNombreEmpresaObjetivo` `Text`
- `gArchivoLog` `Text`

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

Cada registro del catalogo deberia permitir algo como:

- `Empresa`
- `ConceptoFuenteNormalizado`
- `ConceptoSistema`
- `CodigoSistema`
- `TipoRegistro`
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
      "conceptos": [
        {
          "conceptoFuenteNormalizado": "bonos operador no prestacional",
          "conceptoSistema": "BONO NO PRESTACIONAL OPERADOR",
          "codigoSistema": "BNP001",
          "tipoRegistro": "DEVENGADO",
          "activo": true,
          "observaciones": "Concepto usado para bono operador"
        }
      ]
    }
  ]
}
```

## Reglas base de resolucion

1. tomar un concepto candidato
2. normalizar el texto de `ConceptoFuente`
3. filtrar catalogo por empresa objetivo
4. buscar coincidencia exacta por concepto fuente normalizado
5. si hay una sola coincidencia activa:
   - marcar como `RESUELTO`
6. si hay varias coincidencias activas:
   - marcar como `AMBIGUO`
7. si no hay coincidencias:
   - marcar como `NO_ENCONTRADO`

## Regla de prioridad

La resolucion debe seguir este orden:

1. coincidencia exacta deterministica en catalogo
2. coincidencia por alias o palabra clave, si se define explicitamente
3. fallback controlado con IA solo si ya existe una lista cerrada de candidatos

La IA no debe inventar conceptos del sistema.

## Salidas sugeridas

- `gEstadoResolucionCatalogoNomina` `Text`
- `gMensajeError` `Text`
- `gListaConceptosNominaResueltos` `List`
- `gTotalConceptosNominaResueltos` `Number`
- `gTotalConceptosNominaAmbiguos` `Number`
- `gTotalConceptosNominaNoEncontrados` `Number`
- `gHayConceptosNominaResueltos` `Boolean`

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

Valores sugeridos:

- `EstadoResolucionCatalogo = RESUELTO | AMBIGUO | NO_ENCONTRADO | DESCARTADO`
- `MetodoResolucionCatalogo = EXACTO | ALIAS | IA | MANUAL`
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
- `MANUAL` o `IA` en fase futura
- `BAJA`
- `RequiereRevisionCatalogo = True`

### Caso 3. Concepto candidato sin equivalencia cargada

Resultado esperado:

- `NO_ENCONTRADO`
- `MANUAL`
- `BAJA`
- `RequiereRevisionCatalogo = True`

## Que no debe hacer este subflujo

- no releer el Excel
- no volver a expandir conceptos
- no buscar empleados en el sistema
- no registrar datos en la UI
- no decidir valores finales del negocio por IA sin lista cerrada de candidatos

## Logging esperado

- inicio del subflujo
- empresa objetivo
- total de conceptos candidatos recibidos
- total resueltos
- total ambiguos
- total no encontrados
- resumen final

No se recomienda loggear todas las lineas resueltas salvo modo debug.

## Flujo propuesto en PAD

1. inicializar estado, contadores y listas de salida
2. validar que `gListaConceptosNominaCandidatos` tenga elementos
3. cargar catalogo de conceptos de la empresa objetivo
4. recorrer cada candidato
5. normalizar `ConceptoFuente`
6. buscar coincidencias activas en catalogo
7. construir registro enriquecido de salida
8. agregar a lista de resueltos
9. actualizar contadores por estado
10. registrar resumen final

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
   - resolucion exacta deterministica por empresa
   - sin IA
   - sin alias complejos
2. version 2:
   - alias controlados
   - candidatos multiples
   - fallback IA solo sobre opciones cerradas
