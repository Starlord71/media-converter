# Descarga de video

**[English](video-download.md) | [Español](video-download.es.md)**

Secuencia de la pestaña de Video, desde ingresar una URL hasta terminar una descarga. yt-dlp trabaja
dentro de un directorio aislado para que una corrida cancelada nunca deje archivos parciales en la
carpeta del usuario.

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant Tab as VideoDownloader.razor
    participant Dialogs as FileDialogService
    participant Coordinator as OperationCoordinator
    participant Core as IVideoDownloaderService
    participant YtDlp as yt-dlp
    participant Ffmpeg as ffmpeg

    User->>Tab: Pega una URL y elige una carpeta de destino
    Tab->>Tab: UrlSupport.IsWellFormedVideoUrl habilita Download
    Tab->>Dialogs: PickFolderAsync(title)
    Dialogs-->>Tab: carpeta de destino

    User->>Tab: Clic en "Descargar"
    Tab->>Coordinator: Begin (ocupado)
    Tab->>Core: DownloadAsync(url, folder, format, progress, token)
    Core->>Core: valida la URL, crea la carpeta y un directorio de trabajo aislado
    Core->>YtDlp: Process.Start(--newline --progress, --ffmpeg-location)
    loop mientras yt-dlp corre
        YtDlp-->>Core: líneas de progreso
        Core-->>Tab: ProgressInfo(percent, Downloading)
    end
    alt éxito
        Core->>Core: localiza el archivo final y lo mueve a la carpeta
        Core->>Core: borra el directorio de trabajo
        Core-->>Tab: OperationResult.Ok
    else error de yt-dlp
        Core->>Core: YtDlpErrorClassifier clasifica el stderr
        Core-->>Tab: Fail(código mapeado, por ejemplo NetworkError)
    else el usuario cancela
        Tab->>Core: token.Cancel()
        Core->>YtDlp: Kill(entireProcessTree)
        Core->>Core: borra el directorio de trabajo
        Core-->>Tab: Fail(Cancelled)
    end
    Tab->>Coordinator: Dispose
```

El `DownloadFormat` solicitado (MP4, MP3 o M4A) se pasa a yt-dlp a través de `YtDlpArgumentBuilder`,
que elige el mejor stream y delega la extracción o el muxing a ffmpeg.
