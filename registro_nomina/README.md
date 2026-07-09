# Registro Nomina

Este frente contiene el pipeline independiente para registro de novedades en el aplicativo web de nomina.

## Alcance

- Arranca desde el archivo Excel de analisis ya generado por el pipeline principal.
- Consume como fuente operativa la hoja `registroNovedadesNomina`.
- Permite checkpoints e interaccion humana sin contaminar el flujo analitico principal.
- Agrupa UI por checkpoints operativos, no por cada click o control individual.

## Estructura

- `Main_RegistroNomina.txt`
- `src/`

## Subflujos base

- `SF_InitRegistroNomina`
- `SF_PrepararContextoRegistroNomina`
- `SF_TomarSiguienteRegistroPendienteNomina`
- `SF_RegistrarRegistroNomina`
- `SF_ConfirmarResultadoRegistroNomina`
- `SF_ActualizarResultadoRegistroExcel`
