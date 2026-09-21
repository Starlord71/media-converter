# Conversión de audio

**[English](audio-conversion.md) | [Español](audio-conversion.es.md)**

Secuencia de la pestaña de Audio, desde elegir un archivo hasta terminar una conversión de M4A a MP3
(o de MP3 a M4A).

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant Tab as AudioConverter.razor
    participant Dialogs as FileDialogService
    participant Wpf as WPF OpenFileDialog
    participant Coordinator as OperationCoordinator
    participant Core as IAudioConverterService
    participant Ffmpeg as ffmpeg

    User->>Tab: Clic en "Elegir archivo..."
    Tab->>Dialogs: PickFileAsync(title, filter)
    Dialogs->>Wpf: fileDialog.js, luego dispatcher.InvokeAsync
    Wpf-->>Dialogs: ruta seleccionada o null
    Dialogs-->>Tab: ruta
    alt cancelado o extensión no soportada
        Tab->>Tab: mantiene la selección, o muestra UnsupportedFormat
    else soportado
        Tab->>Tab: TryDetectFormat, destino por defecto, BuildTargetPath (sin sobrescribir)
    end

    User->>Tab: Clic en "Convertir"
    Tab->>Coordinator: Begin (ocupado, controles deshabilitados)
    Tab->>Core: ConvertAsync(source, output, target, progress, token)
    Core->>Core: valida entradas, resuelve el directorio de salida, verifica ffmpeg
    Core->>Ffmpeg: Process.Start(-progress pipe:1)
    loop mientras ffmpeg corre
        Ffmpeg-->>Core: progreso key=value y stderr
        Core-->>Tab: ProgressInfo(percent, Converting)
    end
    alt éxito
        Core-->>Tab: OperationResult.Ok
        Tab->>Tab: progreso 100 Completed, muestra la ruta de salida
    else el usuario cancela
        Tab->>Core: token.Cancel()
        Core->>Ffmpeg: Kill(entireProcessTree)
        Core-->>Tab: Fail(Cancelled)
        Tab->>Tab: aviso neutral, limpia el progreso
    else ffmpeg falla
        Core-->>Tab: Fail(ConversionFailed)
        Tab->>Tab: error localizado, limpia el progreso
    end
    Tab->>Coordinator: Dispose (libera el flag de ocupado)
```

## Video a audio

La pestaña Video a audio reutiliza exactamente el mismo pipeline de `IAudioConverterService`: ffmpeg
descarta el flujo de video con `-vn`, por lo que un origen MP4 sigue el mismo camino de código que un
archivo de audio; solo cambian el selector de archivo y el destino por defecto.

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant Tab as VideoToAudio.razor
    participant Dialogs as FileDialogService
    participant Core as IAudioConverterService
    participant Ffmpeg as ffmpeg

    User->>Tab: Clic en "Elegir archivo..."
    Tab->>Dialogs: PickFileAsync(title, filter)
    Dialogs-->>Tab: ruta seleccionada
    alt no es un MP4
        Tab->>Tab: muestra UnsupportedFormat, limpia la selección
    else es MP4
        Tab->>Tab: BuildTargetPath(path, targetFormat) junto al origen
        Note over Tab: targetFormat es MP3 por defecto y puede cambiarse a M4A
    end

    User->>Tab: Clic en "Extraer audio"
    Tab->>Core: ConvertAsync(source, output, targetFormat, progress, token)
    Note over Core,Ffmpeg: mismo pipeline de conversión, ffmpeg descarta el video con -vn
    Core->>Ffmpeg: Process.Start(-vn -progress pipe:1)
    Ffmpeg-->>Core: progreso
    Core-->>Tab: ProgressInfo(percent, Converting)
    Core-->>Tab: OperationResult.Ok o Fail(code)
```
