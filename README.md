<h1 align="center">Práctica 3: Controlador de máquina expendedora</h1>

<p align="center">
<img width="298" height="200" alt="200" src="https://github.com/user-attachments/assets/78d0496e-5b8f-4823-a65b-67f36b2af4c5" />
</p>

## INTRODUCCIÓN

En esta práctica se desarrolla el diseño y la implementación del controlador software para una máquina expendedora automática basada en la plataforma Arduino UNO, haciendo uso de los diferentes componentes dispuestos en el kit de la asignatura.
A continuación se señalarán los componentes usados en el circuito junto a su implementación en el código, y los mecanismos usados para el cumplimiento de los comportamientos.

## MONTAJE DEL CIRCUITO

### ⚙️ Componentes hardware:
- Arduino UNO.
- Pantalla LCD.
- Módulo Joystick.
- Sensor de Temperatura y Humedad DHT11.
- Sensor de Distancia por Ultrasonidos.
- Pulsador / Botón externo.
- 2 LEDs estándar: LED1 y LED2.
- Potenciometro.

### 📷Circuito:


## ESQUEMÁTICO

## ARQUITECTURA
Como paso inicial localicé los diferentes modos y submodos contenidos en el ejercicio, los cuales declaré por medio de enums para cada una de los diferentes modos principales.

### 🟢 MODO ARRANQUE
En este modo inicial no tenemos ningún derivado, su única función es mostrar el arranque del sistema por medio de los dispositivos hardware, concretamente la pantalla LCD y el LED1.
##### Pantalla LCD: 
- Se encarga de mostrar por pantalla el mensaje "CARGANDO..." durante 3 segundos.
##### LED1:
- Al mismo tiempo, el led parpadea durante tres segundos en intervalos de un segundo. Para esta implementación se ha hecho uso de la clase Threads la cual ha sido modificada para realizar estos parpadeos al iniciar el sistema, y en caso de que se reinicie el sistema y por tanto se ejecute nuevamente el modo arranque.

Una vez concluida esta ejecución se procede al apagado del LED1 y al paso al segundo modo principal, el modo servicio. 
### 🟡 MODO SERVICIO
En este segundo modo disponemos de un total de cinco submodos los cuales son controlados por un thread (thread_temporizador), encargado de gestionar los intervalos establecidos en cada uno de los submodos.
Para mayor seguridad a la hora de transitar por los diferentes submodos se ha creado una función encargada de gestionar la limpieza y inicialización  de cada uno de estos subestados, garantizando la limpieza de la pantalla lcd, sin caer en parpadeos en la pantalla, y la actualización de variables como es el caso del indice del catálogo, permitiendo de esa forma acceder a la lista de productos siempre desde el producto inicial.

#### Modo ESPERA
- En este subestado inicial realizamos la lectura de la distancia por medio del ultrasonidos, derivando en dos casos
  ##### CLIENTE DETECTADO
  - Al medir una distancia menor a 1 metro realizamos el cambio de estado a el modo entorno.
  - Para una mejor medición posteriormente en el modo Administración se ha establecido un thread encargado de que la medición de la distancia se realice cada 150 ms, de tal forma que evitamos mediciones erroneas y posteriormente permitimos una mejor visualización de las distancias calculadas.
  ##### CLIENTE NO DETECTADO
  - En caso de que no se haya detectado ninguna medición menor a un metro permanecemos a la espera y mostramos por la pantalla LCD el mensaje "ESPERANDO CLIENTE".
    
#### Modo ENTORNO
- Una vez hemos detectado un cliente mostramos por pantalla durante un intervalo de 5 segundos la temperatura y humedad detectada por el sensor DHT11, el cual comprobamos que ninguna lectura haya fallado antes de escribir en por pantalla. Posteriormente transicionamos al subestado Catálogo donde mostraremos el menú de productos disponibles junto a sus precios.
  
#### Modo CATALOGO
- En este modo realizamos la navegación y selección en el catálogo, para ello hacemos uso de el joystick, del cual leemos su valor para su pin del eje y, y por medio de el establecemos el movimiento realizado y transicionamos el producto mostrado del catálogo. Para evitar una navegación imprecisa guardamos los movimientos anteriores , de forma que evitamos transicionar muy rápido y realizar un borrado de pantalla innecesario.
- Para un mejor entendimiento guardé la primera fila de la pantalla LCD para el encabezado de la lista, indicando que el primer termino será el producto y el segundo su precio base. La segunda fila de la pantalla es la encargada de mostrar los productos y de reescribirse cuando se navegue con el joystick.
- Para cambiar al siguiente subestado se debe seleccionar con el pulsador de el joystick el producto elegido, de esa forma se detectará la pulsación y se pasará al modo Preparación.

#### Modo PREPARACIÓN
- En este modo se hace uso de la pantalla LCD, la cual mostrará el mensaje "Preparando café..." durante un intervalo aleatorio, y el LED2 el cuál indicará el progreso de la preparación aumentando su brillo según se alcanza el tiempo establecido. Para que no tomar valores superiores al tope de luminosidad, 255, se ha establecido un tope.
- Este intervalo aleatorio se selecciona por medio de random, que proporcionará un tiempo entre 4 y 8 segundos. Para evitar repeticiones se hace uso de una semilla.

