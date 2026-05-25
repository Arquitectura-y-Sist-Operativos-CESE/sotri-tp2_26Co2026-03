# Preguntas y Respuestas sobre Colas (Queues) en FreeRTOS

## 1. ¿Cómo crear una Cola?
Para crear una cola se utiliza la función **`xQueueCreate()`**. 
Debes especificar dos parámetros:
* **La longitud:** El número máximo de elementos que la cola puede contener.
* **El tamaño del elemento:** La cantidad de bytes que ocupa cada dato (generalmente se usa `sizeof(tipo_de_dato)`).
La función devuelve un manejador de tipo `QueueHandle_t` que usarás para referenciar esta cola en el futuro. Si devuelve `NULL`, significa que no hubo suficiente memoria RAM (Heap) para crearla.

## 2. ¿Cómo eliminar una Cola?
Se utiliza la función **`vQueueDelete()`**. 
Le pasas como parámetro el manejador (`QueueHandle_t`) de la cola que deseas destruir. Al hacerlo, FreeRTOS libera toda la memoria dinámica que había asignado para esa estructura y sus datos.

## 3. ¿Cómo gestiona una Cola los datos que contiene?
Por defecto, una cola gestiona los datos bajo el modelo **FIFO** (*First In, First Out* - Primero en entrar, primero en salir). Sin embargo, también cuenta con funciones para comportarse como **LIFO** (*Last In, First Out*).
Un aspecto clave de FreeRTOS es que la gestión se hace **por copia** (Copy by Value), no por referencia. Esto significa que cuando envías una variable a la cola, FreeRTOS hace una copia exacta de sus bytes y los guarda en la memoria interna de la cola. Si los datos son muy grandes (como un buffer enorme), es una mejor práctica enviar *el puntero* (la dirección de memoria) del dato en lugar del dato completo para no saturar la memoria y perder tiempo de CPU.

## 4. ¿Cómo enviar datos a una Cola?
Para enviar datos desde una Tarea, se utilizan principalmente:
* **`xQueueSend()`** o **`xQueueSendToBack()`**: Coloca el dato al final de la cola (FIFO).
* **`xQueueSendToFront()`**: Coloca el dato al principio de la cola (LIFO).
Se le pasa el manejador de la cola, un puntero al dato a enviar, y el **Tiempo de Bloqueo** (cuántos *ticks* esperar si la cola está llena). 
*Nota: Si necesitas enviar datos desde una Interrupción (ISR), debes usar sus equivalentes terminadas en `FromISR` (ej. `xQueueSendFromISR()`).*

## 5. ¿Cómo recibir datos de una Cola?
Se utiliza la función **`xQueueReceive()`**.
Sus parámetros son: el manejador de la cola, un puntero a un buffer o variable donde se copiará el dato recibido, y el **Tiempo de Bloqueo** (cuántos *ticks* esperar si la cola está vacía). Al recibir el dato, este es eliminado de la cola. Si solo quieres leer el dato sin sacarlo de la cola, puedes usar **`xQueuePeek()`**.
*Desde una interrupción se utiliza `xQueueReceiveFromISR()`.*

## 6. ¿Qué significa bloquearse en una Cola?
"Bloquearse" significa que una tarea entra en estado **Blocked** (suspendida temporalmente, sin consumir tiempo de procesamiento del CPU) mientras espera que ocurra un evento en la cola. Esto puede pasar de dos formas:
* **Al leer:** Si una tarea intenta recibir un dato de una cola vacía, se bloquea a la espera de que otra tarea o interrupción escriba algo en ella.
* **Al escribir:** Si una tarea intenta enviar un dato a una cola que está llena, se bloquea a la espera de que alguien lea un dato y libere espacio.
Este bloqueo durará hasta que el evento suceda o hasta que se agote el tiempo límite (*Timeout*) que le pasaste como parámetro a la función.

## 7. ¿Cómo bloquearse en varias Colas?
Si una tarea necesita esperar datos de más de una cola al mismo tiempo (por ejemplo, esperar un mensaje de la UART y un evento de un botón), se utilizan los **Queue Sets** (Conjuntos de Colas).
El proceso es:
1.  Creas el conjunto con `xQueueCreateSet()`.
2.  Agregas las colas deseadas al conjunto con `xQueueAddToSet()`.
3.  La tarea llama a **`xQueueSelectFromSet()`**, la cual bloquea a la tarea hasta que *cualquiera* de las colas del conjunto reciba un dato.

