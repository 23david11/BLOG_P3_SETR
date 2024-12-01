# BLOG_P3_SETR

David Pons Canet
01/12/24

## Objetivo

Se busca diseñar e implementar un controlador para una máquina expendedora que esté basado
en Arduino UNO y en los sensores/actuadores que se proporcionan en el kit Arduino. La práctica
tendrá que integrar obligatoriamente los siguientes componentes hardware:

  ● Arduino UNO
  ● LCD
  ● Joystick
  ● Sensor Temperatura/Humedad DHT11
  ● Sensor Ultrasonidos
  ● Botón
  ● 2 LEDS normales (LED1, LED2)

La maquina expendedora, consta de tres fases, ARRANQUE(setup), SERVICIO y ADMIN (maquina de estados).


## Arranque

Para implementar el arranque de la maquina expendedora, como algo que solo se tiene que ejecutar 
una vez, al iniciarse la placa, he decidido crear una funcion 'startup' y llamarla solo una vez
en el setup.

### Arduino Thread

Para el control del led en el arranque, he decidido usar la librería thread. Esta libreria me permite 
poder simular un thread que ejecute cada x timepo. He creado una clase LedThread, que me permite crear
un "thread" para que el led pueda parpadear cada segundo.


## Servicio

Para el desarrollo de servicio, lo he hecho con maquinas de estado. Hay dos estados principales
(WAIT y SELL). 

En este estado SERVICE, comprobamos si el boton ha estado pulsado durante 2-3 segundos para reiniciar
el estado de SERVICE.

Dentro de WAIT, hay una maquina de estados, con los estados DISTANCE y SENSORS. En el estado DISTANCE
printea en el lcd "esperando client" hasta que detecta una pesona a menos de 1m, y pasa al estado
SENSORS, donde durante 5 segundos enseña los valores del sensor de temperatura y humedad., despues 
activa el estado SELL, a la vez que vuelve a dejar este subestado a DISTANCE, porque una vez acabe
este cliente vuelva a esperar una persona.

En el estado SELL, hay otra maquina de estados con los subestados, SELECT, PREPARE y RETIRE.
en SELECT permitimos movernos por el menu y seleccionar el producto deseado. En PREPARE, simulamos
la preparacion del producto con un led encendiendose gradualmente. Y RETIRE, simplemente esperamos 
tres segundos desde que el producto esta listo.

Para moverme por el menu, lo hago a traves de un joystick. Con el metodo get_direction, me devuelve 
la posicion en la que esta el joystick, y con el metodo get_joystick con el resultado de get_direction
me permite moverme arriba y abajo por el menu.

Para seleccionar el producto usamos el metodo selectOption con argumento = 1, ya que queremos ejecutar
la funcion 1 definida para esa linia.


## Admin

Una vez se setea el modo admin, se enseña la pantalla 2 del menu. 

La funcionalidad de admin, tambien es una maquina de estados, con los estados SENSORS, DISTANCE,
COUNTER, PRICES,  y NONE.

En el estado SENSORS, simplemente mostramos los valores de los sensores, en DISTANCE la distancia, 
y en COUNTER el contador de sengundos que la placa lleva encendida.

En el estado PRICES, hay otra maquina de estados con los subestados SELECT y MODIFY. Cuando entramos
a PRICES, volvemos a cambiar a la pantalla 1 del menu para que vuelvan a salir los productos, para poder
seleccionar el producto a modificar.
En SELECT, solo la usamos para seleccionar el producto a modificar con selectOption, pero esta vez con
argumento = 2, para ejecutar la segunda funcion designada para esa linea del menu (modificar los precios).
Y en MODIFY, ejecutamos la funcion para modificar el precio como nos piden los requerimientos.

En el estado NONE, podemos navegar por el menu de la misma forma que en SERVICE.

## Interrupciones

### Service-Admin

Para cambiar de modo entre admin y service, lo hacemos a traves de un boton, conectadoa una interrupcion 
fisica del Arduino en modo CHANGE.

Por lo que cada vez que el estado del boton cambie, va a invertir el valor de un booleano 'reset', y se
guardara el tiempo actual, para en el bucle principal, poder ir calculando el tiempo que lleva pulsado el
boton. Y tambien para evitar rebotes(pulsaciones fantasma).

