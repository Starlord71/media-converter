# Arquitectura

**[English](architecture.md) | [Español](architecture.es.md)**

Vista por capas de la solución. La dirección de la dependencia es estrictamente en un solo sentido:
la App depende de Core, y Core no depende de nada (sin WPF, sin Blazor, sin tipos de UI).

```mermaid
flowchart TB
    subgraph App["MediaConverter.App - net9.0-windows (host WPF)"]
        direction TB
        Window["MainWindow.xaml<br/>BlazorWebView, HostPage wwwroot/index.html"]
        subgraph Components["Componentes Razor"]
            Main["Main.razor<br/>barra de idioma, barra de pestañas, pantallas de configuración"]
            AudioTab["AudioConverter.razor"]
            VideoTab["VideoDownloader.razor"]
            VideoToAudioTab["VideoToAudio.razor"]
        end
        subgraph AppServices["Servicios de la App (inyección de dependencias)"]
            Language["LanguageService<br/>aplica CultureInfo, persiste settings.json"]
            Dialogs["FileDialogService (scoped)<br/>envuelve IJSRuntime y los diálogos de WPF"]
            Coordinator["OperationCoordinator<br/>una operación a la vez, flag de ocupado"]
            Validators["UrlSupport, AudioFileSupport,<br/>VideoFileSupport"]
            Localizer["LocalizerExtensions<br/>mapea ErrorCode y ProgressStage a texto"]
        end
        Resources["Resources.resx (inglés) + Resources.es.resx (español)"]
        Assets["wwwroot: index.html, css/app.css, js/fileDialog.js"]
    end

    subgraph Core["MediaConverter.Core - net9.0 (biblioteca de clases, sin UI)"]
        direction TB
        Interfaces["Interfaces<br/>IAudioConverterService, IVideoDownloaderService,<br/>IBinariesProvisioningService"]
        Services["Servicios<br/>AudioConverterService, VideoDownloaderService,<br/>BinariesProvisioningService"]
        Parsers["Parsers y clasificadores<br/>FfmpegProgressParser, YtDlpProgressParser,<br/>YtDlpArgumentBuilder, YtDlpErrorClassifier,<br/>DownloadArtifactLocator"]
        BinarySub["Subsistema de binarios<br/>BinaryCatalog, BinaryDownloader, BinaryPaths,<br/>BinarySources, GitHubReleaseParser,<br/>ProvisioningExceptionMapper, VersionComparer"]
        Models["Modelos<br/>OperationResult, ProgressInfo, ErrorCode,<br/>AudioFormat, DownloadFormat, ProgressStage"]
    end

    Window --> Components
    Main --> AudioTab
    Main --> VideoTab
    Main --> VideoToAudioTab
    AudioTab --> AppServices
    VideoTab --> AppServices
    VideoToAudioTab --> AppServices
    Localizer --> Resources
    Components -.-> Assets
    Dialogs <-->|"interop de JavaScript"| Assets

    App -->|"interfaces, métodos async,<br/>IProgress y CancellationToken"| Core
    Services --> Parsers
    Services --> BinarySub
    Interfaces --> Models
    Services --> Models

    subgraph External["Herramientas externas y archivos (carpeta de datos de la aplicación)"]
        FFmpeg["ffmpeg.exe"]
        YtDlp["yt-dlp.exe"]
        Settings["settings.json (preferencia de idioma)"]
    end
    Services --> FFmpeg
    Services --> YtDlp
    BinarySub --> FFmpeg
    BinarySub --> YtDlp
    Language --> Settings
```

La regla de oro: Core nunca sabe que existe una UI. Expone interfaces, métodos async, progreso a
través de `IProgress<ProgressInfo>` y cancelación a través de `CancellationToken`, y reporta fallas
como valores `ErrorCode` legibles por máquina. La App traduce esos códigos al idioma activo, así
Core se mantiene libre de textos orientados al usuario.

## Componentes de Core

Vista de clases de MediaConverter.Core. Core no referencia tipos de WPF ni de Blazor, por lo que
puede probarse de forma aislada y reutilizarse desde cualquier host.

```mermaid
classDiagram
    direction TB

    class IAudioConverterService {
        <<interface>>
        +ConvertAsync(sourcePath, outputPath, targetFormat, progress, cancellationToken) OperationResult
    }
    class IVideoDownloaderService {
        <<interface>>
        +DownloadAsync(url, outputDirectory, format, progress, cancellationToken) OperationResult
    }
    class IBinariesProvisioningService {
        <<interface>>
        +ProvisionAsync(progress, cancellationToken) OperationResult
        +UpdateAsync(progress, cancellationToken) OperationResult
    }

    class AudioConverterService
    class VideoDownloaderService
    class BinariesProvisioningService

    IAudioConverterService <|.. AudioConverterService
    IVideoDownloaderService <|.. VideoDownloaderService
    IBinariesProvisioningService <|.. BinariesProvisioningService

    class OperationResult {
        +bool Succeeded
        +ErrorCode ErrorCode
        +Ok() OperationResult
        +Fail(code) OperationResult
    }
    class ProgressInfo {
        +double Percentage
        +ProgressStage Stage
        +string Detail
    }
    class ErrorCode {
        <<enumeration>>
        Unknown
        Cancelled
        InvalidInput
        FileNotFound
        OutputPathInvalid
        UnsupportedFormat
        InvalidUrl
        NetworkError
        AccessDenied
        InsufficientDiskSpace
        BinaryNotFound
        BinaryDownloadFailed
        ProvisioningFailed
        ConversionFailed
        DownloadFailed
    }
    class AudioFormat {
        <<enumeration>>
        M4A
        MP3
    }
    class DownloadFormat {
        <<enumeration>>
        MP4
        MP3
        M4A
    }
    class ProgressStage {
        <<enumeration>>
        Starting
        Analyzing
        Downloading
        Converting
        Installing
        Finalizing
        Completed
    }

    AudioConverterService ..> OperationResult
    AudioConverterService ..> ProgressInfo
    VideoDownloaderService ..> OperationResult
    VideoDownloaderService ..> ProgressInfo
    BinariesProvisioningService ..> OperationResult
    BinariesProvisioningService ..> ProgressInfo
    OperationResult ..> ErrorCode
    ProgressInfo ..> ProgressStage
```

### Tipos auxiliares

- Pipeline de audio: `FfmpegProgressParser` interpreta la salida de ffmpeg `-progress pipe:1` como
  un porcentaje.
- Pipeline de descarga: `YtDlpArgumentBuilder` construye los argumentos de yt-dlp,
  `YtDlpProgressParser` interpreta sus líneas de progreso, `YtDlpErrorClassifier` mapea el stderr a
  un `ErrorCode`, y `DownloadArtifactLocator` encuentra el único archivo que produjo yt-dlp.
- Subsistema de binarios: `BinaryCatalog` y `BinaryDefinition` describen qué descargar,
  `BinaryDownloader` realiza el trabajo HTTP, `BinarySources` y `GitHubReleaseParser` resuelven las
  versiones, `BinaryPaths` resuelve dónde viven los binarios, `ProvisioningExceptionMapper` mapea
  excepciones a `ErrorCode`, y `VersionComparer` decide cuándo hace falta una actualización.
