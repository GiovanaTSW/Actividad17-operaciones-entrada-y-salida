# Actividad #17: Reporte de Laboratorio

## Operaciones de Entrada/Salida en Windows — desde la personalizacion hasta el sistema

**Tecnologico de Software — Sistemas Operativos**
**Unidad II: Memoria y Procesos de Entrada/Salida**
**Profesor:** Jorge Javier Pedrozo Romero

---

## ¿Qué vas a hacer en este laboratorio?

Cuando conectas una USB a tu computadora, escuchas un sonido, aparece una notificacion y el sistema carga el dispositivo automaticamente. Lo que parece simple es en realidad una cadena de operaciones de Entrada/Salida gestionadas por el sistema operativo: el controlador detecta el hardware, el manejador (driver) lo registra, el administrador Plug and Play asigna recursos, y el shell de Windows reproduce el sonido y actualiza el Explorador.

En este laboratorio vas a explorar ese proceso en dos niveles: primero desde afuera, personalizando sonidos e iconos directamente desde PowerShell para ver como Windows gestiona eventos de dispositivo en el registro y el sistema de archivos. Despues desde adentro, monitoreando y midiendo operaciones de E/S en tiempo real.

**Objetivo:** Comprender el sistema de Entrada/Salida de Windows a traves de la personalizacion de eventos de dispositivo y la exploracion programatica de las operaciones de E/S, usando PowerShell como herramienta principal.

## Estructura del laboratorio

| # | Parte | Que haces | Puntos |
|---|-------|-----------|--------|
| 0 | Preparacion del entorno | Abrir PowerShell, Explorador y descargar archivos | — |
| 1 | Cambio de sonidos del sistema | Personalizas los sonidos de conexion/desconexion de USB desde PowerShell | 2.0 |
| 2 | Cambio de iconos | Cambias iconos de carpeta o unidad USB desde PowerShell | 1.5 |
| 3 | Monitoreo de dispositivos en tiempo real | Observas los eventos internos al conectar un USB | 2.0 |
| 4 | Operaciones de E/S desde codigo | Mides tiempos de lectura/escritura y comparas E/S bloqueante vs. asincrona | 1.5 |
| 5 | Reporte y reversion de cambios | Completas el reporte y restauras la configuracion original | 1.0 |
| | **TOTAL** | | **8.0** |

---

## Parte 0 — Preparacion del entorno

Antes de comenzar la practica, configura tu espacio de trabajo siguiendo estos pasos en orden.

### Paso 1 — Abre PowerShell como administrador

1. Presiona `Win + X` en el teclado.
2. En el menu que aparece, selecciona **Windows PowerShell (Administrador)** o **Terminal (Administrador)** segun tu version de Windows.
3. Si Windows te pide confirmacion de Control de Cuentas de Usuario (UAC), haz clic en **Si**.
4. Deja la ventana de PowerShell abierta. La usaras durante toda la practica.

> **Importante:** Esta practica requiere PowerShell con permisos de administrador. Sin ellos, algunos comandos fallaran al modificar el registro de Windows.

### Paso 2 — Abre el Explorador de Windows en el Escritorio

1. Presiona `Win + E` para abrir el Explorador de Windows.
2. En la barra lateral izquierda, haz clic en **Escritorio**.
3. Haz clic en la barra de direcciones (la parte superior donde aparece la ruta).
4. La ruta del Escritorio se seleccionara automaticamente. Copia esa ruta con `Ctrl + C`.

La ruta del Escritorio normalmente tiene este formato:

```text
C:\Users\TuNombreDeUsuario\Desktop
```

### Paso 3 — Navega al Escritorio desde PowerShell

En la ventana de PowerShell, escribe `cd` seguido de la ruta que copiaste. Por ejemplo:

```powershell
cd "C:\Users\TuNombreDeUsuario\Desktop"
```

Sustituye `TuNombreDeUsuario` por tu nombre real de usuario. Presiona Enter. El prompt de PowerShell debera mostrar que estas en la carpeta del Escritorio.

### Paso 4 — Descarga los archivos de la practica

