# SF_Init revisado

Esta propuesta reescribe `SF_Init` con una estructura mas robusta para Power Automate Desktop, integrando manejo de errores tecnicos y corrigiendo varios problemas detectados en la version actual:

- uso inconsistente de `pURLAppNomina` y `pUrlAppNomina`
- uso de `gTotalHojas` cuando el init ya no procesa hojas
- escritura de logs con `Overwrite` donde debe ser `Append`
- construccion de `gArchivoLog` con espacios al inicio
- cadenas con comillas literales innecesarias
- falta de validacion real sobre el resultado de lectura de credenciales

## Criterio de manejo de error

Separar dos tipos de fallos:

- fallos controlados de negocio:
  - no existe directorio de entrada
  - no hay archivos para procesar
  - URL no definida
  - credenciales no recuperables
- fallos tecnicos no esperados:
  - error al escribir log
  - error al listar archivos
  - error al ejecutar PowerShell
  - error al abrir navegador

Los fallos controlados se manejan con `IF ... EXIT FUNCTION`.
Los fallos tecnicos se manejan con `Begin error handling`.

## Flujo sugerido

### Variables base

```text
DateTime.GetCurrentDateTime.Local DateTimeFormat: DateTime.DateTimeFormat.DateAndTime CurrentDateTime=> gFechaHoraInicio
SET gEstadoInicializacion TO $'''INICIANDO'''
SET gMensajeError TO $''''''
SET gSesionAppAbierta TO False
SET gAutenticado TO False
SET gTotalArchivosAProcesar TO 0
Text.ConvertDateTimeToText.FromCustomDateTime DateTime: gFechaHoraInicio CustomFormat: $'''yyyyMMdd_HHmmss''' Result=> gIdEjecucionTmp
SET gIdEjecucion TO $'''HE_%gIdEjecucionTmp%'''
SET pUrlAppNomina TO $'''https://qa.practiko.com.co/madrecol/'''
SET pDirectorioEntrada TO $'''C:\\Dev\\Aspraco\\bandeja_trabajo'''
SET pDirectorioLogs TO $'''C:\\Dev\\Aspraco\\bandeja_trabajo\\logs'''
SET pUsaCredencialesSeguras TO True
```

### Bloque principal

