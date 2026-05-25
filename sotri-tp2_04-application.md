# Manejo de Interrupciones en FreeRTOS

A continuación, se detallan las respuestas a tus consultas sobre la gestión y el comportamiento de las Rutinas de Servicio de Interrupción (ISR) dentro del ecosistema de FreeRTOS.

## 1. ¿Qué funciones de la API de FreeRTOS se pueden usar dentro de una rutina de servicio de interrupción?

En FreeRTOS, **nunca** debes llamar a una función de la API estándar dentro de una ISR (por ejemplo, no puedes usar `xQueueSend` o `xSemaphoreTake`). Hacerlo corromperá el estado del sistema operativo.

En su lugar, FreeRTOS proporciona un conjunto de funciones dedicadas diseñadas específicamente para ser seguras dentro de interrupciones. Estas funciones siempre terminan con el sufijo **`FromISR`**. 
Algunos ejemplos comunes incluyen:
* `xQueueSendFromISR()`
* `xSemaphoreGiveFromISR()`
* `vTaskNotifyGiveFromISR()`
* `xEventGroupSetBitsFromISR()`

**Características clave de las funciones `FromISR`:**
1.  **No bloquean:** Una ISR no es una tarea; no puede entrar en estado *Blocked*. Por lo tanto, si una cola está llena, `xQueueSendFromISR()` simplemente retorna un error inmediatamente en lugar de esperar.
2.  **Cambio de contexto diferido:** Estas funciones suelen aceptar un puntero a una variable `BaseType_t` (generalmente llamada `pxHigherPriorityTaskWoken`). Si al ejecutar la función se despierta una tarea con mayor prioridad que la tarea actualmente interrumpida, esta variable se pone en `pdTRUE`. Al final de la ISR, el desarrollador debe forzar un cambio de contexto (usando un macro como `portYIELD_FROM_ISR()`) para que al salir de la interrupción, el CPU salte directamente a la tarea más prioritaria.

---

## 2. Métodos para delegar el procesamiento de interrupciones a una Tarea

El concepto de delegar trabajo fuera de la ISR se conoce como **Procesamiento de Interrupción Diferido (Deferred Interrupt Processing)**. La regla de oro en sistemas embebidos es mantener las ISR lo más cortas y rápidas posible, delegando el procesamiento pesado a una Tarea de FreeRTOS.

Los métodos principales son:
* **Semáforos Binarios:** La ISR simplemente hace un `Give` y la tarea está bloqueada haciendo un `Take`. Es el método clásico.
* **Notificaciones de Tarea (Task Notifications):** Es el método **más recomendado y moderno**. La ISR usa `vTaskNotifyGiveFromISR()` para señalar a una tarea específica. Es un 45% más rápido y consume menos memoria que un semáforo, ya que no requiere crear un objeto de sincronización independiente (va directo al TCB de la tarea).
* **Colas de Datos (Queues):** Útil si la ISR necesita delegar el trabajo *y* pasar información al mismo tiempo (ej. enviar el dato crudo del ADC a la tarea para que esta le aplique filtros matemáticos).
* **Llamadas a funciones diferidas (Timer Daemon Task):** Usando `xTimerPendFunctionCallFromISR()`, puedes encolar la ejecución de una función para que sea procesada por la tarea demonio de los *Software Timers* de FreeRTOS.

---

## 3. ¿Cómo usar una cola para transferir datos dentro y fuera de una ISR?

Para usar una cola interactuando con una ISR, se utilizan las versiones de la API que terminan en `FromISR`.

* **De la ISR a una Tarea (Lo más común):**
    Si tienes un periférico como la UART que recibe bytes, cada vez que llega un byte, salta la ISR. Dentro de la ISR, lees el byte del registro de hardware y lo envías a la cola usando `xQueueSendFromISR()`. 
    La tarea receptora, que estaba bloqueada con un `xQueueReceive()` estándar, se despierta, toma el byte y lo procesa.

* **De una Tarea a la ISR (Menos común, pero posible):**
    Si una tarea necesita enviar un bloque de datos por un periférico, puede meter los datos en una cola. Luego, habilita la interrupción de "buffer de transmisión vacío" del periférico. Dentro de la ISR, se utiliza `xQueueReceiveFromISR()` para sacar el siguiente dato de la cola y ponerlo en el registro de transmisión del hardware.