Descarga los siguientes archivos y guardalos directamente en el Escritorio:

| Archivo | Descripcion |
|---------|-------------|
| `actividad17_windows.html` | Este archivo de instrucciones |
| `conexion.wav` | Sonido personalizado para cuando se conecta un USB |
| `desconexion.wav` | Sonido personalizado para cuando se desconecta un USB |

Puedes verificar que los archivos estan en el Escritorio ejecutando en PowerShell:

```powershell
Get-ChildItem -Path "PATH" | Select-Object Name
```

Debes ver en la lista los tres archivos antes de continuar.

### Paso 5 — Guarda la configuracion original (antes de cambiar nada)

Antes de modificar cualquier configuracion, guarda los valores originales de los sonidos del sistema. Esto te permitira revertir los cambios al terminar la practica.

```powershell
# Leer y mostrar los sonidos actuales antes de modificarlos
$originalConexion    = (Get-ItemProperty `
    "HKCU:\AppEvents\Schemes\Apps\.Default\DeviceConnect\.Current")."(Default)"
$originalDesconexion = (Get-ItemProperty `
    "HKCU:\AppEvents\Schemes\Apps\.Default\DeviceDisconnect\.Current")."(Default)"

Write-Host "Sonido original de conexion:    $originalConexion"
Write-Host "Sonido original de desconexion: $originalDesconexion"
Write-Host ""
Write-Host "Anota estos valores. Los necesitaras para revertir los cambios al final."
```

Anota los valores que aparezcan. Si el campo aparece vacio, tambien anotalo. Los usaras en la Parte 5 para restaurar la configuracion.

---

## Parte 1 — Cambio de sonidos del sistema (2 pts)

Los sonidos que Windows reproduce al conectar o desconectar un dispositivo son eventos registrados en el registro del sistema operativo, bajo la clave `HKCU:\AppEvents`. Modificarlos desde PowerShell es una operacion de escritura directa en el registro: una operacion de E/S real.

En esta parte usaras los archivos `conexion.wav` y `desconexion.wav` que descargaste en el Escritorio.

### 1.1 — Explorar los sonidos disponibles en el sistema

Primero observa que eventos de sonido tiene registrados Windows y cuales estan activos:

```powershell
# Ver todos los eventos de sonido registrados en el sistema
Get-ChildItem "HKCU:\AppEvents\Schemes\Apps\.Default" |
    Select-Object PSChildName |
    Sort-Object PSChildName

# Ver el sonido asignado actualmente a conexion y desconexion de dispositivo
Write-Host ""
Write-Host "--- Sonido actual de conexion ---"
Get-ItemProperty "HKCU:\AppEvents\Schemes\Apps\.Default\DeviceConnect\.Current" |
    Select-Object "(Default)"

Write-Host "--- Sonido actual de desconexion ---"
Get-ItemProperty "HKCU:\AppEvents\Schemes\Apps\.Default\DeviceDisconnect\.Current" |
    Select-Object "(Default)"
```

**Que hace cada parte de este comando:**

| Comando | Que hace | Por que importa |
|---------|----------|-----------------|
| `Get-ChildItem "HKCU:\AppEvents..."` | Lista todas las claves de eventos de sonido en el registro | Muestra la estructura interna donde Windows guarda la configuracion de audio del sistema |
| `Get-ItemProperty "...DeviceConnect\.Current"` | Lee el valor actual del sonido asignado al evento de conexion de dispositivo | Es la ruta del archivo .wav que Windows reproduce cuando conectas un USB |
| `Select-Object "(Default)"` | Muestra solo el campo Default, que contiene la ruta del archivo de audio | Filtra la informacion relevante del resto de metadatos de la clave |

### 1.2 — Asignar los archivos WAV descargados

Ahora asigna los archivos `conexion.wav` y `desconexion.wav` del Escritorio como los nuevos sonidos del sistema:

```powershell
# Rutas de los archivos WAV en el Escritorio
$wavConexion    = "PATH\conexion.wav"
$wavDesconexion = "PATH\desconexion.wav"

# Verificar que los archivos existen antes de asignarlos
if (-not (Test-Path $wavConexion)) {
    Write-Host "ERROR: No se encuentra conexion.wav en el Escritorio." -ForegroundColor Red
    return
}
if (-not (Test-Path $wavDesconexion)) {
    Write-Host "ERROR: No se encuentra desconexion.wav en el Escritorio." -ForegroundColor Red
    return
}

# Escribir las nuevas rutas en el registro de Windows
Set-ItemProperty `
    -Path "HKCU:\AppEvents\Schemes\Apps\.Default\DeviceConnect\.Current" `
    -Name "(Default)" `
    -Value $wavConexion

Set-ItemProperty `
    -Path "HKCU:\AppEvents\Schemes\Apps\.Default\DeviceDisconnect\.Current" `
    -Name "(Default)" `
    -Value $wavDesconexion

Write-Host "Sonido de conexion    -> $wavConexion"
Write-Host "Sonido de desconexion -> $wavDesconexion"
Write-Host "Cambios aplicados correctamente."
```

**Que hace cada parte de este comando:**

| Comando | Que hace | Por que importa |
|---------|----------|-----------------|
| `PATH\conexion.wav` | Construye la ruta completa al archivo WAV usando la variable de entorno del usuario | Hace el script portable: funciona para cualquier nombre de usuario sin cambiar codigo |
| `Test-Path $wavConexion` | Verifica que el archivo existe en el Escritorio antes de intentar asignarlo | Evita errores silenciosos: si el archivo no esta, el script lo dice explicitamente |
| `Set-ItemProperty -Path "HKCU:\AppEvents..."` | Escribe la nueva ruta del archivo WAV en el registro de Windows | Esta es la operacion de E/S principal: una escritura directa al registro del sistema operativo |

### 1.3 — Verificar y probar el cambio

```powershell
# Leer los valores actuales del registro para confirmar el cambio
$actual = (Get-ItemProperty `
    "HKCU:\AppEvents\Schemes\Apps\.Default\DeviceConnect\.Current")."(Default)"
