# SF_Init

Documento base del subflujo `SF_Init` para el RPA de registro de novedades en aplicativo de nomina. Este documento define la propuesta inicial de estructura y comportamiento del subflujo. Puede ajustarse durante la implementacion, pero sirve como linea base para mantener claridad sobre el objetivo y el diseno esperado. La version actual asume que el aplicativo de nomina es una pagina web y no un ejecutable de escritorio, y que el proceso puede trabajar con multiples archivos Excel de entrada.

## Objetivo

Preparar el entorno de ejecucion, validar prerequisitos, cargar la configuracion base, construir identificadores y rutas de trabajo, preparar logging y dejar abierto el navegador sobre la URL del aplicativo para subflujos posteriores. La deteccion y procesamiento de archivos Excel se delega a subflujos posteriores.

## Rutas operativas minimas

La configuracion de rutas debe distinguir al menos:

- `directorioEntrada`
- `directorioSalidas`
- `directorioLogs`
- `directorioPlantillas`

`SF_Init` debe validar y dejar disponibles estas rutas para evitar que los subflujos de descubrimiento tomen como entrada archivos generados por la propia automatizacion.

## Ajuste vigente

Aunque el resto de variables operativas aun no se ha migrado por completo a `config.json`, `SF_Init` debe leer desde ya dos bloques separados de IA:

- `analisisNovedadTextoIA`
- `resolucionCatalogoIA`

Ambos pueden reutilizar la misma `credentialAliasIA` si se trabaja con la misma API key, pero deben mantener modelos y timeouts independientes por fase.

## Firma sugerida del subflujo

### Entradas

- `pUrlAppNomina` `Text`
- `pDirectorioEntrada` `Text`
- `pDirectorioSalidas` `Text`
- `pDirectorioLogs` `Text`
- `pDirectorioPlantillas` `Text`
- `pRutaConfigJson` `Text`
- `pUsaCredencialesSeguras` `Boolean`
- `pUsuarioNomina` `Text`
- `pNombreProceso` `Text`

### Salidas

- `gEstadoInicializacion` `Text`
- `gMensajeError` `Text`
- `gFechaHoraInicio` `Datetime`
- `gIdEjecucion` `Text`
- `gArchivoLog` `Text`
- `gSesionAppAbierta` `Boolean`
- `gAutenticado` `Boolean`
- `gUsuarioNomina` `Text`
- `gArchivosPendientes` `List`
- `gTotalArchivosPendientes` `Number`

### Variables IA cargadas en Init

#### Analisis de novedad texto

- `pProveedorIAAnalisisTexto` `Text`
- `pModeloIAAnalisisTexto` `Text`
- `pEndpointIAAnalisisTexto` `Text`
- `pCredentialAliasIAAnalisisTexto` `Text`
- `pTimeoutIAsegundosAnalisisTexto` `Number`
- `pDebugIAAnalisisTexto` `Boolean`

#### Resolucion de catalogo

- `pProveedorIAResolucionCatalogo` `Text`
- `pModeloIAResolucionCatalogo` `Text`
- `pEndpointIAResolucionCatalogo` `Text`
- `pCredentialAliasIAResolucionCatalogo` `Text`
- `pTimeoutIAsegundosResolucionCatalogo` `Number`
- `pDebugIAResolucionCatalogo` `Boolean`

#### Compatibilidad temporal

Mientras `SF_AnalizarNovedadTextoNomina` siga consumiendo nombres antiguos, `SF_Init` puede seguir poblando estas variables como alias de compatibilidad desde el bloque de analisis de texto:

- `pProveedorIA`
- `pModeloIA`
- `pEndpointIA`
- `pCredentialAliasIA`
- `pTimeoutIAsegundos`
- `pDebugIA`

## Bloque sugerido en config.json

