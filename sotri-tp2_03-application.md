# Resumen: Semáforos Binarios vs. Semáforos Contadores en FreeRTOS

Ambos tipos de semáforos se usan para la sincronización entre tareas o entre interrupciones (ISR) y tareas. Comparten el mismo tipo de variable (`SemaphoreHandle_t`) y se manejan con las mismas funciones básicas:
* **Dar (`xSemaphoreGive`):** Emite la señal (incrementa el semáforo).
* **Tomar (`xSemaphoreTake`):** Consume la señal (decrementa el semáforo). Si está en 0, la tarea se bloquea esperando.

## 1. Creación y Uso
* **Semáforo Binario:** Nace en **0** (vacío). Solo puede valer 0 o 1.
  ```c
  SemaphoreHandle_t xSemBinario = xSemaphoreCreateBinary();
  ```
* **Semáforo Contador:** Nace con el valor inicial que le pasemos y puede llegar hasta un tope definido.
  ```c
  // xSemaphoreCreateCounting(Maximo, Inicial);
  SemaphoreHandle_t xSemContador = xSemaphoreCreateCounting(10, 0);
  ```

## 2. Diferencias Clave (Cuándo usar cuál)

| Característica | Semáforo Binario | Semáforo Contador |
| :--- | :--- | :--- |
| **Límite** | Máximo 1. | Mayor a 1 (lo definimos nosotros). |
| **Sincronización** | Ideal para avisos simples (ej. "Terminó el ADC, leé el dato"). | Ideal para hardware que genera interrupciones muy rápido. |
| **Manejo de Recursos** | No aplica. | Útil para gestionar un pool de recursos (ej. "Tengo 5 buffers de memoria disponibles"). |
| **Pérdida de eventos en ráfagas** | **Sí.** Si ocurren 3 eventos rápidos y la tarea aún no leyó el primero, el semáforo se queda trabado en 1. Se pierden los otros 2. | **No.** Si ocurren 3 eventos rápidos, el contador sube a 3. La tarea se despertará 3 veces seguidas procesando todo sin perder nada. |