Write-Host "Sonido de conexion activo: $actual"

# Reproducir el sonido directamente desde PowerShell para verificarlo
$player = New-Object System.Media.SoundPlayer $actual
$player.PlaySync()
Write-Host "Sonido reproducido. Ahora desconecta y vuelve a conectar tu USB para confirmarlo."
```

**Que hace cada parte de este comando:**

| Comando | Que hace | Por que importa |
|---------|----------|-----------------|
| `Get-ItemProperty "...DeviceConnect\.Current"` | Vuelve a leer el registro para confirmar que el valor se guardo correctamente | Valida que la operacion de escritura anterior tuvo efecto real en el sistema |
| `New-Object System.Media.SoundPlayer` | Crea un objeto reproductor de audio de la libreria .NET del sistema | Demuestra que PowerShell puede interactuar directamente con componentes del sistema operativo |
| `$player.PlaySync()` | Reproduce el archivo WAV de forma sincrona (bloqueante): PowerShell espera a que termine | Ejemplo directo de E/S bloqueante: el proceso no continua hasta que la reproduccion termina |

> 🎥 **Graba un video con sonido para esta parte**
>
> El video debe mostrar: la ejecucion de los tres scripts (1.1, 1.2 y 1.3) en PowerShell, y la prueba real conectando y desconectando el USB para escuchar el nuevo sonido. Debe escucharse claramente el audio.

---

## Parte 2 — Cambio de iconos (1.5 pts)

Los iconos de carpetas en Windows se almacenan como metadatos en un archivo oculto llamado `desktop.ini` dentro de cada carpeta. Cambiar un icono desde PowerShell implica crear ese archivo y escribir la referencia al icono: otra operacion de E/S sobre el sistema de archivos.

Elige una de las dos opciones: cambiar el icono de una carpeta (**Opcion A**) o cambiar el icono de la unidad USB (**Opcion B**). Puedes hacer ambas si quieres.

### 2.1 — Explorar los iconos disponibles en el sistema

```powershell
# Verificar que existen los archivos que contienen iconos del sistema
$archivos = @(
    "C:\Windows\System32\shell32.dll",
    "C:\Windows\System32\imageres.dll",
    "C:\Windows\System32\moricons.dll"
)