```json
{
  "analisisNovedadTextoIA": {
    "proveedorIA": "mistral",
    "modeloIA": "mistral-medium-latest",
    "endpointIA": "https://api.mistral.ai/v1/chat/completions",
    "credentialAliasIA": "Aspraco_Mistral_API",
    "timeoutIAsegundos": 60,
    "debugIA": false
  },
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
- Variable: `gTotalArchivosPendientes`
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

15. `If folder exists`
- Folder path: `pDirectorioEntrada`

16. Rama `No`
- `Set variable`
  - `gMensajeError = "No se encontro el directorio de entrada configurado"`
- `Set variable`
  - `gEstadoInicializacion = "ERROR"`
- `Write text to file`
  - File path: `gArchivoLog`
  - Text:

```text
ERROR | %gMensajeError% | Directorio entrada: %pDirectorioEntrada%
```

  - Append new line: `True`
- `Exit subflow`
  - Outputs ya cargadas

17. Rama `Si`
- `Write text to file`
  - File path: `gArchivoLog`
  - Text:

```text
OK | Directorio de entrada encontrado | %pDirectorioEntrada%
```

  - Append new line: `True`

18. `If`
- Condition: `pUrlAppNomina` is empty

19. Rama `No`
- `Set variable`
  - `gMensajeError = "No se definio la URL del aplicativo de nomina"`
- `Set variable`
  - `gEstadoInicializacion = "ERROR"`
- `Write text to file`
  - File path: `gArchivoLog`
  - Text:

```text
ERROR | %gMensajeError%
```

  - Append new line: `True`
- `Exit subflow`

20. Rama `Si`
- `Write text to file`
  - File path: `gArchivoLog`
  - Text:

```text
OK | URL aplicativo configurada | %pUrlAppNomina%
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

25. `Get files in folder`
- Folder: `pDirectorioEntrada`
- File filter: `*.xlsx`
- Include subfolders: `False`
- Fail upon empty list: `False`
- Output:
  - `gArchivosPendientes`

26. `Get details of a list`
- List: `gArchivosPendientes`
- Detail: `Count`
- Output:
  - `gTotalArchivosPendientes`

27. `Write text to file`
- File path: `gArchivoLog`
- Text:

```text
Total archivos Excel detectados: %gTotalArchivosPendientes%
```

- Append new line: `True`

28. `If`
- Condition: `gTotalArchivosPendientes = 0`

29. Rama `Si`
- `Set variable`
  - `gMensajeError = "No se encontraron archivos Excel para procesar en el directorio de entrada"`
- `Set variable`
  - `gEstadoInicializacion = "ERROR"`
- `Write text to file`
  - File path: `gArchivoLog`
  - Text:

```text
ERROR | %gMensajeError%
```

  - Append new line: `True`
- `Exit subflow`

30. Rama `No`
- `Write text to file`
- File path: `gArchivoLog`
- Text:

```text
Archivos listos para procesamiento posterior
```

- Append new line: `True`

31. `Launch new Microsoft Edge`
- Initial URL: `pUrlAppNomina`
- Launch mode: `Normal`
- Instance output:
  - `BrowserInstance`

32. `Wait for web page content`
- Browser instance: `BrowserInstance`
- Timeout: `30000`

33. `Write text to file`
- File path: `gArchivoLog`
- Text:

```text
Navegador abierto correctamente | URL: %pUrlAppNomina%
```

- Append new line: `True`

34. `Set variable`
- Variable: `gEstadoInicializacion`
- Value: `"OK"`

35. `Write text to file`
- File path: `gArchivoLog`
- Text:

```text
Inicializacion completada correctamente
```

- Append new line: `True`

## Alcance sobre archivos de entrada

`SF_Init` solo realiza una deteccion inicial de archivos Excel pendientes en el directorio configurado para validar que exista carga de trabajo disponible. No debe interpretar estructura, clasificar por tipo funcional ni abrir archivos especificos.

La responsabilidad posterior debe separarse asi:

- `SF_DescubrirArchivosEntrada`
  - lista archivos pendientes
  - clasifica por tipo, por ejemplo `NovedadesNomina` y `HorasExtra`
  - prepara la cola o coleccion de procesamiento
- `SF_ProcesarArchivo`
  - recibe un archivo puntual y su tipo
  - delega lectura y transformacion al subflujo especializado
- `SF_LeerNovedadesNomina`
  - interpreta hojas, tablas y novedades del Excel de nomina
- `SF_LeerHorasExtra`
  - interpreta hojas, tablas y estructura del Excel de horas extra

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
- `Close web browser`
  - Solo si `BrowserInstance` existe y corresponde cerrarlo
- `End`

## Variables adicionales recomendadas

- `gPasswordNomina` `Sensitive text`
- `BrowserInstance` `Browser`
- `gIdEjecucionTmp` `Text`
- `gTotalArchivosPendientes` `Number`

## Reglas practicas

- No dejes la contrasena en logs.
- No abras un archivo Excel especifico en `SF_Init`.
- No proceses datos aqui; solo valida y prepara.
- La lectura de hojas, tablas y registros debe ocurrir en subflujos posteriores por tipo de archivo.
- Si el siguiente subflujo reutiliza la misma sesion web, no cierres el navegador al finalizar `SF_Init`.

## Nota de evolucion

Esta propuesta es una linea base. A medida que se implemente el flujo, puede refinarse para reflejar la realidad del aplicativo, el origen de los datos, la forma final del logging y los mecanismos reales de autenticacion y manejo de errores.