## 8. ¿Cómo sobrescribir datos en una Cola?
Se utiliza la API **`xQueueOverwrite()`** (o `xQueueOverwriteFromISR()` en interrupciones).
Esta función está diseñada **exclusivamente para colas con una longitud máxima de 1** (es decir, colas que guardan un único valor, habitualmente llamadas buzones o *mailboxes*). Esta función escribe el nuevo dato en la cola, y si esta ya estaba llena con un valor anterior, simplemente lo sobrescribe sin importar, y nunca bloquea a la tarea.

## 9. ¿Cómo vaciar una Cola?
Para descartar todos los elementos contenidos actualmente en una cola, se usa **`xQueueReset()`**. 
Esta función devuelve la cola a su estado inicial, completamente vacía.

## 10. ¿Cuál es el efecto de las prioridades de las Tareas al escribir y leer en una Cola?
FreeRTOS es un sistema preemptivo basado en prioridades, y las colas respetan esta filosofía a la perfección:
* **Desbloqueo por Prioridad:** Si hay *varias* tareas bloqueadas esperando para leer de una misma cola vacía, y de repente llega un dato, FreeRTOS despertará (desbloqueará) **únicamente a la tarea con mayor prioridad**.
* **Desbloqueo por Antigüedad:** Si varias tareas bloqueadas esperando el dato tienen **la misma** prioridad, FreeRTOS despertará a la que lleve más tiempo esperando.
* (Lo mismo aplica a la inversa: si varias tareas están bloqueadas intentando escribir en una cola llena, la que tenga mayor prioridad escribirá primero cuando se libere un espacio).


# Paso 03: Comunicación entre tareas usando Colas

En este paso reemplazamos el mecanismo de banderas (polling) que veníamos usando por una **Cola (Queue)** de FreeRTOS para comunicar la `task_btn` con la `task_led`. 

## ¿Qué hicimos?
1. **Creación de la cola:** Creamos una cola global (`xLedEventQueue`) con capacidad para guardar hasta 5 eventos.
2. **Envío desde el Botón (Task BTN):** En la máquina de estados del botón, reemplazamos la función que levantaba la bandera por un `xQueueSend`. A esto lo configuramos de forma **no bloqueante** (tiempo de espera = 0). Lo hicimos así para que la tarea del botón nunca se quede trabada y pueda seguir leyendo el pin físico continuamente, aunque la cola estuviera llena.
3. **Recepción en el LED (Task LED):** Acá metimos el cambio más grande. Pasamos a usar `xQueueReceive` con un sistema de **bloqueo adaptativo**:
    * Si el LED está apagado (`ST_LED_OFF`), bloqueamos la tarea por tiempo infinito (`portMAX_DELAY`). La tarea se "duerme" por completo y no gasta CPU hasta que el botón envía algo.
    * Si el LED está parpadeando (`ST_LED_BLINK`), la bloqueamos con un timeout igual al tiempo del parpadeo (`DEL_LED_MAX`). De esta forma, si no llega la orden de apagar, la tarea se despierta por timeout, hace el toggle (cambio de estado) del LED, y vuelve a esperar.

## ¿Qué observamos al depurar el código?
* **Ahorro brutal de CPU:** Al monitorear la variable `g_task_idle_cnt`, vimos que se incrementó un montón en comparación a la versión anterior. Como ahora las tareas se bloquean de verdad en vez de dar vueltas en un *while* preguntando por una bandera, el microcontrolador pasa mucho más tiempo en reposo (en la tarea Idle).
* **Cero pérdida de eventos:** Al usar una cola, los eventos se guardan en orden (FIFO). Ya no pasa que un evento sobrescriba a otro si el usuario presiona el botón súper rápido.
* **Respuesta en tiempo real:** En cuanto apretamos el botón, el sistema operativo despierta a la tarea del LED inmediatamente, eliminando cualquier tipo de lag en la respuesta.