foreach ($a in $archivos) {
    $existe = Test-Path $a
    Write-Host "$a  ->  Existe: $existe"
}
```

**Que hace este comando:**

| Comando | Que hace | Por que importa |
|---------|----------|-----------------|
| `@("ruta1","ruta2",...)` | Crea un arreglo con las rutas de los archivos a verificar | `shell32.dll`, `imageres.dll` y `moricons.dll` son las bibliotecas del sistema que contienen los iconos que Windows usa en toda su interfaz |
| `Test-Path $a` | Verifica si el archivo existe en esa ruta | Confirma que los recursos del sistema estan disponibles antes de referenciarlos |

> **Dato:** `shell32.dll` contiene mas de 300 iconos con indices numericos. El icono de la carpeta amarilla clasica es el indice 3. Puedes probar distintos indices (0, 3, 20, 45, 100...) para ver diferentes opciones.

### Opcion A — Cambiar el icono de una carpeta

```powershell
# Define la carpeta a personalizar (puedes cambiar el nombre)
$carpeta = "PATH\MiCarpetaOS"
New-Item -ItemType Directory -Path $carpeta -Force | Out-Null

# Elige el icono: cambia el indice para probar distintos (0, 3, 20, 45, 100...)
$archivoIcono = "C:\Windows\System32\shell32.dll"
$indiceIcono  = 20

# Crear el archivo desktop.ini con la referencia al icono
$contenido = @"
[.ShellClassInfo]
IconResource=$archivoIcono,$indiceIcono
[ViewState]
Mode=
Vid=
FolderType=Generic
"@

$iniPath = Join-Path $carpeta "desktop.ini"
Set-Content -Path $iniPath -Value $contenido -Encoding Unicode

# Marcar la carpeta y el archivo como sistema para que Windows respete el icono
attrib +s +h $iniPath
attrib +r $carpeta

Write-Host "Icono aplicado en: $carpeta"
Write-Host "Archivo desktop.ini creado en: $iniPath"

# Reiniciar el Explorador para que actualice la vista
Stop-Process -Name explorer -Force
Start-Sleep -Seconds 2
Start-Process explorer
Write-Host "Explorador reiniciado. Verifica el nuevo icono en el Escritorio."
```

**Que hace cada parte de este comando:**

| Comando | Que hace | Por que importa |
|---------|----------|-----------------|
| `New-Item -ItemType Directory` | Crea la carpeta si no existe | Operacion de E/S: crea un nuevo nodo en el sistema de archivos |
| `Set-Content -Path $iniPath` | Escribe el archivo `desktop.ini` con la referencia al icono elegido | Es la operacion de escritura principal: Windows lee este archivo para saber que icono mostrar en la carpeta |
| `attrib +s +h $iniPath` | Marca el archivo `desktop.ini` como sistema (+s) y oculto (+h) | Windows solo respeta el `desktop.ini` si tiene estos atributos; sin ellos ignora la configuracion de icono |
| `Stop-Process -Name explorer` / `Start-Process explorer` | Reinicia el proceso `Explorer.exe` | Fuerza al shell de Windows a volver a leer los metadatos de las carpetas y mostrar el nuevo icono |

### Opcion B — Cambiar el icono de la unidad USB

```powershell
# Detectar automaticamente la letra de la unidad USB conectada
$usb = Get-WmiObject Win32_LogicalDisk |
    Where-Object { $_.DriveType -eq 2 } |
    Select-Object -First 1

