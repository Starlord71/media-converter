# Audio conversion

**[English](audio-conversion.md) | [Español](audio-conversion.es.md)**

Sequence for the Audio tab, from picking a file to a finished M4A to MP3 (or MP3 to M4A) conversion.

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

    User->>Tab: Click "Choose file..."
    Tab->>Dialogs: PickFileAsync(title, filter)
    Dialogs->>Wpf: fileDialog.js, then dispatcher.InvokeAsync
    Wpf-->>Dialogs: selected path or null
    Dialogs-->>Tab: path
    alt cancelled or unsupported extension
        Tab->>Tab: keep selection, or show UnsupportedFormat
    else supported
        Tab->>Tab: TryDetectFormat, default target, BuildTargetPath (non-overwriting)
    end

    User->>Tab: Click "Convert"
    Tab->>Coordinator: Begin (busy, controls disabled)
    Tab->>Core: ConvertAsync(source, output, target, progress, token)
    Core->>Core: validate inputs, resolve output directory, check ffmpeg
    Core->>Ffmpeg: Process.Start(-progress pipe:1)
    loop while ffmpeg runs
        Ffmpeg-->>Core: progress key=value and stderr
        Core-->>Tab: ProgressInfo(percent, Converting)
    end
    alt success
        Core-->>Tab: OperationResult.Ok
        Tab->>Tab: progress 100 Completed, show the output path
    else user cancels
        Tab->>Core: token.Cancel()
        Core->>Ffmpeg: Kill(entireProcessTree)
        Core-->>Tab: Fail(Cancelled)
        Tab->>Tab: neutral notice, clear progress
    else ffmpeg fails
        Core-->>Tab: Fail(ConversionFailed)
        Tab->>Tab: localized error, clear progress
    end
    Tab->>Coordinator: Dispose (release busy)
```

## Video to audio

The Video to audio tab reuses the exact same `IAudioConverterService` pipeline: ffmpeg discards the
video stream with `-vn`, so an MP4 source follows the identical code path as an audio file, only the
picker and the default target differ.

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant Tab as VideoToAudio.razor
    participant Dialogs as FileDialogService
    participant Core as IAudioConverterService
    participant Ffmpeg as ffmpeg

    User->>Tab: Click "Choose file..."
    Tab->>Dialogs: PickFileAsync(title, filter)
    Dialogs-->>Tab: selected path
    alt not an MP4
        Tab->>Tab: show UnsupportedFormat, clear the selection
    else MP4
        Tab->>Tab: BuildTargetPath(path, targetFormat) next to the source
        Note over Tab: targetFormat defaults to MP3 and can be switched to M4A
    end

    User->>Tab: Click "Extract audio"
    Tab->>Core: ConvertAsync(source, output, targetFormat, progress, token)
    Note over Core,Ffmpeg: same conversion pipeline, ffmpeg drops the video with -vn
    Core->>Ffmpeg: Process.Start(-vn -progress pipe:1)
    Ffmpeg-->>Core: progress
    Core-->>Tab: ProgressInfo(percent, Converting)
    Core-->>Tab: OperationResult.Ok or Fail(code)
```
