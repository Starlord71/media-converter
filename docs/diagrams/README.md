# Diagrams

**[English](README.md) | [Español](README.es.md)**

Detailed Mermaid diagrams for MediaConverter, in English and Spanish. GitHub renders Mermaid inside
Markdown, so each file below can be read directly.

| Diagram | What it shows |
| --- | --- |
| [Architecture](architecture.md) | Layered view of the WPF host, the Razor components, the App services, Core and the external tools, plus the Core class diagram (interfaces, models, services and helper types). |
| [Audio conversion](audio-conversion.md) | Sequence from picking a file to a finished audio conversion, including cancellation, plus the video-to-audio extraction that reuses the same pipeline. |
| [Video download](video-download.md) | Sequence from entering a URL to a finished download, including the isolated work directory. |
| [Binaries provisioning](binaries-provisioning.md) | First-run download and installation of ffmpeg and yt-dlp. |
| [UI operation states](ui-operation-states.md) | Tab lifecycle: empty, ready, running, success, cancelled and failed. |

The screenshots and the demo GIF live in [`../images`](../images).