```text
ON BLOCK ERROR
    SET gEstadoInicializacion TO $'''ERROR'''
    SET gMensajeError TO Error.Message

    IF NOT(IsEmpty(gArchivoLog)) THEN
        File.WriteText File: gArchivoLog TextToWrite: $'''ERROR CONTROLADO | %gMensajeError%''' AppendNewLine: True IfFileExists: File.IfFileExists.Append Encoding: File.FileEncoding.UTF8
    END

    IF NOT(IsEmpty(Browser_Practiko)) THEN
        WebAutomation.CloseWebBrowser BrowserInstance: Browser_Practiko
    END

    EXIT FUNCTION
END

IF (Folder.IfFolderExists.DoesNotExist Path: pDirectorioLogs) THEN
    Folder.Create FolderPath: pDirectorioEntrada FolderName: $'''logs''' Folder=> gWorkingDirectory
END

SET gArchivoLog TO $'''%pDirectorioLogs%\\Log_%gIdEjecucion%.txt'''

File.WriteText File: gArchivoLog TextToWrite: $'''Inicio ejecucion | Proceso: Horas Extra | Id: %gIdEjecucion%''' AppendNewLine: True IfFileExists: File.IfFileExists.Append Encoding: File.FileEncoding.UTF8
File.WriteText File: gArchivoLog TextToWrite: $'''Fecha inicio: %gFechaHoraInicio%''' AppendNewLine: True IfFileExists: File.IfFileExists.Append Encoding: File.FileEncoding.UTF8

IF (Folder.IfFolderExists.DoesNotExist Path: pDirectorioEntrada) THEN
    SET gMensajeError TO $'''No se encontro el directorio de entrada configurado'''
    SET gEstadoInicializacion TO $'''ERROR'''
    File.WriteText File: gArchivoLog TextToWrite: $'''ERROR | %gMensajeError% | Directorio entrada: %pDirectorioEntrada%''' AppendNewLine: True IfFileExists: File.IfFileExists.Append Encoding: File.FileEncoding.UTF8
    EXIT FUNCTION
ELSE
    File.WriteText File: gArchivoLog TextToWrite: $'''OK | Directorio de entrada encontrado | %pDirectorioEntrada%''' AppendNewLine: True IfFileExists: File.IfFileExists.Append Encoding: File.FileEncoding.UTF8
END

Folder.GetFiles Folder: pDirectorioEntrada FileFilter: $'''*.xlsx''' IncludeSubfolders: False FailOnAccessDenied: True SortBy1: Folder.SortBy.NoSort SortDescending1: False SortBy2: Folder.SortBy.NoSort SortDescending2: False SortBy3: Folder.SortBy.NoSort SortDescending3: False Files=> gListaArchivosAProcesar
SET gTotalArchivosAProcesar TO gListaArchivosAProcesar.Count

File.WriteText File: gArchivoLog TextToWrite: $'''Total archivos Excel a procesar: %gTotalArchivosAProcesar%''' AppendNewLine: True IfFileExists: File.IfFileExists.Append Encoding: File.FileEncoding.UTF8

IF gTotalArchivosAProcesar = 0 THEN
    SET gMensajeError TO $'''No se encontraron archivos Excel para procesar en el directorio de entrada'''
    SET gEstadoInicializacion TO $'''ERROR'''
    File.WriteText File: gArchivoLog TextToWrite: $'''ERROR | %gMensajeError%''' AppendNewLine: True IfFileExists: File.IfFileExists.Append Encoding: File.FileEncoding.UTF8
    EXIT FUNCTION
ELSE
    File.WriteText File: gArchivoLog TextToWrite: $'''Archivos listos para procesamiento posterior''' AppendNewLine: True IfFileExists: File.IfFileExists.Append Encoding: File.FileEncoding.UTF8
END

IF IsEmpty(pUrlAppNomina) THEN
    SET gMensajeError TO $'''No se definio la URL del aplicativo de nomina'''
    SET gEstadoInicializacion TO $'''ERROR'''
    File.WriteText File: gArchivoLog TextToWrite: $'''ERROR | %gMensajeError%''' AppendNewLine: True IfFileExists: File.IfFileExists.Append Encoding: File.FileEncoding.UTF8
    EXIT FUNCTION
ELSE
    File.WriteText File: gArchivoLog TextToWrite: $'''OK | URL aplicativo inicializada | %pUrlAppNomina%''' AppendNewLine: True IfFileExists: File.IfFileExists.Append Encoding: File.FileEncoding.UTF8
END

IF pUsaCredencialesSeguras = True THEN
    Scripting.RunPowershellScript.RunScript Script: $'''$TargetName = "practiko"
$CredentialType = 1

$Code = @'
using System;
using System.Runtime.InteropServices;

public class NativeCreds {
    [DllImport("Advapi32.dll", SetLastError = true, CharSet = CharSet.Unicode)]
    public static extern bool CredReadW(
        string target,
        uint type,
        int reserved,
        out IntPtr credentialPtr
    );

    [DllImport("Advapi32.dll", SetLastError = true)]
    public static extern void CredFree(IntPtr buffer);

    [StructLayout(LayoutKind.Sequential, CharSet = CharSet.Unicode)]
    public struct CREDENTIAL {
        public uint Flags;
        public uint Type;
        public string TargetName;
        public string Comment;
        public System.Runtime.InteropServices.ComTypes.FILETIME LastWritten;
        public uint CredentialBlobSize;
        public IntPtr CredentialBlob;
        public uint Persist;
        public uint AttributeCount;
        public IntPtr Attributes;
        public string TargetAlias;
        public string UserName;
    }
}
'@

if (-not ("NativeCreds" -as [type])) {
    Add-Type -TypeDefinition $Code
}

$credPtr = [IntPtr]::Zero

if ([NativeCreds]::CredReadW($TargetName, $CredentialType, 0, [ref]$credPtr)) {
    try {
        $cred = [System.Runtime.InteropServices.Marshal]::PtrToStructure(
            $credPtr,
            [Type][NativeCreds+CREDENTIAL]
        )

        $bytes = New-Object byte[] $cred.CredentialBlobSize
        [System.Runtime.InteropServices.Marshal]::Copy(
            $cred.CredentialBlob,
            $bytes,
            0,
            $cred.CredentialBlobSize
        )

        $password = [System.Text.Encoding]::Unicode.GetString($bytes).TrimEnd([char]0)

        [Console]::WriteLine($cred.UserName)
        [Console]::WriteLine($password)
    }
    finally {
        [NativeCreds]::CredFree($credPtr)
    }
}
else {
    $win32Error = [System.Runtime.InteropServices.Marshal]::GetLastWin32Error()
    [Console]::WriteLine("ERROR_CREDREAD_$win32Error")
    [Console]::WriteLine("ERROR_CREDREAD_$win32Error")
}''' ScriptOutput=> gCredentialManagerString

    Text.SplitText.Split Text: gCredentialManagerString StandardDelimiter: Text.StandardDelimiter.NewLine DelimiterTimes: 1 Result=> gCredenciales

    IF gCredenciales.Count < 2 THEN
        SET gMensajeError TO $'''No fue posible interpretar la respuesta de credenciales'''
        SET gEstadoInicializacion TO $'''ERROR'''
        File.WriteText File: gArchivoLog TextToWrite: $'''ERROR | %gMensajeError%''' AppendNewLine: True IfFileExists: File.IfFileExists.Append Encoding: File.FileEncoding.UTF8
        EXIT FUNCTION
    END

    IF Text.StartsWith(Text: gCredenciales[0], Value: $'''ERROR_CREDREAD_''', IgnoreCase: False) THEN
        SET gMensajeError TO $'''No fue posible leer las credenciales desde Credential Manager'''
        SET gEstadoInicializacion TO $'''ERROR'''
        File.WriteText File: gArchivoLog TextToWrite: $'''ERROR | %gMensajeError%''' AppendNewLine: True IfFileExists: File.IfFileExists.Append Encoding: File.FileEncoding.UTF8
        EXIT FUNCTION
    END

    SET gUsuarioNomina TO gCredenciales[0]
    SET gPasswordNomina TO gCredenciales[1]
    File.WriteText File: gArchivoLog TextToWrite: $'''Configuracion credenciales | Seguras: %pUsaCredencialesSeguras% | Usuario: %gUsuarioNomina%''' AppendNewLine: True IfFileExists: File.IfFileExists.Append Encoding: File.FileEncoding.UTF8
ELSE
    SET gMensajeError TO $'''No fue posible leer las credenciales de acceso del Credential Manager'''
    SET gEstadoInicializacion TO $'''ERROR'''
    File.WriteText File: gArchivoLog TextToWrite: $'''ERROR | %gMensajeError%''' AppendNewLine: True IfFileExists: File.IfFileExists.Append Encoding: File.FileEncoding.UTF8
    EXIT FUNCTION
END

WebAutomation.LaunchEdge.LaunchEdge Url: pUrlAppNomina ClearCache: False ClearCookies: False WindowState: WebAutomation.BrowserWindowState.Normal WaitForPageToLoadTimeout: 60 Timeout: 60 PiPUserDataFolderMode: WebAutomation.PiPUserDataFolderModeEnum.AutomaticProfile TargetDesktop: $'''{"DisplayName":"Local computer","Route":{"ServerType":"Local","ServerAddress":""},"DesktopType":"local"}''' BrowserInstance=> Browser_Practiko

SET gSesionAppAbierta TO True

File.WriteText File: gArchivoLog TextToWrite: $'''Navegador abierto correctamente | URL: %pUrlAppNomina%''' AppendNewLine: True IfFileExists: File.IfFileExists.Append Encoding: File.FileEncoding.UTF8
SET gEstadoInicializacion TO $'''OK'''
File.WriteText File: gArchivoLog TextToWrite: $'''Inicializacion completada correctamente''' AppendNewLine: True IfFileExists: File.IfFileExists.Append Encoding: File.FileEncoding.UTF8
```

