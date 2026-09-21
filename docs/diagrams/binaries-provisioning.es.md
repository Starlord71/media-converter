# Aprovisionamiento de binarios

**[English](binaries-provisioning.md) | [Español](binaries-provisioning.es.md)**

Flujo del primer arranque que prepara ffmpeg y yt-dlp en la carpeta de datos de la aplicación
(`%LOCALAPPDATA%\MediaConverter`). El progreso es real (basado en bytes), y una falla se mapea a un
código legible por máquina que la App localiza.

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant Main as Main.razor
    participant Language as LanguageService
    participant Core as IBinariesProvisioningService
    participant Downloader as BinaryDownloader
    participant Fs as Sistema de archivos

    User->>Main: Elige un idioma en el primer arranque
    Main->>Language: SetLanguage(code)
    Language->>Fs: escribe settings.json
    Main->>Core: ProvisionAsync(progress, token)

    loop por cada binario en BinaryCatalog
        Core->>Fs: verifica el binario y su tamaño
        alt ya utilizable
            Core->>Core: omite
        else falta
            Core->>Downloader: FetchLatestVersionAsync
            Core->>Downloader: DownloadAsync a target.part
            Downloader-->>Core: progreso mediante IProgress
            Core-->>Main: ProgressInfo(percent, Downloading)
            Core->>Fs: extrae el archivo o mueve el binario a su lugar
            Core-->>Main: ProgressInfo(100, Installing)
            Core->>Fs: escribe el marcador de versión
        end
    end

    alt éxito
        Core-->>Main: OperationResult.Ok
        Main->>Main: abre el espacio de trabajo con pestañas
    else sin internet o falla
        Core->>Core: ProvisioningExceptionMapper mapea la excepción
        Core-->>Main: Fail(NetworkError u otro código)
        Main->>Main: muestra el mensaje localizado y un botón Reintentar
    end
```

En corridas posteriores el idioma se lee de `settings.json` y el aprovisionamiento arranca de
inmediato; si los binarios ya están presentes y son válidos, se omiten.
