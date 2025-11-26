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
  
  1. Programación no bloqueante 

El sistema debe evitar `delay()` en el flujo principal. 

En nuestra solución toda la temporización se gestiona con `millis()` lo que permite que la máquina siga respondiendo mientras se realizan tareas como el parpadeo inical del LED1, la espera entre estados, el tiempo de preparación del café, el tiempo de "retire bebida" y el tiempo entre lecturas del joystick o sensores. Aun así hacemos uso de micro-delays en zonas justificadas como microsegundos para ultrasonidos y pequeños bucles con `wdt_reset()` para mensajes fijos.
 
  
  2. Uso de multihilo con la librería Thread 

Debemos usar en la práctica todo el conocimiento adquirido en clase. En este caso, las técnicas de pseudo-concurrencia mediante hilos cooperativos. 

En nuestra solución, hacemos uso de tres threads independientes que explicaremos más adelante. Pero con ellos podemos garantizar que de esta forma estamos separando comportamientos y evitando que el flujo principal se sobrecargue. Además, ThreadController coordina la ejecución sin bloquear.
     
  3. Interrupción hardware para el botón principal

El enunciado recomineda explicitamente utilizar interrupciones hardware para manejar los botones. 

En nuestra solución, implementamos una ISR (Interrupt Service Routine) asociada al botón principal. En la ISR se detecta cualquier cambio del botón (DOWN/UP) y se registra con debounce por microsegundos. De esta forma estamos garantizando la precisión en la lectura del tiempo de pulsación sin bloquear la CPU.
    
  4. Uso de Watchdog para evitar bloqueos

El enunciado exige mantener el código seguro utilizando el watchdog para evitar bloqueos.

En nuestra solución, usamos watchdog y si por cualquier motivo el programa dejara de ejecutarse correctamente, el watchdog forzaría un reinicio automático del Arduino. De esta forma evitamos que se quede bloqueado permanentemente.
    
  5. Manejo de sensores (DHT11, ultrasonidos, joystick, LCD):

Debemos hacer un uso de los sensores tal como nos pide el enunciado.

En nuestra solución, Todos los sensores están integrados tal como pide el enunciado. LCD con mensajes dinámicos, DHT11 para temperatura y humedad, Ultrasonidos con distancia no bloqueante, Joystick para navegación en menú y selección y Botón principal con interrupción.

### b) Descripción de la solución


## Video del funcionamiento 

https://urjc-my.sharepoint.com/:v:/g/personal/j_sanmiguel_2023_alumnos_urjc_es/EYvx4qyawulAp8ZAKsQBYAEBKMjOTqol7JHkyb5TP-n9Jw?e=DUc7jn&nav=eyJyZWZlcnJhbEluZm8iOnsicmVmZXJyYWxBcHAiOiJTdHJlYW1XZWJBcHAiLCJyZWZlcnJhbFZpZXciOiJTaGFyZURpYWxvZy1MaW5rIiwicmVmZXJyYWxBcHBQbGF0Zm9ybSI6IldlYiIsInJlZmVycmFsTW9kZSI6InZpZXcifX0%3D

## Circuito Fritzing 
<img width="1046" height="612" alt="Captura desde 2025-11-26 03-58-08" src="https://github.com/user-attachments/assets/06cb0272-ddda-424b-b45b-b80638980e25" />