Una vez cumplido esto apagamos el led, y transicionamos al siguiente modo.

#### Modo RETIRAR
- Nuevamente limpiamos la pantalla y escribimos "RETIRE BEBIDA" durante un periodo de tres segundos. Una vez terminados volvemos nuevamente al estado inicial de reposo, modo ESPERA.

Por medio de interrupciones incorporamos la posibilidad de reiniciar el estado interno de esta sección y regresar inmediatamente al modo ESPERA, al pulsar durante un tiempo determinado de 2 a 3 segundos el botón externo del circuito.

### 🟠 MODO ADMINISTRACIÓN
Al igual que en el modo servicio hacemos uso de interrupciones, en este caso para acceder y salir del modo administración. En este caso se debe cumplir que la pulsación se realice durante el modo servicio, durante un tiempo mayor o igual a cinco segundos, una vez cumplido eso pasamos directamente a administración e iniciamos desde el submodo navegación.

#### Modo NAVEGACIÓN
Este modo al igual que el de catálogo nos permite vislumbrar las opciones de un una lista, en este caso se nos muestran las opciones de submodo, a las cuales podemos acceder por medio del pulsador del joystick como en el caso anterior. Como añadido en esta sección al acceder a los siguientes submodos podemos retroceder del submenú moviendo el joystick hacia la izquierda, que se leerá por medio del pin del eje x.

#### Modo TEMPERATURA
- Mostramos en la pantalla del LCD la temperatura y humedad leída por el sensor DHT11, y actualizamos de forma dinámica.

#### Modo DISTANCIA
- Mostramos la lectura de la distancia en centímetros por la pantalla LCD.
  
#### Modo CONTADOR
- Mostramos el tiempo transcurrido desde el arranque de la placa. Para ello iniciamos a contar el tiempo transcurrido en el inicio del loop, y realizamos el calculo en el modo contador.
  
#### Modo PRECIOS
- Mostramos nuevamente el catálogo de productos, pero esta vez al seleccionar permitimos modificar el precio aumentando y disminuyendo el precio de cinco centimos en cinco céntimos en función de si el joystick se ha movido hacia arriba o hacia abajo, y fijar por medio nuevamente del pulsador del mismo. Además, permitimos descartar selecciones moviendo el joystick hacia la izquierda.
- Como consecuencia de esta sección se modificó el array con los precios estableciéndolos en céntimos, y realizando su transición a euros por medio de una función a parte.

### 🔧 WATCHDOG
Para garantizar que el sistema se recupere ante un bloqueo inesperado se ha configurado el temporizador Watchdog del microcontrolador mediante la librería avr/wdt.h. Se activa una sola vez en setup() con wdt_enable(WDTO_2S) y se refresca con wdt_reset() en cada iteración de loop(). El tiempo se eligió a partir de lo más largo que puede bloquear una vuelta del bucle principal: la medida de distancia, que al no usar un timeout propio en pulseIn puede esperar hasta 1 segundo cuando no llega eco, más la lectura del sensor DHT11. Dos segundos dejan un margen amplio sobre ese peor caso y evitan reinicios falsos. Si el programa dejara de ejecutar el bucle y no llegara a refrescarlo, la placa se reiniciaría y volvería al modo Arranque, con lo que los precios modificados en el modo Administración recuperan sus valores iniciales, ya que no se guardan en EEPROM. Para comprobarlo se provocó un bloqueo deliberado y se verificó que el sistema se reiniciaba.

### THREADS
Para evitar el uso de delay(), que bloquearía el programa e impediría atender el joystick o el botón externo mientras se espera, se ha hecho uso de la librería ArduinoThread junto a un ThreadController, que en cada iteración del loop() comprueba qué threads deben ejecutarse. Se han utilizado tres: uno para el parpadeo del LED1 en el modo Arranque, que cuenta los parpadeos y notifica cuando ha terminado; thread_temporizador, un temporizador reutilizado por varios submodos del modo Servicio (ENTORNO, PREPARACIÓN y RETIRAR), cada uno con su intervalo, que al cumplirse ejecuta una función de callback que realiza la transición al siguiente estado; y thread_distancia, que establece un intervalo para la medición de la distancia, de modo que no se lea de forma continua y la distancia resulte más estable y fácil de visualizar en el modo Administración. De esta forma el sistema sigue respondiendo mientras espera. Por el contrario, para el botón externo se emplean interrupciones, ya que es necesario medir con precisión la duración de cada pulsación.

## PROBLEMAS ENCONTRADOS
- El principal problema detectado fue la correcta distribución de los componentes en el circuito, ya que debido a las dimensiones de algunos de ellos, como es el caso de la pantalla LCD, fue complejo disponerlos de tal manera que se pudiesen acceder a todos los componentes con los que había que interactuar visualmente o físicamente. Este problema también se debe a la falta de cables largos y la abundancia de cables cortos que causan una distribución muy pegada entre los componentes.
- La navegación con el joystick inicialmente supuso un reto, ya que inicialmente no tenía en cuenta movimientos anteriores y eso causaba problemas en la escritura del LCD y problemas para navegar correctamente por los menús.
- El contraste del LCD supuso un problema para la visualización, pero se arregló al introducir al circuito un potenciómetro que permite ajustar el contraste en cualquier momento.
  
## VIDEOS



