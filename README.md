# Práctica3: Controlador Máquina Expendedora

El objetivo de esta Practica3: "Controlador Máquina Expendedora" para la asignatura de Sistema empotrados y de tiempo real ha sido diseñar e implementar un controlador para una máquina expendedora basada
en Arduino UNO, integrando varios sensores y actuadores reales del kit de prácticas. 

Para esta práctica hemos usado los siguientes componentes hardware: 
- Arduino UNO
- LCD
- Joystick
- Sensor Temperatura/Humedad DHT11
- Sensor Ultrasonidos
- Botón
- 2 LEDS normales (LED1, LED2)

La maquina debe funcionar como un sistema autónomo, mostrando información a través de un LCD, detectando su presencia mediante un sensor de ultrasonidos, midiendo temperatura y humedad con un DHT11, y permitiendo la selección de productos mediante un joystick y un botón principal.

## Descripción 
### a) Exigencias de la práctica 

Antes de empezar a explicar la solución debemos tener claro las exigencias del enunciado y como hemos garantizado su cumplimiento en nuestra solución:
  
  1. `Programación no bloqueante`:

El sistema debe evitar `delay()` en el flujo principal. 

En nuestra solución toda la temporización se gestiona con `millis()` lo que permite que la máquina siga respondiendo mientras se realizan tareas como el parpadeo inical del LED1, la espera entre estados, el tiempo de preparación del café, el tiempo de "retire bebida" y el tiempo entre lecturas del joystick o sensores. Aun así hacemos uso de micro-delays en zonas justificadas como microsegundos para ultrasonidos y pequeños bucles con `wdt_reset()` para mensajes fijos.
 
  
  2. `Uso de multihilo con la librería Thread`:

Debemos usar en la práctica todo el conocimiento adquirido en clase. En este caso, las técnicas de pseudo-concurrencia mediante hilos cooperativos. 

En nuestra solución, hacemos uso de tres threads independientes que explicaremos más adelante. Pero con ellos, podemos garantizar que estamos separando comportamientos y evitando que el flujo principal se sobrecargue. Además, ThreadController coordina la ejecución sin bloquear.
     
  3. `Interrupción hardware para el botón principal`:

El enunciado recomineda explicitamente utilizar interrupciones hardware para manejar los botones. 

En nuestra solución, implementamos una ISR (Interrupt Service Routine) asociada al botón principal. En la ISR se detecta cualquier cambio del botón (DOWN/UP) y se registra con debounce por microsegundos. De esta forma estamos garantizando la precisión en la lectura del tiempo de pulsación sin bloquear la CPU.
    
  4. `Uso de Watchdog para evitar bloqueos`:

El enunciado exige mantener el código seguro utilizando el watchdog para evitar bloqueos.

En nuestra solución, usamos watchdog y si por cualquier motivo el programa dejara de ejecutarse correctamente, el watchdog forzaría un reinicio automático del Arduino. De esta forma evitamos que se quede bloqueado permanentemente.
    
  5. `Manejo de sensores (DHT11, ultrasonidos, joystick, LCD)`:

Debemos hacer un uso de los sensores tal como nos pide el enunciado.

En nuestra solución, Todos los sensores están integrados tal como pide el enunciado. LCD con mensajes dinámicos, DHT11 para temperatura y humedad, Ultrasonidos con distancia no bloqueante, Joystick para navegación en menú y selección y Botón principal con interrupción.

### b) Descripción de la solución

El programa principal está organizado al rededor de una máquina de estados bien definida, los estados del sistema son:

1. `SERVICE_WAIT` → mientras el usuario no está presente. Se muestra “ESPERANDO CLIENTE”.
2. `SERVICE_SENSORS` → Se muestra la temperatura y humedad medido por el DHT11 durante 5 segundos.
3. `SERVICE_MENU` → Se muestra la lista de productos y precios navegable con el joystick.
4. `SERVICE_PREPARATION` → Tiempo aleatorio de preparación de café + LED2 progresivo.
5. `SERVICE_RETIRE` → Se muestra “RETIRE BEBIDA” por 3 segundos.

De esta forma la transición entre estados se controla mediante condiciones temporales, sensores o selección del usuario.

Además, el sistema utiliza tres hilos cooperativos gestionados por `ThreadController`. Estos hilos permiten dividir la lógica en bloques independientes que se ejecutan de manera periódica. Esto mejora la legibilidad, evita mezclas de responsabilidades y reduce el riesgo de bloquear accidentalmente el flujo principal. A continuación vamos a describir cada uno de los hilos, su función, su intervalo de ejecución y el motivo por el que existe:

1. `buttonThread`

Intervalo: 50 ms 