if ($usb) {
    $letraUSB = $usb.DeviceID
    Write-Host "USB detectada en: $letraUSB"

    # Crear autorun.inf con el icono elegido (cambia el indice para probar otros)
    $autorun = @"
[AutoRun]
icon=C:\Windows\System32\shell32.dll,7
label=Mi USB - Sistemas Operativos
"@
    Set-Content -Path "$letraUSB\autorun.inf" -Value $autorun -Encoding ASCII
    Write-Host "Archivo autorun.inf creado en: $letraUSB"
    Write-Host "Desconecta y vuelve a conectar el USB para ver el icono."
} else {
    Write-Host "No se detecto ninguna unidad USB conectada."
}
```

**Que hace cada parte de este comando:**

| Comando | Que hace | Por que importa |
|---------|----------|-----------------|
| `Get-WmiObject Win32_LogicalDisk` | Consulta todos los discos logicos del sistema via WMI | WMI (Windows Management Instrumentation) es la interfaz de administracion del sistema operativo; acceder a ella es una operacion de E/S interna |
| `Where-Object { $_.DriveType -eq 2 }` | Filtra solo las unidades de tipo 2, que corresponden a medios removibles (USB) | DriveType 2 = disco removible. Otros tipos: 3 = disco fijo, 4 = red, 5 = CD/DVD |
| `Set-Content autorun.inf` | Escribe el archivo `autorun.inf` en la raiz de la unidad USB | Windows lee este archivo al montar la unidad para obtener configuracion como el icono y la etiqueta |

> 🎥 **Graba un video para esta parte**
>
> El video debe mostrar: la ejecucion de los scripts en PowerShell y el resultado visible en el Explorador de Windows (el icono cambiado en la carpeta o en la unidad USB).

---

## Parte 3 — Monitoreo de dispositivos en tiempo real (2 pts)

Ahora vamos a ver lo que ocurre por dentro cuando conectas un USB. Windows genera eventos internos que podemos capturar en tiempo real con PowerShell usando WMI.

### 3.1 — Capturar eventos de conexion/desconexion en tiempo real

```powershell
# Registrar un observador de eventos de cambio de dispositivo
$accion = {
    $e = $Event.SourceEventArgs.NewEvent
    $tipo = switch ($e.EventType) {
        1 { "Configuracion cambiada" }
        2 { "Dispositivo conectado" }
        3 { "Dispositivo desconectado" }
        4 { "Nodo de dock cambiado" }
    }
    Write-Host "[$((Get-Date).ToString('HH:mm:ss'))]  Evento tipo $($e.EventType): $tipo"
}

Register-WmiEvent -Query "SELECT * FROM Win32_DeviceChangeEvent" -Action $accion
Write-Host "Monitoreo activo. Conecta y desconecta tu USB (tienes 30 segundos)..."
Start-Sleep -Seconds 30
Get-EventSubscriber | Unregister-Event -ErrorAction SilentlyContinue
Write-Host "Monitoreo finalizado."
```

**Que hace cada parte de este comando:**

| Comando | Que hace | Por que importa |
|---------|----------|-----------------|
| `Register-WmiEvent` | Suscribe PowerShell a los eventos WMI de cambio de dispositivo en tiempo real | Demuestra la E/S no bloqueante: PowerShell no se queda esperando; Windows lo notifica cuando ocurre el evento |
| `-Query "SELECT * FROM Win32_DeviceChangeEvent"` | Consulta WQL (lenguaje de WMI) que indica que eventos escuchar | `Win32_DeviceChangeEvent` es la clase del sistema que representa cualquier cambio en dispositivos conectados |
| `-Action { ... }` | Bloque de codigo que se ejecuta automaticamente cada vez que llega un evento | Esto es un callback asincrono: el sistema llama a este codigo cuando detecta el evento, sin que el script lo este pidiendo activamente |
| `$e.EventType` | Numero que identifica el tipo de evento: 2 = conectado, 3 = desconectado | Estos codigos vienen directamente del manejador de dispositivos de Windows |
| `Unregister-Event` | Cancela la suscripcion al evento al terminar | Buena practica: libera el recurso del sistema operativo que estaba siendo usado para el monitoreo |

### 3.2 — Listar dispositivos activos y sus manejadores

```powershell
# Todos los dispositivos activos con su clase de driver
Get-PnpDevice | Where-Object Status -eq 'OK' |
    Select-Object FriendlyName, Class, InstanceId |
    Sort-Object Class |
    Format-Table -AutoSize

