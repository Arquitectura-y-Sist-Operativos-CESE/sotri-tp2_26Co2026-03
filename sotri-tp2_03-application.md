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


## Paso 03:

# Paso 03: Sincronización entre tareas usando Semáforo Binario

En este paso, modificamos el mecanismo de comunicación entre la `task_btn` y la `task_led` sustituyendo los esquemas previos por un **Semáforo Binario** de FreeRTOS.

## ¿Qué hicimos?
1. **Creación del Semáforo:** Inicializamos un semáforo binario global (`xLedBinarySemaphore`) que por defecto arranca en estado vacío (0 fichas disponibles).
2. **Liberación desde el Botón (Task BTN - Give):** Modificamos la máquina de estados del botón para que ejecute un `xSemaphoreGive()` tanto al **presionar** como al **soltar** el pulsador. Este envío se configuró de forma **no bloqueante** (timeout = 0) para garantizar que la tarea del botón continúe escaneando el hardware continuamente sin riesgo de colgarse si el semáforo ya vale 1.
3. **Recepción en el LED (Task LED - Take):** Implementamos un mecanismo de **bloqueo adaptativo** mediante la función `xSemaphoreTake()` según el estado del LED:
   * **LED Apagado (`ST_LED_OFF`):** Bloqueo indefinido (`portMAX_DELAY`). La tarea se suspende por completo y no consume CPU hasta que el botón libere la ficha.
   * **LED Parpadeando (`ST_LED_BLINK`):** Bloqueo con tiempo límite (*timeout*) igual al período de parpadeo (`DEL_LED_MAX`). Si el botón envía una señal (ej. se soltó), la tarea se despierta inmediatamente y cambia de estado; si no llega ninguna señal, se despierta por timeout, invierte el estado físico del pin del LED (*toggle*) y se vuelve a bloquear.

## ¿Qué observamos al depurar el código?
* **Filtrado de rebotes (Ráfagas):** Debido a que el semáforo binario solo puede almacenar un valor máximo de 1, si se producen rebotes eléctricos rápidos en el botón, los múltiples *Gives* consecutivos no saturan la memoria del sistema. El semáforo actúa como un filtro natural de eventos redundantes.
* **Señalización pura sin datos:** Notamos que a diferencia de una cola, el semáforo no transporta datos (no le dice a la tarea qué evento exacto ocurrió). Por lo tanto, la `task_led` resuelve qué acción tomar basándose estrictamente en su estado interno (si estaba apagada, pasa a parpadear; si estaba parpadeando, se apaga).