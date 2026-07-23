# Registro Horas Extra

Frente de prueba para PAD enfocado en calcular y registrar una hora de salida en un control especial tipo reloj.

## Alcance actual

- Usa una sola hoja fija del Excel: `Cesar Gomez`.
- Mantiene un patron de lectura lineal por fila.
- Arranca en la fila `3`.
- Usa la fecha de la columna `B`, la jornada ordinaria de la columna `J` y las horas extras diurnas de la columna `K`.
- Termina cuando encuentra vacia la columna `B`.
- Toma un `IdEmpleado` quemado: `7937156`.
- Calcula la hora final de salida y la deja lista para poblarla en PAD.

## Lectura fuente

- Usa la misma bandeja de trabajo del flujo principal: la ruta configurada en `rutas.directorioEntrada`.
- En esa bandeja busca un unico `.xlsx` cuyo nombre contenga `hora` o `extra`.
- Descarta archivos temporales `~$`.
- Si encuentra mas de un candidato, exige definir `pRutaArchivoRegistroHorasExtra` de forma explicita.

- Fila inicial: `3`
- Fecha: columna `B`
- Mant. Hora Ordinario: columna `J`
- Mant. Horas Extras Diurna: columna `K`

Si una fila tiene fecha en `B` pero viene vacia en `J` o `K`, el flujo la omite y sigue con la siguiente.

## Regla de negocio implementada

La columna `J3` no es una hora del reloj. Define una hora base de salida:

- `8.5` -> `16:30`
- `7.5` -> `15:30`
- `2.5` -> `08:30`

La columna `K3` representa horas decimales extras. Se convierten a minutos con esta regla:

- `1.12` -> `67` minutos -> `01:07`
- `0.55` -> `33` minutos
- `1.50` -> `90` minutos -> `01:30`

Luego se calcula:

- `HoraSalida = HoraBase + HorasExtrasDiurnas`

El resultado operativo para PAD queda preparado en formato `HH:mm` de 24 horas, más `HH` y `mm` por separado.

## Salida operativa para PAD

El flujo deja listas estas variables:

- `gFechaUIRegistroHorasExtraActual`
- `gHoraSalidaUI24RegistroHorasExtraActual`
- `gHoraSalidaHHRegistroHorasExtraActual`
- `gHoraSalidaMMRegistroHorasExtraActual`

## Corte UI propuesto

- `SF_PrepararContextoRegistroHorasExtra`: autenticacion, empresa y llegada a `Nomina electronica > Novedades`.
- `SF_RegistrarNovedadHorasExtra`: apertura de la novedad base de HE.
- `SF_RegistrarDetalleHorasExtraEmpleado`: diligenciamiento del detalle del empleado, fecha y reloj.
- `SF_RegistrarRegistroHorasExtra`: orquestador que llama los dos subflujos anteriores.

## Estrategia sugerida para el control reloj

1. Intentar texto directo `HH:mm`.
2. Si el control segmenta hora/minuto, escribir `HH`, `TAB`, `mm`.
3. Si el widget bloquea texto, mapear spinner o flechas sobre cada segmento.

## Nota

Por ahora no se escribe resultado de vuelta en el Excel, porque no definiste una celda de salida ni columnas de estado para esta hoja real. El seguimiento queda en el log del flujo.