# Solo los dispositivos USB
Write-Host ""
Write-Host "--- Dispositivos USB detectados ---"
Get-PnpDevice | Where-Object { $_.InstanceId -like "USB*" } |
    Select-Object FriendlyName, Status, InstanceId |
    Format-Table -AutoSize
```

**Que hace cada parte de este comando:**

| Comando | Que hace | Por que importa |
|---------|----------|-----------------|
| `Get-PnpDevice` | Lista todos los dispositivos Plug and Play que Windows tiene registrados | Muestra el resultado del trabajo del administrador Plug and Play: cada dispositivo que aparece fue procesado por una cadena de manejadores |
| `Where-Object Status -eq 'OK'` | Filtra solo los dispositivos que estan funcionando correctamente | Los dispositivos con error tienen su propio driver fallido o faltante; los OK tienen un manejador activo |
| `InstanceId -like "USB*"` | Filtra los dispositivos cuyo identificador empieza con USB | El Instance ID es el identificador unico que el sistema asigna a cada dispositivo; los USB siempre empiezan con ese prefijo |

> 🎥 **Graba un video con sonido para esta parte**
>
> El video debe mostrar: el script de monitoreo corriendo, la accion de conectar y desconectar el USB, y los eventos que aparecen en PowerShell. Tambien la ejecucion del script de listado de dispositivos.

---

## Parte 4 — Operaciones de E/S desde codigo (1.5 pts)

Ahora vas a medir directamente las operaciones de lectura y escritura del sistema de E/S, y comparar el comportamiento bloqueante con el asincrono.

### 4.1 — Lectura y escritura con medicion de tiempos

```powershell
$archivo = "PATH\practica_es.txt"
$datos   = "Operaciones de E/S en Windows - Sistemas Operativos - " + (Get-Date)

# ESCRITURA BLOQUEANTE
$sw = [System.Diagnostics.Stopwatch]::StartNew()
[System.IO.File]::WriteAllText($archivo, $datos)
$sw.Stop()
Write-Host "Escritura completada en: $($sw.Elapsed.TotalMilliseconds) ms"
Write-Host "Archivo creado en: $archivo"

# LECTURA BLOQUEANTE
$sw = [System.Diagnostics.Stopwatch]::StartNew()
$contenido = [System.IO.File]::ReadAllText($archivo)
$sw.Stop()
Write-Host "Lectura completada en:   $($sw.Elapsed.TotalMilliseconds) ms"
Write-Host "Contenido leido: $contenido"
```

**Que hace cada parte de este comando:**

| Comando | Que hace | Por que importa |
|---------|----------|-----------------|
| `Stopwatch::StartNew()` | Inicia un cronometro de alta precision | Mide el tiempo real que el sistema operativo tarda en completar la operacion de E/S |
| `[System.IO.File]::WriteAllText()` | Escribe el contenido completo en el archivo de forma bloqueante | El script no avanza hasta que el sistema confirma que la escritura termino: esto es E/S bloqueante |
| `[System.IO.File]::ReadAllText()` | Lee el contenido completo del archivo de forma bloqueante | Igual que la escritura: el proceso espera hasta tener todos los datos antes de continuar |
| `$sw.Elapsed.TotalMilliseconds` | Muestra el tiempo transcurrido en milisegundos | Permite comparar cuanto tarda la E/S bloqueante vs. la asincrona en el siguiente script |

### 4.2 — E/S asincrona (no bloqueante)

```powershell
$archivo = "PATH\practica_asincrona.txt"
$datos   = [System.Text.Encoding]::UTF8.GetBytes("Escritura asincrona - " + (Get-Date))
$stream  = [System.IO.File]::Open($archivo, [System.IO.FileMode]::Create)

$sw = [System.Diagnostics.Stopwatch]::StartNew()

# Iniciar la escritura de forma asincrona (no bloqueante)
$tarea = $stream.WriteAsync($datos, 0, $datos.Length)

# El script continua ejecutandose mientras la escritura ocurre en segundo plano
Write-Host "Escritura iniciada en segundo plano..."
Write-Host "El script sigue corriendo mientras escribe (E/S no bloqueante)"
Write-Host "Tiempo hasta aqui: $($sw.Elapsed.TotalMilliseconds) ms"

