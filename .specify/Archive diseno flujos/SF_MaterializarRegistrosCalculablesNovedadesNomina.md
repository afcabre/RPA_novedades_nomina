# SF_MaterializarRegistrosCalculablesNovedadesNomina

Documento base del subflujo `SF_MaterializarRegistrosCalculablesNovedadesNomina` para el RPA de registro de novedades en aplicativo de nomina.

## Objetivo

Tomar la conciliacion de novedades nomina y materializar en una primera version solo los registros:

- `EstadoConciliacion = DERIVADO_CALCULABLE`
- `TipoSugeridoIA = INCAPACIDAD`

La formula acordada en V1 es:

```text
ValorCalculado = SueldoBaseTexto / 30 * DiasMencionadosIA
```

## Ubicacion en la secuencia

Este subflujo debe ejecutarse:

1. despues de `SF_ConciliarItemsNomina`
2. antes de escribir la hoja `conciliacionNovedadesNomina`
3. antes de `SF_ResolverCatalogoConceptosNomina`

## Razon de separacion

- `SF_ConciliarItemsNomina` solo detecta que existe un derivado calculable
- `SF_MaterializarRegistrosCalculablesNovedadesNomina` convierte ese derivado en un valor monetario materializado
- `SF_ResolverCatalogoConceptosNomina` opera luego sobre un registro ya valorizado

## Alcance V1

Solo se materializa:

- `INCAPACIDAD`

No se materializa aun:

- `VACACIONES`
- `LICENCIA`
- otros eventos calculables

## Entrada esperada

- `gListaConciliacionNovedadesNomina`
- `gTotalConciliacionNovedadesNomina`
- `gArchivoLog`

## Regla V1 de materializacion

Si un registro cumple:

- `EstadoConciliacion = DERIVADO_CALCULABLE`
- `TipoSugeridoIA = INCAPACIDAD`
- `SueldoBaseTexto` numerico
- `DiasMencionadosIA` numerico

Entonces:

- `ValorConceptoFuente = SueldoBaseTexto / 30 * DiasMencionadosIA`
- `EstadoConciliacion = CONCILIADO`
- `FuenteAutoritativa = CALCULO`
- `PermiteAutoRegistro = True`
- `RequiereCalculo = False`
- `MetodoConciliacion = REGLA_CALCULO`
- `ConfianzaConciliacion = ALTA`

## Salidas sugeridas

- `gEstadoMaterializacionRegistrosCalculablesNovedadesNomina`
- `gMensajeError`
- `gListaConciliacionNovedadesNomina`
- `gTotalConciliacionNovedadesNomina`
- `gTotalRegistrosCalculablesEvaluadosNovedadesNomina`
- `gTotalRegistrosCalculablesMaterializadosNovedadesNomina`
- `gTotalRegistrosCalculablesSinMaterializarNovedadesNomina`
- `gHayRegistrosCalculablesMaterializadosNovedadesNomina`

## Contrato operativo

El subflujo no crea un contrato nuevo.

Debe devolver la misma `gListaConciliacionNovedadesNomina`, pero con los registros aplicables ya transformados, para que los subflujos posteriores no requieran ramas especiales.

## Criterio conservador

Si no se puede convertir sueldo o dias:

- no se debe inventar valor
- el registro debe conservarse sin materializar
- se debe loggear como `REQUIERE_VALIDACION`

## Persistencia esperada

Como este subflujo corre antes de `SF_EscribirConciliacionNovedadesNominaExcel`, la hoja de conciliacion ya debe reflejar el valor calculado materializado.
