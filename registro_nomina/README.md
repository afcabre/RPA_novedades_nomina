# Registro Nomina

Este frente contiene el pipeline independiente para registro de novedades en el aplicativo web de nomina.

## Alcance

- Arranca desde el archivo Excel de analisis ya generado por el pipeline principal.
- Consume como fuente operativa la hoja `registroNovedadesNomina`.
- Trata ese Excel como archivo maestro de corrida: salida del pipeline analitico y entrada viva del pipeline de registro UI.
- Permite checkpoints e interaccion humana sin contaminar el flujo analitico principal.
- Agrupa UI por checkpoints operativos, no por cada click o control individual.
- Asume que el contexto operativo ya entra con empresa seleccionada; este frente no depende de `idEmpresa` ni `nombreEmpresa`.

## Ruta operativa

- Carpeta maestra compartida:
  - `salidas\\analisis_nomina`
- Lectura del frente `registro_nomina`:
  - consume y actualiza el mismo archivo Excel de analisis
- Carpeta auxiliar futura del frente UI, si se necesita:
  - `salidas\\registro_nomina`

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