# Esperar a que la tarea termine para cerrar el archivo correctamente
$tarea.Wait()
$stream.Close()

$sw.Stop()
Write-Host "Escritura asincrona completada."
Write-Host "Tiempo total: $($sw.Elapsed.TotalMilliseconds) ms"
```

**Que hace cada parte de este comando:**

| Comando | Que hace | Por que importa |
|---------|----------|-----------------|
| `[System.IO.File]::Open()` | Abre un stream de archivo para escritura, a diferencia de `WriteAllText` que lo abre y cierra internamente | Un stream abierto es necesario para poder usar metodos asincronos; da control directo sobre la operacion de E/S |
| `$stream.WriteAsync()` | Inicia la escritura en segundo plano y devuelve inmediatamente una tarea (`Task`) | Esta es la diferencia clave con E/S bloqueante: el metodo regresa el control al script sin esperar a que termine |
| `$tarea.Wait()` | Espera a que la tarea asincrona termine antes de cerrar el archivo | Aunque la escritura fue asincrona, hay que esperar a que termine antes de cerrar el stream para no corromper el archivo |

> **Conexion con la practica:** El sonido que escuchaste al conectar el USB en la Parte 1 fue reproducido de forma asincrona: Windows no paro todo el sistema para ejecutarlo. Utilizo el mismo principio que acabas de programar.

### 4.3 — Monitorear la E/S con el Monitor de Rendimiento

1. Ejecuta este comando para abrir el Monitor de Rendimiento: `perfmon`
2. En el panel izquierdo ve a **Monitor de Rendimiento** y haz clic en el boton `+`.
3. Agrega los contadores: **PhysicalDisk → Disk Read Bytes/sec**, **Disk Write Bytes/sec** y **Disk Transfers/sec**.
4. Ejecuta el script de la seccion 4.1 varias veces seguidas y observa los picos de actividad.

> 🎥 **Graba un video para esta parte**
>
> El video debe mostrar: la ejecucion de ambos scripts (bloqueante y asincrono) con sus tiempos visibles, y el Monitor de Rendimiento con los picos de actividad de disco durante la ejecucion.

---

## Parte 5 — Reporte y reversion de cambios (1 pt)

### 5.1 — Revertir todos los cambios

Al terminar la practica, restaura la configuracion original de tu sistema. Ejecuta los siguientes scripts en orden.

#### Revertir Parte 1 — Restaurar sonidos originales

```powershell
# Sustituye los valores entre comillas por los que anotaste en el Paso 5 de la Parte 0
# Si estaban vacios originalmente, usa una cadena vacia: ""
$sonidoOriginalConexion    = "C:\Windows\Media\Windows Hardware Insert.wav"
$sonidoOriginalDesconexion = "C:\Windows\Media\Windows Hardware Remove.wav"

Set-ItemProperty `
    -Path "HKCU:\AppEvents\Schemes\Apps\.Default\DeviceConnect\.Current" `
    -Name "(Default)" `
    -Value $sonidoOriginalConexion

Set-ItemProperty `
    -Path "HKCU:\AppEvents\Schemes\Apps\.Default\DeviceDisconnect\.Current" `
    -Name "(Default)" `
    -Value $sonidoOriginalDesconexion

Write-Host "Sonidos restaurados a la configuracion original."
```

#### Revertir Parte 2 Opcion A — Restaurar icono de carpeta

```powershell
$carpeta = "PATH\MiCarpetaOS"
$iniPath = Join-Path $carpeta "desktop.ini"

# Quitar atributos de sistema para poder eliminar el archivo
attrib -s -h $iniPath
attrib -r $carpeta

# Eliminar el archivo desktop.ini
Remove-Item -Path $iniPath -Force -ErrorAction SilentlyContinue
Write-Host "desktop.ini eliminado. El icono volvera al predeterminado."

