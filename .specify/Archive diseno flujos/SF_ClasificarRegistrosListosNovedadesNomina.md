# SF_ClasificarRegistrosListosNovedadesNomina

Documento base del subflujo `SF_ClasificarRegistrosListosNovedadesNomina` para el RPA de registro de novedades en aplicativo de nomina.

## Objetivo

Tomar la resolucion de catalogo ya enriquecida con apoyo IA cuando exista, y decidir si cada registro queda:

- `LISTO_AUTORREGISTRO`
- `REQUIERE_VALIDACION_USUARIO`

Este subflujo no registra aun en el sistema ni escribe aun en Excel. Su responsabilidad es dejar una particion operativa limpia para la siguiente fase.

## Principio rector

En esta fase no se introduce heuristica adicional.

La clasificacion debe basarse solo en reglas duras ya disponibles en memoria:

- estado de conciliacion
- permiso de autorregistro
- necesidad de calculo
- resultado final de catalogo
- validacion requerida en catalogo base o IA

## Entradas esperadas

- `gListaResolucionCatalogoNovedadesNomina`
- `gListaResolucionCatalogoNovedadesNominaIA`
- `gConceptosSistemaEmpresaActual`
- `gArchivoLog`

## Salidas sugeridas

- `gEstadoClasificacionRegistrosListosNovedadesNomina`
- `gMensajeError`
- `gListaClasificacionRegistrosListosNovedadesNomina`
- `gTotalClasificacionRegistrosListosNovedadesNomina`
- `gHayClasificacionRegistrosListosNovedadesNomina`
- `gListaRegistrosListosAutorregistroNovedadesNomina`
- `gTotalRegistrosListosAutorregistroNovedadesNomina`
- `gHayRegistrosListosAutorregistroNovedadesNomina`
- `gListaRegistrosRequierenValidacionNovedadesNomina`
- `gTotalRegistrosRequierenValidacionNovedadesNomina`
- `gHayRegistrosRequierenValidacionNovedadesNomina`

## Regla de consolidacion previa

La clasificacion debe consolidar primero un resultado final de catalogo por registro:

1. tomar la resolucion base desde `gListaResolucionCatalogoNovedadesNomina`
2. buscar match por:
   - `IdEjecucion`
   - `LlaveRegistroAnalisis`
   - `OrigenAnalisis`
3. si existe sugerencia IA con codigo/concepto:
   - usar esa sugerencia como resultado final
4. si no existe:
   - conservar la resolucion base

## Reglas de clasificacion

### `LISTO_AUTORREGISTRO`

Solo si se cumplen todas estas condiciones:

- `EstadoConciliacion = CONCILIADO`
- `PermiteAutoRegistro = True`
- `RequiereCalculo = False`
- `EstadoResolucionCatalogoFinal = RESUELTO`
- existe `CodigoSistemaFinal`
- existe `ConceptoSistemaFinal`
- `RequiereValidacionUsuarioFinal = False`

### `REQUIERE_VALIDACION_USUARIO`

En cualquier otro caso.

Ejemplos tipicos:

- `CONFLICTO`
- `TEXTO_SIN_RESPALDO`
- `DERIVADO_CALCULABLE` mientras no exista una fase posterior que materialice el valor
- sugerencia IA presente
- catalogo no resuelto
- validacion de usuario aun obligatoria

## Contrato V1 de salida

Formato por item en `gListaClasificacionRegistrosListosNovedadesNomina`:

```text
IdEjecucion|LlaveRegistroAnalisis|OrigenAnalisis|ArchivoOrigen|HojaOrigen|FilaExcel|Ciudad|NombreEmpleado|CedulaFuente|Area|SueldoBaseTexto|AuxExtralegalBaseTexto|ConceptoFuenteExcel|ColumnaOrigenExcel|ValorConceptoFuente|NovedadTextoOriginal|TextoFragmentoRelacionado|TipoSugeridoIA|MontoMencionadoIA|DiasMencionadosIA|EstadoConciliacion|FuenteAutoritativa|PermiteAutoRegistro|RequiereCalculo|MetodoConciliacion|ConfianzaConciliacion|MotivoRevision|BaseResolucionCatalogo|EstadoResolucionCatalogoFinal|MetodoResolucionCatalogoFinal|CodigoSistemaFinal|ConceptoSistemaFinal|TipoRegistroSistemaFinal|ConfianzaResolucionCatalogoFinal|RequiereValidacionUsuarioFinal|ObservacionResolucionCatalogoFinal|EstadoClasificacionRegistro|MotivoClasificacionRegistro|__END__
```

## Subflujo posterior esperado

La siguiente fase deberia consumir preferiblemente:

- `gListaRegistrosListosAutorregistroNovedadesNomina`

Y dejar para revision humana:

- `gListaRegistrosRequierenValidacionNovedadesNomina`

## Persistencia posterior recomendada

Para mantener el patron ya adoptado, la escritura a Excel debe ir en un subflujo hermano posterior, por ejemplo:

- `SF_EnriquecerConciliacionNovedadesNominaConClasificacionRegistro`

Ese enriquecimiento podria escribir sobre la misma hoja `conciliacionNovedadesNomina` las columnas:

- `EstadoClasificacionRegistro`
- `MotivoClasificacionRegistro`

## Criterio de seguridad

Este subflujo debe ser conservador.

Si existe cualquier duda, debe clasificar como:

- `REQUIERE_VALIDACION_USUARIO`