## Cambios recomendados frente a la version actual

### 1. Unificar nombres de variables

Usar siempre:

- `pUrlAppNomina`
- `gListaArchivosAProcesar`
- `gTotalArchivosAProcesar`

Evitar variantes como:

- `pURLAppNomina`
- `gTotalHojas`

### 2. Dejar de usar `Overwrite` en el log

En `SF_Init` todas las escrituras de log deben ser `Append`, excepto que intencionalmente se quiera reiniciar el archivo.

### 3. Evitar comillas literales innecesarias

Incorrecto:

```text
SET gMensajeError TO $'''\"ERROR\"'''
```

Correcto:

```text
SET gMensajeError TO $'''ERROR'''
```

### 4. Evitar `LABEL/GOTO` para la carpeta de logs

No hace falta reintentar por salto. Es suficiente:

- validar si existe
- crear si no existe
- continuar

### 5. Mover configuracion a `config.json`

La siguiente evolucion deberia sacar de `SF_Init` estos valores quemados:

- `pUrlAppNomina`
- `pDirectorioEntrada`
- `pDirectorioLogs`
- alias de credencial como `practiko`

### 6. Extraer lectura de credenciales

Conviene mover el bloque PowerShell a un subflujo como `SF_CargarCredenciales`, para que `SF_Init` quede mas limpio y el manejo de errores sea mas local.
