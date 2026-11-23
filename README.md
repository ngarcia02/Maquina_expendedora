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
-Al mismo tiempo, el LED parpadea tres veces, en intervalos de un segundo. Para esta implementación se ha creado la clase LedThread, que hereda de Thread y cuenta los parpadeos completados. Cuando llega a tres apaga el LED y lo comunica a través del método haTerminado(). Además dispone de un método reiniciar(), por si hiciera falta repetir el parpadeo, y se ejecuta de nuevo cada vez que la placa se reinicia.

Una vez concluida esta ejecución se procede al apagado del LED1 y al paso al segundo modo principal, el modo Servicio. 
### 🟡 MODO SERVICIO
En este segundo modo disponemos de un total de cinco estados: ESPERA, ENTORNO, CATÁLOGO, PREPARACIÓN y RETIRAR. Tres de ellos (ENTORNO, PREPARACIÓN y RETIRAR) tienen una duración y son controlados por un thread (thread_temporizador), encargado de gestionar el intervalo de cada uno. ESPERA depende de la distancia medida y CATÁLOGO de la interacción con el joystick.

Para mayor seguridad a la hora de transitar por los diferentes submodos se ha creado una función encargada de gestionar la limpieza y inicialización  de cada uno de estos subestados, garantizando la limpieza de la pantalla lcd, sin caer en parpadeos en la pantalla, y la actualización de variables como es el caso del indice del catálogo, permitiendo de esa forma acceder a la lista de productos siempre desde el producto inicial.

#### Modo ESPERA
- En este subestado inicial realizamos la lectura de la distancia por medio del ultrasonidos, derivando en dos casos
  ##### CLIENTE DETECTADO
  - Al medir una distancia menor a 1 metro realizamos el cambio de estado a el modo Entorno.
  - La medida no se realiza de forma continua, sino desde un thread (thread_distancia) que mide cada 150 ms. Así se evita encadenar medidas sin pausa entre ellas, que dan lecturas erróneas, y se consigue un valor más estable, que además permite una mejor visualización en el modo Administración.
  ##### CLIENTE NO DETECTADO
  - En caso de que no se haya detectado ninguna medición menor a un metro permanecemos a la espera y mostramos por la pantalla LCD el mensaje "ESPERANDO CLIENTE".
    
#### Modo ENTORNO
- Una vez hemos detectado un cliente mostramos por pantalla durante un intervalo de 5 segundos la temperatura y humedad detectada por el sensor DHT11, comprobando antes que ninguna lectura haya fallado. Posteriormente transicionamos al subestado Catálogo donde mostraremos el menú de productos disponibles junto a sus precios.
  
#### Modo CATALOGO
- En este modo realizamos la navegación y selección en el catálogo, para ello hacemos uso de el joystick, del cual leemos su valor para su pin del eje y, y por medio de el establecemos el movimiento realizado y transicionamos el producto mostrado del catálogo. Para evitar una navegación imprecisa guardamos los movimientos anteriores , de forma que evitamos transicionar muy rápido y realizar un borrado de pantalla innecesario.
- Para un mejor entendimiento guardé la primera fila de la pantalla LCD para el encabezado de la lista, indicando que el primer termino será el producto y el segundo su precio base. La segunda fila de la pantalla es la encargada de mostrar los productos y de reescribirse cuando se navegue con el joystick.
- Para cambiar al siguiente estado se debe seleccionar el producto con el pulsador del joystick. También se detecta como un flanco, para que una sola pulsación equivalga a una sola selección. A continuación se pasa al estado PREPARACIÓN.

#### Modo PREPARACIÓN
- En este estado se hace uso de la pantalla LCD, que muestra el mensaje "Preparando café..." durante un intervalo aleatorio, y del LED2, que indica el progreso de la preparación aumentando su brillo mediante modulación PWM (analogWrite) a medida que se alcanza el tiempo establecido. El brillo es proporcional al tiempo transcurrido y, para no superar el valor máximo de luminosidad (255), se ha establecido un tope.
- Este intervalo aleatorio se selecciona por medio de random, que proporcionará un tiempo aleatorio entre 4 y 8 segundos.

Una vez cumplido esto apagamos el led, y transicionamos al siguiente estado.

#### Modo RETIRAR
- Nuevamente limpiamos la pantalla y escribimos "RETIRE BEBIDA" durante un periodo de tres segundos. Una vez terminados volvemos nuevamente al estado inicial de reposo, modo ESPERA.

Por medio de una interrupción se incorpora la posibilidad de cancelar la operación y reiniciar el estado interno de este modo, regresando inmediatamente al estado ESPERA, al mantener pulsado el botón externo entre 2 y 3 segundos.

### 🟠 MODO ADMINISTRACIÓN
Al igual que en el modo servicio hacemos uso de interrupciones, en este caso para acceder y salir del modo administración. Para entrar, la pulsación debe realizarse durante el modo Servicio y mantenerse durante cinco segundos o más; una vez cumplido esto se pasa directamente a Administración y se inicia en el estado NAVEGACIÓN. Para salir se repite la misma pulsación y se vuelve al modo Servicio, en el estado ESPERA. Mientras se está en este modo, los dos LEDs permanecen encendidos.

