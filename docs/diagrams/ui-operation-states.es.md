# Estados de operación de la UI

**[English](ui-operation-states.md) | [Español](ui-operation-states.es.md)**

Ciclo de vida compartido por las tres pestañas. `OperationCoordinator` reserva el turno de operación
antes del primer `await`, así el estado Running ignora un segundo clic y nunca corren dos
operaciones a la vez.

```mermaid
stateDiagram-v2
    [*] --> Empty
    Empty --> Ready: entrada válida seleccionada
    Ready --> Ready: cambia la entrada, ResetResult
    Ready --> Running: acción clickeada, Operations.Begin
    Running --> Running: segunda acción ignorada mientras está ocupado
    Running --> ConfirmCancel: clic en Cancelar
    ConfirmCancel --> Running: "No, continuar"
    ConfirmCancel --> Cancelled: "Sí, cancelar", token.Cancel
    Running --> Success: OperationResult.Ok
    Running --> Cancelled: ErrorCode.Cancelled, aviso neutral
    Running --> Failed: otro ErrorCode, localizado
    Success --> Ready: cambia la entrada
    Cancelled --> Ready: cambia la entrada
    Failed --> Ready: cambia la entrada
    Success --> [*]: componente descartado
    Cancelled --> [*]: componente descartado
    Failed --> [*]: componente descartado
```

Notas:

- Los estados visuales de éxito, cancelación y error son consistentes en las tres pestañas, y una
  falla siempre limpia la barra de progreso.
- La acción permanece deshabilitada hasta que la entrada es válida: un archivo de origen y un
  formato de destino distinto para las pestañas de audio y de extracción, y una URL bien formada más
  una carpeta de destino para la pestaña de descarga.
- La confirmación de cancelación en línea desaparece sola si la operación termina primero.