```c++
  attachInterrupt(digitalPinToInterrupt(INT_PIN), button_mode, CHANGE);

  void button_mode() {
    if ((millis() - last_reset) > BOUNCE_DELAY) {
      reset = !reset;
      init_time = millis();
      last_reset = millis();
    }
  }
```

Utilizamos last_reset para saber el tiempo que lleva sin recibir señal del boton, y si es mayor a un
BOUNCE_DELAY, consideramos que es una pulsacion valida.


### Select joystick

Para seleccionar los elemenos del menu, usamos el boton que incorpora el joystick, el cual tambien lo
vamos a conectar a una interrupcion fisica.

El funcionamiento es el mismo que con la otra interrupcion.

```c++
  attachInterrupt(digitalPinToInterrupt(SW_PIN), joystick_button, CHANGE);

  void joystick_button() {
    if ((millis() - last_joy) > BOUNCE_DELAY) {
      joy = !joy;
      last_joy = millis();
    }
  }
```

## Menu

Para el menu, he utilizado la libreria 'LiquidMenu'. Este menu, se compone por Lines y Screens.
Con las Lines formas Screens, y las Screens, se añaden al menu para poder visualizarlo.

Para poder editar el precio de los productos, he tenido que crear las lineas con una variable, 
la cual es la que cambiaremos al modificar un precio.

```c++
  LiquidLine linea1_1(1, 0, "Cafe Solo ", cafes[0], "e");

  LiquidScreen pantalla1(linea1_1, linea1_2, linea1_3, linea1_4, linea1_5);

  LiquidMenu menu(lcd,pantalla1,pantalla2);
```

Para poder navegar por el menu, hay que setear el cursor de las linias.

```c++
  linea1_1.set_focusPosition(Position::LEFT);
```

Podemos vincular una a cada linea, la cual se ejecutara si pulsamos el boton del joystic con
la linea selecionada.

```c++
  linea1_1.attach_function(1, prepare);
  linea1_1.attach_function(2, modify_prices); 
```

Para seleccionar la linea, usamos menu.call_function(f), donde f es el numero de que hemos puesto
en attach_function, asi podemos ejecutar mas de una funcion en cada linea cuando nos convenga. 


Para cambiar el cursor e ir navegando en el menu, subes i bajas de linea con:

```c++
  menu.switch_focus(true);  // hacia abajo
  menu.switch_focus(false); // hacia arriba
```

Para que se actualice el menu cuando vas navegadno por el, necesitas usar menu.update().


Para poder utilizar esta libreria, he tenido que modificar los archivos de libreria, ya que, 
por defecto, solo me dejaba añadir 4 lineas, y necesitaba 5. Por lo que mirando la libreria, 
me di cuenta que solo habia definido el constructor de LiquidLine hasta con 4 linas, por lo 
que añadi la definicion en LiquidMenu.h y el constructor en LiquidScreen.cpp.

```c++
  // en LiquidMenu.h
  /// Constructor for 5 LiquidLine object.
  /**
  @param &liquidLine1 - pointer to a LiquidLine object
  @param &liquidLine2 - pointer to a LiquidLine object
  @param &liquidLine3 - pointer to a LiquidLine object
  @param &liquidLine4 - pointer to a LiquidLine object
  @param &liquidLine5 - pointer to a LiquidLine object
  */
  LiquidScreen(LiquidLine &liquidLine1, LiquidLine &liquidLine2,
               LiquidLine &liquidLine3, LiquidLine &liquidLine4,
               LiquidLine &liquidLine5);

  ///@}


  // en LiquidScreen.cpp
  LiquidScreen::LiquidScreen(LiquidLine &liquidLine1, LiquidLine &liquidLine2,
                            LiquidLine &liquidLine3, LiquidLine &liquidLine4, 
                            LiquidLine &liquidLine5)
    : LiquidScreen(liquidLine1, liquidLine2, liquidLine3, liquidLine4) {
    add_line(liquidLine5);
  }
```

He utilizado un pequeño delay de 25 ms al final del loop, para que el lcd se vea consistente
sin temblar.


## Watchdog

Watchdog es un mecanismo de seguridad, el cual nos reinicia el sistema, si la placa se ha quedado 
pillada. He decidido setear el watchdog en 8 segundos, ya que he considerado que en una maquina 
expendora, no seria critico si se enganchara un momento.


## Esquema electronico

!["Esquema electronico"](esquema_p3.jpg)


## Video Funcionamiento

![ver_video](video_p3)