#### Modo NAVEGACIÓN
Este modo, al igual que el de Catálogo nos permite recorrer una lista de opciones, en este caso se nos muestran las opciones del menú de administración, a las cuales podemos acceder por medio del pulsador del joystick como en el caso anterior. Como añadido en esta sección, al acceder a los siguientes submodos podemos retroceder del submenú moviendo el joystick hacia la izquierda, que se leerá por medio del pin del eje x.

#### Modo TEMPERATURA
- Mostramos en la pantalla del LCD la temperatura y humedad leída por el sensor DHT11, y actualizamos de forma dinámica.

#### Modo DISTANCIA
- Se muestra la lectura de la distancia en centímetros por la pantalla LCD. El valor procede del mismo thread de medida (cada 150 ms), y la pantalla solo se redibuja cuando el valor cambia, para evitar parpadeos.
  
#### Modo CONTADOR
- Mostramos el tiempo transcurrido desde el arranque de la placa. Para ello se actualiza el tiempo al inicio de cada iteración de loop() y el cálculo en segundos se realiza en este estado.
  
#### Modo PRECIOS
- Mostramos nuevamente el catálogo de productos, pero esta vez al seleccionar permitimos modificar el precio aumentando y disminuyendo el precio de cinco centimos en cinco céntimos en función de si el joystick se ha movido hacia arriba o hacia abajo, y fijar por medio nuevamente del pulsador del mismo. Además, permitimos descartar selecciones moviendo el joystick hacia la izquierda.
- Como consecuencia de esta sección se modificó el array con los precios estableciéndolos en céntimos, y realizando su transición a euros por medio de una función aparte.

## RECURSOS DE TIEMPO REAL

### 🔧 WATCHDOG
Para garantizar que el sistema se recupere ante un bloqueo inesperado se ha configurado el temporizador Watchdog del microcontrolador mediante la librería avr/wdt.h. Se activa una sola vez en setup() con wdt_enable(WDTO_2S) y se refresca con wdt_reset() en cada iteración de loop(). El tiempo se eligió a partir de lo más largo que puede bloquear una vuelta del bucle principal: la medida de distancia, que al no usar un timeout propio en pulseIn puede esperar hasta 1 segundo cuando no llega eco, más la lectura del sensor DHT11. Dos segundos dejan un margen amplio sobre ese peor caso y evitan reinicios falsos. 
Si el programa dejara de ejecutar el bucle y no llegara a refrescarlo, la placa se reiniciaría y volvería al modo Arranque, con lo que los precios modificados en el modo Administración recuperan sus valores iniciales, ya que no se guardan en EEPROM. Para comprobarlo se provocó un bloqueo deliberado y se verificó que el sistema se reiniciaba.

### THREADS
Para evitar el uso de delay(), que bloquearía el programa e impediría atender el joystick o el botón externo mientras se espera, se ha hecho uso de la librería ArduinoThread junto a un ThreadController, que en cada iteración del loop() comprueba qué threads deben ejecutarse. Se han utilizado tres: 
- Thread del LED1 (LedThread) en el modo Arranque: cuenta los parpadeos y notifica cuando ha terminado.
- thread_temporizador: un temporizador reutilizado por varios estados del modo Servicio (ENTORNO, PREPARACIÓN y RETIRAR), cada uno con su intervalo. Al cumplirse, ejecuta una función de callback que realiza la transición al siguiente estado. Se implementa en la clase Temporizador, que hereda de Thread, porque el método runned() de la librería, necesario para reiniciar la cuenta del intervalo desde el momento de entrar en cada estado, es protegido y solo puede llamarse desde una clase derivada.
- thread_distancia: establece un intervalo (150 ms) para la medición de la distancia, de modo que no se lea de forma continua y la distancia resulte más estable y fácil de visualizar.

De esta forma el sistema sigue respondiendo mientras espera.
 ### Interrupciones
 El botón externo se gestiona mediante una interrupción hardware en el pin 2, configurada en modo CHANGE para detectar tanto la pulsación como la liberación. De esta forma se mide con precisión la duración de cada pulsación sin necesidad de consultar el botón continuamente.
La rutina de interrupción (ISR) es lo más corta posible. Incorpora un antirrebote de 30 ms, guarda el instante en que se pulsa y, al soltar, calcula y guarda la duración, activando un indicador.
## PROBLEMAS ENCONTRADOS
- El principal problema detectado fue la correcta distribución de los componentes en el circuito, ya que debido a las dimensiones de algunos de ellos, como es el caso de la pantalla LCD, fue complejo disponerlos de tal manera que se pudiesen acceder a todos los componentes con los que había que interactuar visualmente o físicamente. Este problema también se debe a la falta de cables largos y la abundancia de cables cortos que causan una distribución muy pegada entre los componentes.
- La navegación con el joystick inicialmente supuso un reto, ya que inicialmente no tenía en cuenta movimientos anteriores y eso causaba problemas en la escritura del LCD y problemas para navegar correctamente por los menús.
- El contraste del LCD supuso un problema para la visualización, pero se arregló al introducir al circuito un potenciómetro que permite ajustar el contraste en cualquier momento.
  
## VIDEOS



