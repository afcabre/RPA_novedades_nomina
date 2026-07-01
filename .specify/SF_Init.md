# SF_Init

Documento base del subflujo `SF_Init` para el RPA de registro de novedades en aplicativo de nomina. Este documento define la propuesta inicial de estructura y comportamiento del subflujo. Puede ajustarse durante la implementacion, pero sirve como linea base para mantener claridad sobre el objetivo y el diseno esperado.

## Objetivo

Preparar el entorno de ejecucion, validar prerequisitos, construir identificadores y rutas de trabajo, preparar logging y confirmar que el archivo fuente de Excel puede ser leido antes de pasar a subflujos posteriores.

## Firma sugerida del subflujo

### Entradas

- `pRutaExcel` `Text`
- `pRutaAppNomina` `Text`
- `pDirectorioLogs` `Text`
- `pUsaCredencialesSeguras` `Boolean`
- `pUsuarioNomina` `Text`
- `pNombreProceso` `Text`

### Salidas

- `gEstadoInicializacion` `Text`
- `gMensajeError` `Text`
- `gFechaHoraInicio` `Datetime`
- `gIdEjecucion` `Text`
- `gArchivoLog` `Text`
- `gHojasExcel` `List`
- `gTotalHojas` `Number`
- `gSesionAppAbierta` `Boolean`
- `gAutenticado` `Boolean`
- `gUsuarioNomina` `Text`

## Flujo literal en PAD

1. `Get current date and time`
- Output:
  - `gFechaHoraInicio`

2. `Set variable`
- Variable: `gEstadoInicializacion`
- Value: `"INICIANDO"`

3. `Set variable`
- Variable: `gMensajeError`
- Value: `""`

4. `Set variable`
- Variable: `gSesionAppAbierta`
- Value: `False`

5. `Set variable`
- Variable: `gAutenticado`
- Value: `False`

6. `Set variable`
- Variable: `gTotalHojas`
- Value: `0`

7. `Convert datetime to text`
- Datetime to convert: `gFechaHoraInicio`
- Custom format: `"yyyyMMdd_HHmmss"`
- Output:
  - `gIdEjecucionTmp`

8. `Set variable`
- Variable: `gIdEjecucion`
- Value: `%"NOV_" + gIdEjecucionTmp%`

9. `If folder exists`
- Folder path: `pDirectorioLogs`

10. `Create folder`
- Folder path: `pDirectorioLogs`
- Solo en rama `If not exists`

11. `Set variable`
- Variable: `gArchivoLog`
- Value: `%"%pDirectorioLogs%\\Log_%gIdEjecucion%.txt"%`

12. `Create file`
- File path: `gArchivoLog`
- If file exists: `Do nothing`

13. `Write text to file`
- File path: `gArchivoLog`
- Text to write:

```text
Inicio ejecucion | Proceso: %pNombreProceso% | Id: %gIdEjecucion%
```

- Append new line: `True`

14. `Write text to file`
- File path: `gArchivoLog`
- Text to write:

```text
Fecha inicio: %gFechaHoraInicio%
```

- Append new line: `True`

15. `If file exists`
- File path: `pRutaExcel`

16. Rama `No`
- `Set variable`
  - `gMensajeError = "No se encontro el archivo Excel en la ruta indicada"`
- `Set variable`
  - `gEstadoInicializacion = "ERROR"`
- `Write text to file`
  - File path: `gArchivoLog`
  - Text:

```text
ERROR | %gMensajeError% | Ruta Excel: %pRutaExcel%
```

  - Append new line: `True`
- `Exit subflow`
  - Outputs ya cargadas

17. Rama `Si`
- `Write text to file`
  - File path: `gArchivoLog`
  - Text:

```text
OK | Archivo Excel encontrado | %pRutaExcel%
```

  - Append new line: `True`

18. `If file exists`
- File path: `pRutaAppNomina`

