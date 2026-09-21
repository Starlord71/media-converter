# Diagramas

**[English](README.md) | [Español](README.es.md)**

Diagramas Mermaid detallados de MediaConverter, en inglés y español. GitHub renderiza Mermaid dentro
de Markdown, así que cada archivo de abajo puede leerse directamente.

| Diagrama | Qué muestra |
| --- | --- |
| [Arquitectura](architecture.es.md) | Vista por capas del host WPF, los componentes Razor, los servicios de la App, Core y las herramientas externas, más el diagrama de clases de Core (interfaces, modelos, servicios y tipos auxiliares). |
| [Conversión de audio](audio-conversion.es.md) | Secuencia desde elegir un archivo hasta terminar una conversión de audio, incluida la cancelación, más la extracción de video a audio que reutiliza el mismo pipeline. |
| [Descarga de video](video-download.es.md) | Secuencia desde ingresar una URL hasta terminar una descarga, incluido el directorio de trabajo aislado. |
| [Aprovisionamiento de binarios](binaries-provisioning.es.md) | Descarga e instalación en el primer arranque de ffmpeg y yt-dlp. |
| [Estados de operación de la UI](ui-operation-states.es.md) | Ciclo de vida de las pestañas: vacío, listo, corriendo, éxito, cancelado y fallido. |

Las capturas de pantalla y el GIF de demo viven en [`../images`](../images).