Función: Procesar eventos generados por la interrupción hardware del botón principal.

Este hilo es responsable de una de las partes más críticas del sistema. Debe detectar cuánto tiempo ha sido pulsado el botón y distinguir entre reset del estado de Servicio (2-3 segundos) y activar o desactivar Admin (≥5 segundos). 

De esta forma, el botón se lee desde una interrupción hardware. Pero la ISR no puede procesar la duración porque no está permitido ejecutar lógica costosa dentro de una interrupción, y esa es la razón de la existencia de este hilo. La ISR solo marca `btnEvent = true`,  el hilo recoge este evento, calcula la duración con `millis()` y ejecuta las acciones correspondientes. 

Usando este hilo, podemos mantener la ISR extremadamente ligera, evitar rebotes, garantizar que el sistema es preciso en las pulsaciones largas y que no se bloquea ningún otro comportamiento del sistema.

2. `adminLEDThread`

Intervalo: 80 ms

Función: Activar y mantener encendidos ambos LEDs cuando Admin está activo.

Este hilo se encarga exclusivamente de actualizar los LED durante el modo Administración. De esta forma, evitamos usar una lógica repetida en distintos puntos del código y solo revisamos la variable global `adminActive` y encendemos ambos leds a máxima potencia, cuando sea el caso.

Usamos este hilo porque así la lógica queda aislada, no interfiere con los estados del loop principal y es fácil de mantener. Además, al tener un intervalo pequeño (80 ms), asegura que los LEDs reflejen el modo Admin casi instantáneamente.

3. `adminMenuThread`

Intervalo: 120 ms 

Función: Gestionar toda la lógica dinámica del menú de administración.

Es el hilo más complejo y el corazón del modo Admin. 

Este hilo procesa: 

- Navegación por las opciones del menú principal
- Selección de opciones mediante el botón del joystick
- Vista dinámica de: Temperatura/Humedad, distancia, contador de segundos (desde inicio del programa)
- Modificación de precios: Incremento/decremento con joystick
- Confirmación con el botón
- Cancelación con joystick a la izquierda

Todo esto se controla mediante una máquina de estados interna (`adminState`), completamente independiente de la máquina del Servicio.

Este hilo nos ayuda a mantener el Servicio y el Admin completamente separados, para que así la navegación del usuario no interfiera con la máquina de estados del café. Además, asi el codigo es mucho más limpio y hace que sea imposible qeu el Servicio bloquee accidentalmente la navegación del Admin.

Con un intervalo de 120 ms, se actualiza suficientemente rápido para sentirse en tiempo real, además, no castiga al microcontrolador ni desperdicia tiempo de CPU y permite que el usuario navegue sin lag.

## Conclusión
La solución desarrollada cumple todos los requisitos del enunciado integrando correctamente sensores, actuadores, una máquina de estados y un sistema de control no bloqueante. El uso combinado de threads, interrupciones hardware y temporización con millis() permite un funcionamiento fluido y seguro, evitando bloqueos y manteniendo siempre la capacidad de respuesta.

Además, la separación entre el modo Servicio y el modo Admin, junto con el watchdog, aporta robustez y fiabilidad al sistema. En conjunto, el controlador implementado es estable, modular y refleja adecuadamente los conceptos trabajados en la asignatura de Sistemas Empotrados y de Tiempo Real.


## Videos/Imágenes

### Video del funcionamiento 

https://urjc-my.sharepoint.com/:v:/g/personal/j_sanmiguel_2023_alumnos_urjc_es/EYvx4qyawulAp8ZAKsQBYAEBKMjOTqol7JHkyb5TP-n9Jw?e=DUc7jn&nav=eyJyZWZlcnJhbEluZm8iOnsicmVmZXJyYWxBcHAiOiJTdHJlYW1XZWJBcHAiLCJyZWZlcnJhbFZpZXciOiJTaGFyZURpYWxvZy1MaW5rIiwicmVmZXJyYWxBcHBQbGF0Zm9ybSI6IldlYiIsInJlZmVycmFsTW9kZSI6InZpZXcifX0%3D

### Circuito Fritzing 

(NOTA: en la imagen no aparace un DHT11 (no aparecía en fritzing) aparece un RHT1 pero las conexíones son las que tendría mi DHT11) CONCLUSION abajo.
<img width="1046" height="612" alt="Captura desde 2025-11-26 03-58-08" src="https://github.com/user-attachments/assets/06cb0272-ddda-424b-b45b-b80638980e25" />
![WhatsApp Image 2025-11-27 at 05 12 01](https://github.com/user-attachments/assets/204f88db-c845-498f-9d2e-8c1729ca371a)