# Reiniciar el Explorador para aplicar el cambio
Stop-Process -Name explorer -Force
Start-Sleep -Seconds 2
Start-Process explorer
Write-Host "Explorador reiniciado."
```

#### Revertir Parte 2 Opcion B — Eliminar autorun.inf del USB

```powershell
$usb = Get-WmiObject Win32_LogicalDisk |
    Where-Object { $_.DriveType -eq 2 } |
    Select-Object -First 1

if ($usb) {
    $letraUSB = $usb.DeviceID
    Remove-Item -Path "$letraUSB\autorun.inf" -Force -ErrorAction SilentlyContinue
    Write-Host "autorun.inf eliminado de $letraUSB"
    Write-Host "Desconecta y vuelve a conectar el USB para ver el icono predeterminado."
} else {
    Write-Host "No se detecto ninguna unidad USB conectada."
}
```

### 5.2 — Reporte

Redacta un reporte breve que incluya lo siguiente. Puede ser en Word o PDF.

| Seccion | Contenido requerido |
|---------|---------------------|
| **Portada** | Nombre completo, matricula, grupo y fecha |
| **Parte 1 — Sonidos** | Liga o nombre de los videos grabados. Responde: ¿en que parte del registro se guardo el cambio de sonido? ¿Que tipo de operacion de E/S fue? |
| **Parte 2 — Iconos** | Liga o nombre del video. Responde: ¿que archivo creo PowerShell para cambiar el icono? ¿Por que fue necesario reiniciar el Explorador? |
| **Parte 3 — Monitoreo** | Liga o nombre del video. Responde: ¿cuantos eventos se generaron al conectar el USB? ¿Cuales fueron los codigos de evento? |
| **Parte 4 — Codigo E/S** | Liga o nombre del video. Registra los tiempos de escritura y lectura bloqueante. Explica con tus palabras la diferencia entre `WriteAllText` y `WriteAsync`. |
| **Conclusion** | ¿Que relacion encontraste entre el sonido que escuchaste al conectar el USB y el concepto de E/S no bloqueante? Minimo 5 oraciones. |

---

## Criterios de Evaluacion

| Criterio | Puntos | Descripcion |
|----------|--------|-------------|
| Parte 1: Sonidos — video con sonido | 2.0 | Se escucha claramente el nuevo sonido al conectar/desconectar el USB. Los scripts se ejecutaron sin error. |
| Parte 2: Iconos — video con resultado visible | 1.5 | Se ve el icono cambiado en el Explorador de Windows. |
| Parte 3: Monitoreo — video con eventos capturados | 2.0 | Se ven los eventos de conexion/desconexion en tiempo real. Se ejecuto el listado de dispositivos. |
| Parte 4: Codigo E/S — video con tiempos y perfmon | 1.5 | Se ven los tiempos de ambos scripts y el Monitor de Rendimiento activo. |
| Reporte escrito con analisis y conclusion | 1.0 | Responde las preguntas de cada parte y conecta los conceptos con la conclusion. |
| **TOTAL** | **8.0** | |

---

## Entrega

Sube a **Teams** antes del **viernes 20 de febrero a las 23:59 hrs** un PDF del reporte y los videos grabados (pueden ser links de Google Drive o YouTube no listado). Nombra el PDF como:

```text
A17_ApellidoNombre_Matricula.pdf
```

## Referencias

- Carretero, J., Garcia-Carballeira, F., de Miguel, P., & Perez, J. M. (2001). *Sistemas Operativos: Una vision aplicada*. McGraw-Hill. Capitulo 7: Entrada/Salida.
- Silberschatz, A., Galvin, P. B., & Gagne, G. (2018). *Operating System Concepts* (10a ed.). Wiley.
- Tanenbaum, A. S. & Bos, H. (2015). *Modern Operating Systems* (4a ed.). Pearson. Capitulo 5: I/O.
- Microsoft Docs. (s.f.). *Windows Driver Kit documentation*. https://docs.microsoft.com/en-us/windows-hardware/drivers/
- Microsoft Docs. (s.f.). *Get-PnpDevice*. https://docs.microsoft.com/en-us/powershell/module/pnpdevice/get-pnpdevice