*Nota:* Si vas a transferir ráfagas de datos muy grandes o muy rápidas, usar una cola elemento por elemento desde una ISR es ineficiente por la sobrecarga del RTOS. En esos casos, es mejor usar DMA (Acceso Directo a Memoria) y que el DMA lance una interrupción solo cuando haya terminado de mover todo el bloque.

---

## 4. ¿Cuál es el modelo de anidamiento de interrupciones disponible en algunas portaciones de FreeRTOS?

En arquitecturas que soportan prioridades de hardware (como los ARM Cortex-M de los STM32), FreeRTOS implementa un modelo de **Anidamiento Controlado (Controlled Interrupt Nesting)**.

Esto significa que una interrupción de mayor prioridad puede interrumpir a una interrupción de menor prioridad que se esté ejecutando en ese momento. FreeRTOS gestiona esto dividiendo las prioridades de interrupción en dos grupos principales, configurables en `FreeRTOSConfig.h`:

1.  **Por debajo de `configMAX_SYSCALL_INTERRUPT_PRIORITY` (Interrupciones del RTOS):**
    * Las interrupciones en este nivel **pueden** hacer uso de las funciones de la API de FreeRTOS (`...FromISR`).
    * FreeRTOS tiene permiso para deshabilitar (enmascarar) temporalmente estas interrupciones cuando necesita ejecutar secciones críticas del código del kernel (por ejemplo, al insertar un elemento en una lista).

2.  **Por encima de `configMAX_SYSCALL_INTERRUPT_PRIORITY` (Interrupciones Críticas):**
    * Estas interrupciones son de tan alta prioridad que el RTOS **nunca** las deshabilita. Su latencia de respuesta está dictada puramente por el hardware (típicamente unos pocos ciclos de reloj).
    * **La restricción:** Debido a que el RTOS no las protege ni las conoce, **NUNCA** debes llamar a ninguna función de la API de FreeRTOS desde estas interrupciones. Son exclusivas para tareas de hardware ultra-críticas (ej. control de inversores de motores o apagado de emergencia).


# Paso 03

# Informe de Observación: Sincronización de ISR y Tareas mediante Semáforo Binario

Este documento detalla la estructura y el comportamiento dinámico del sistema tras la implementación del mecanismo de sincronización entre el hardware (interrupción) y el software (RTOS).

---

## 1. Mecanismo Utilizado
Se implementó un **Semáforo Binario** de FreeRTOS para establecer un canal de comunicación directo y eficiente entre la **Rutina de Servicio de Interrupción (ISR)** del botón externo y la tarea encargada de procesarlo (`task_btn`).

---

## 2. Comportamiento Observado durante la Depuración

Al monitorear las variables del sistema y los estados de ejecución en la herramienta de depuración, se registraron las siguientes métricas y comportamientos:

* **Estado de la tarea:** Mientras el botón físico no es presionado, la tarea `task_btn` permanece estrictamente en estado **Blocked** (Bloqueada). Esto se logra de manera determinista mediante el uso del parámetro `portMAX_DELAY` en la llamada a la función `xSemaphoreTake()`.
* **Uso de CPU:** Al encontrarse la tarea en estado bloqueado, `task_btn` es removida de la lista de tareas listas para ejecutar (*Ready List*). De esta forma, **no consume ciclos de procesamiento**, permitiendo que la tarea inactiva (`Idle Task`) u otras tareas de menor prioridad tomen el control del CPU. Esto elimina por completo el desperdicio crítico de recursos de procesamiento asociado al método de consulta continua (*polling*) implementado en las prácticas previas.
* **Latencia y Respuesta:** En el instante en que se presiona el botón físico, el periférico de hardware (EXTI) dispara la interrupción de manera inmediata. La ISR ejecuta la función segura `xSemaphoreGiveFromISR()`, la cual es no bloqueante por diseño. Inmediatamente después, el uso de la macro `portYIELD_FROM_ISR()` evalúa el estado del programador: si la prioridad de `task_btn` es mayor que la de la tarea que estaba corriendo, el sistema operativo fuerza un **cambio de contexto inmediato**, logrando que el microcontrolador responda de manera prácticamente instantánea al evento físico.
* **Sincronización:** Se comprobó experimentalmente que el semáforo binario actúa puramente como una **bandera de sincronización de eventos asincrónicos** entre el hardware y el software, actuando como un puente ideal que minimiza el *jitter* y la latencia.