19. Rama `No`
- `Set variable`
  - `gMensajeError = "No se encontro el ejecutable del aplicativo de nomina"`
- `Set variable`
  - `gEstadoInicializacion = "ERROR"`
- `Write text to file`
  - File path: `gArchivoLog`
  - Text:

```text
ERROR | %gMensajeError% | Ruta App: %pRutaAppNomina%
```

  - Append new line: `True`
- `Exit subflow`

20. Rama `Si`
- `Write text to file`
  - File path: `gArchivoLog`
  - Text:

```text
OK | Aplicativo encontrado | %pRutaAppNomina%
```

  - Append new line: `True`

21. `If`
- Condition: `pUsaCredencialesSeguras = True`

22. Rama `Si`
- `Get username and password from Windows Credential Manager`
- Credential name: el alias que se defina, por ejemplo `NominaApp`
- Outputs:
  - `gUsuarioNomina`
  - `gPasswordNomina`
- Si esta accion falla, manejar en bloque de error

23. Rama `No`
- `Set variable`
  - `gUsuarioNomina = pUsuarioNomina`

24. `Write text to file`
- File path: `gArchivoLog`
- Text:

```text
Configuracion credenciales | Seguras: %pUsaCredencialesSeguras% | Usuario: %gUsuarioNomina%
```

- Append new line: `True`

25. `Launch Excel`
- Launch Excel: `Open the following document`
- Document path: `pRutaExcel`
- Visible: `False`
- Read only: `True`
- Instance output:
  - `ExcelInstance`

26. `Get all worksheets from Excel workbook`
- Excel instance: `ExcelInstance`
- Output:
  - `gHojasExcel`

27. `Get details of a list`
- List: `gHojasExcel`
- Detail: `Count`
- Output:
  - `gTotalHojas`

28. `Write text to file`
- File path: `gArchivoLog`
- Text:

```text
Total hojas detectadas: %gTotalHojas%
```

- Append new line: `True`

29. `If`
- Condition: `gTotalHojas = 0`

30. Rama `Si`
- `Set variable`
  - `gMensajeError = "El Excel no contiene hojas para procesar"`
- `Set variable`
  - `gEstadoInicializacion = "ERROR"`
- `Write text to file`
  - File path: `gArchivoLog`
  - Text:

```text
ERROR | %gMensajeError%
```

  - Append new line: `True`
- `Close Excel`
  - Excel instance: `ExcelInstance`
  - Save document: `False`
- `Exit subflow`

31. Rama `No`
- `Close Excel`
  - Excel instance: `ExcelInstance`
  - Save document: `False`

32. `Set variable`
- Variable: `gEstadoInicializacion`
- Value: `"OK"`

33. `Write text to file`
- File path: `gArchivoLog`
- Text:

```text
Inicializacion completada correctamente
```

- Append new line: `True`

## Bloque de error recomendado

Envuelve desde la accion 25 en `Begin error handling`.

### En `On block error`

- `Set variable`
  - `gEstadoInicializacion = "ERROR"`
- `Set variable`
  - `gMensajeError = %Error.Message%`
- `Write text to file`
  - File path: `gArchivoLog`
  - Text:

```text
ERROR CONTROLADO | %gMensajeError%
```

  - Append new line: `True`
- `Close Excel`
  - Solo si `ExcelInstance` existe
- `End`

## Variables adicionales recomendadas

- `gPasswordNomina` `Sensitive text`
- `ExcelInstance` `Excel instance`
- `gIdEjecucionTmp` `Text`

## Reglas practicas

- No dejes la contrasena en logs.
- Abre el Excel en `Read only` en `SF_Init`.
- No proceses datos aqui; solo valida y prepara.

## Nota de evolucion

Esta propuesta es una linea base. A medida que se implemente el flujo, puede refinarse para reflejar la realidad del aplicativo, el origen de los datos, la forma final del logging y los mecanismos reales de autenticacion y manejo de errores.
