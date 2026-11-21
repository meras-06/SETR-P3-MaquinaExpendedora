# Práctica 3: Controlador Máquina Expendedora

**Asignatura:** Sistemas Empotrados y de Tiempo Real (SETR)  
**Grado:** Ingeniería en Robótica Software — Universidad Rey Juan Carlos (URJC)  
**Autor:** Jorge Merás Ordax  

---

## 1. Introducción y Objetivo

En esta práctica se ha diseñado y montado el hardware y el software de control para una máquina expendedora basada en Arduino UNO (ATmega328P).

El sistema gestiona al mismo tiempo la interfaz de usuario (pantalla LCD, joystick con botón, pulsador externo y dos LEDs) y la lectura de los sensores de temperatura/humedad (DHT11) y distancia (HC-SR04), evitando bloqueos en la ejecución. Todo el funcionamiento se organiza en tres modos: Arranque, Servicio y Administración.

---

## 2. Montaje Hardware

### 2.1. Componentes utilizados
Todos los elementos pertenecen al kit de laboratorio entregado:
* Placa **Arduino UNO** y protoboard.
* **Pantalla LCD** y potenciómetro de 10 kΩ para control de contraste.
* **Módulo Joystick analógico** (ejes X, Y y botón integrado SW).
* **Sensor de Temperatura y Humedad DHT11**.
* **Sensor de Distancia por Ultrasonidos HC-SR04**.
* **Pulsador externo** para eventos de reinicio y acceso a Administración.
* **2 LEDs estándar** (`LED1` y `LED2`) con resistencias de 220 Ω.

### 2.2. Asignación de pines

La elección de pines se ha planificado para evitar conflictos de periféricos internos del microcontrolador:

| Componente | Pin del componente | Pin Arduino UNO | Justificación técnica |
| :--- | :--- | :--- | :--- |
| **Pulsador externo** | Terminal 1 / 2 | **Pin 2** / GND | Interrupción externa hardware `INT0` con `INPUT_PULLUP`. |
| **Joystick (Botón)** | SW | **Pin 3** | Lectura digital sin estado flotante mediante `INPUT_PULLUP`. |
| **Joystick (Ejes)** | VRx / VRy | **A1** / **A0** | Entradas al conversor ADC de 10 bits (rango 0–1023). |
| **Sensor DHT11** | DATA | **Pin 4** | Bus digital de 40 bits. |
| **LED1 (Estado)** | Ánodo (+) | **Pin 5** | Salida digital (con resistencia de 220 Ω a GND). |
| **LED2 (Progreso)** | Ánodo (+) | **Pin 6 (PWM)** | Salida PWM gobernada por **Timer0**, evitando el conflicto con **Timer1** en los pines 9, 10 y 11. |
| **Pantalla LCD** | RS / E | **Pin 12** / **Pin 11** | Líneas de control del bus LCD (pin `RW` conectado a GND). |
| **Pantalla LCD** | DB4 / DB5 / DB6 / DB7 | **10 / 9 / 8 / 7** | Bus de datos en modo de 4 bits. |
| **HC-SR04** | TRIG / ECHO | **A2** / **A3** | Pines analógicos empleados como pines digitales para liberar los pines serie `0` y `1`. |
| **Semilla aleatoria** | *(Sin conectar)* | **A5** | Lectura de ruido analógico flotante para `randomSeed()`. |

### 2.3. Montaje en el simulador: CirkitDesigner

![Montaje en CirkitDesigner](montaje_simulado.png)

### 2.4. Montaje real

![Vista general del montaje](montaje_real.jpeg)

### 2.5. Vídeo del funcionamiento 

[Vídeo del funcionamiento](video.mp4)

---

## 3. Arquitectura Software

Para mantener el código limpio, legible y reutilizable, la solución se ha dividido en **9 archivos independientes**: cada periférico tiene su propia clase heredada de `Thread`, mientras que la configuración, el catálogo, las interrupciones y la máquina de estados se separan en distintos ficheros.

```text
Practica3/
├── Practica3.ino          -> Punto de entrada: inicialización (setup) y bucle principal (loop)
├── Configuracion.h        -> Constantes de pines hardware y periodos de muestreo
├── Catalogo.h             -> Estructura de datos (struct Producto), precios y opciones del menú
├── Interrupciones.h       -> Variables volatile, ISR hardware (Pin 2) e ISR software (Timer1)
├── LedThread.h            -> Clase heredada de Thread para parpadeo, estado fijo y PWM de LEDs
├── UltrasonidosThread.h   -> Clase heredada de Thread para medición de distancia con HC-SR04
├── DHTThread.h            -> Clase heredada de Thread para lectura de temperatura y humedad
├── JoystickThread.h       -> Clase heredada de Thread para muestreo ADC y eventos del joystick
└── MaquinaEstados.h       -> Funciones estructuradas de los modos Arranque, Servicio y Admin
```
---

## 4. Recursos de Tiempo Real utilizados

### 4.1. Multitarea cooperativa con `ArduinoThread`

El **ATmega328P** no tiene paralelismo real en hardware (solo ejecuta una instrucción por ciclo de reloj), pero la máquina expendedora necesita atender varias tareas con frecuencias muy distintas: leer el joystick cada 80 ms, medir la distancia cada 200 ms, hacer parpadear el LED de arranque cada 1000 ms, leer el DHT11 cada 1500 ms y refrescar tanto el LCD como el PWM sin tirones.

- **Por qué no usar `delay()`:** `delay()` se queda en un bucle de espera activa mirando `micros()` y bloquea por completo la CPU. Meter un `delay(1000)` en el arranque o un `delay(5000)` al enseñar la temperatura impediría detectar pulsaciones o cambios en los sensores, además de poder hacer saltar el Watchdog.
- **Por qué `ArduinoThread` y no varios `if (millis() - prev >= T)` en `loop()`:** llevar muchos contadores manuales dentro del bucle principal acaba ensuciando el código y complica bastante añadir o modificar periféricos.
- **Ventajas de heredar de `Thread` y usar `ThreadController`:**
  1. Cada periférico tiene su propia clase heredada de `Thread` en su fichero `.h` (`LedThread`, `UltrasonidosThread`, `DHTThread`, `JoystickThread`), sobrescribiendo `shouldRun()` y `run()` (llamando siempre a `Thread::run()`). Esto deja el código bien separado y fácil de reutilizar.
  2. En cada vuelta del `loop()`, `controller.run()` comprueba a qué hilos les toca ejecutarse y lanza solo esos. Como la planificación es cooperativa, si un hilo se queda colgado bloquea todo el programa; por eso es importante evitar esperas largas en los sensores y tener activo el Watchdog.
  3. Los hilos se pueden activar o desactivar sobre la marcha. Por ejemplo, `led1Thread` se desactiva (`led1Thread->enabled = false`) en cuanto terminan los 3 parpadeos del arranque, y `led2Thread` se deja desactivado para no gastar CPU mientras se controla directamente por PWM o estado fijo.

| Hilo (`Thread`) | Intervalo | Motivo de la periodicidad |
| :--- | :---: | :--- |
| `LedThread` (`led1Thread`) | **1000 ms** | Marca el intervalo de 1 segundo para los 3 parpadeos del Modo Arranque; al terminar se desactiva (`enabled = false`). |
| `UltrasonidosThread` | **200 ms** | Con 5 lecturas por segundo se detecta bien si hay alguien a menos de 1 m y se actualiza rápido en Modo Admin, dejando tiempo de sobra para que vuelva el eco anterior. |
| `DHTThread` | **1500 ms** | El DHT11 apenas llega a 1 Hz de frecuencia máxima; leerlo más rápido suele dar errores. |
| `JoystickThread` | **80 ms** | 12.5 lecturas por segundo son suficientes para moverse con fluidez por los menús sin saturar el ADC. |

---

### 4.2. Interrupciones Hardware (`INT0`) y Software (`TimerOne`)

#### Interrupciones frente a *polling*
Hacer *polling* gasta ciclos de CPU comprobando el pin del pulsador aunque nadie lo esté tocando, y además se pueden perder pulsaciones o medir mal el tiempo si justo coincide con la lectura de un sensor. Con interrupciones, el microcontrolador pausa un momento el programa principal, guarda el contexto, ejecuta la ISR correspondiente y vuelve justo donde estaba.

#### Interrupción por Hardware (`INT0` en Pin 2, modo `CHANGE`)
- Mientras corre una ISR en Arduino las interrupciones globales están desactivadas, así que `millis()` no avanza y nunca se puede usar `delay()` dentro. Por eso `isrBotonExterno()` hace lo mínimo: lee el tiempo actual, aplica el anti-rebote, guarda cuándo se pulsó y activa las banderas `volatile`.
- Al pulsar o soltar un botón mecánico los contactos rebotan durante unos milisegundos generando flancos falsos. Para evitarlo (*debounce*), se ignora cualquier cambio que ocurra a menos de `40 ms` del anterior.
- Solo se marcan como `volatile` las variables compartidas entre las ISRs y el código principal (`contadorSegundos`, `tPresionBtnExt`, `tUltimoCambioISR`, `btnExtPresionado`, `flagCancelarServicio`, `flagToggleAdmin`). Así el compilador lee siempre su valor real desde RAM en vez de usar registros cacheados, sin penalizar el rendimiento del resto de variables.

#### Interrupción por Software (`TimerOne`) y prevención de conflictos de Timers
De los tres temporizadores hardware que tiene el ATmega328P se han usado dos:
- **`Timer0` (8 bits):** se encarga de `millis()`, `micros()`, `delay()` y el PWM a **980 Hz** de los **pines 5 y 6**. Al poner **`LED2` en el Pin 6**, su brillo por PWM durante la preparación del café funciona sin chocar con `millis()` ni con `TimerOne`.
- **`Timer1` (16 bits):** se reserva entero para **`TimerOne`**, configurado a $1\,000\,000\ \mu\text{s}$ ($1\text{ s}$) para lanzar `isrTimer1Segundo()` y sumar `contadorSegundos` con precisión. Como `TimerOne` pisa el PWM de los **pines 9, 10 y 11**, esos pines se usan solo como señales digitales para el LCD (nunca con `analogWrite()`).

---

### 4.3. Mecanismo de Seguridad: Watchdog Timer

El **Watchdog Timer (WDT)** es un temporizador independiente integrado en el microcontrolador con su **propio oscilador interno de 128 kHz**, separado del reloj principal de 16 MHz y del resto de timers. Si el código no reinicia su cuenta con `wdt_reset()` antes de que pase el tiempo configurado, el Watchdog fuerza un **reset físico del Arduino**.

#### Configuración en `WDTO_4S` (4 segundos)
Al principio de `setup()` se llama a `wdt_disable()` y luego a `wdt_enable(WDTO_4S)`, y en cada iteración de `loop()` se ejecuta `wdt_reset()`.
- En el peor caso, si en la misma vuelta coinciden el timeout del HC-SR04 (25 ms), una lectura del DHT11 (~20–25 ms), el muestreo del joystick y una actualización del LCD (~2–5 ms), todo el ciclo tarda menos de **60 ms**.
- Poner el límite en **4 segundos** da margen de sobra para que no haya reinicios falsos en uso normal, pero recupera la placa rápido si se produce un bloqueo real.

#### Influencia del Watchdog
En cada iteración completa de `loop()` se garantiza que el conjunto `controller.run()` + `actualizarMaquinaExpendedora()` termine siempre por debajo de 4 segundos y llegue a `wdt_reset()`. Además, al ser `ArduinoThread` cooperativo, si el `run()` de cualquiera de los hilos de periféricos (`UltrasonidosThread`, `DHTThread`, `JoystickThread`, `LedThread`) se quedara colgado, `controller.run()` nunca devolvería el control al `loop()` y el Watchdog resetearía la placa automáticamente. En el tramo de `setup()` posterior a `wdt_enable(WDTO_4S)`, si la inicialización de algún periférico (`lcd.begin()`, `dht.begin()`, etc.) bloqueara el arranque más de 4 segundos, el Watchdog forzaría un nuevo intento de arranque. Lo mismo ocurriría si el procesador quedase atrapado en una ISR o con las interrupciones globales deshabilitadas (`noInterrupts()`), ya que el Watchdog opera por hardware de forma independiente y resetearía el sistema al cumplirse los 4 segundos.

#### Casos donde NO influye el Watchdog
El Watchdog solo vigila que el programa no se quede congelado, pero **no puede detectar**:
- Errores de lógica que no bloqueen el bucle: si una condición mal escrita impidiera salir de un estado (por ejemplo, quedarse atascado en `SERV_ESPERA`), `loop()` seguiría dando vueltas y llamando a `wdt_reset()`, así que para el Watchdog todo iría bien.
- Sensores desconectados que devuelven error rápido: si se suelta el DHT11 (`readTemperature()` devuelve `NaN`) o el HC-SR04 (`pulseIn()` agota los 25 ms y devuelve `0`), el hilo acaba enseguida y se sigue refrescando `wdt_reset()`.
- Cualquier problema antes de ejecutar `wdt_enable(WDTO_4S)` durante el arranque o el bootloader.
- Fallos físicos en actuadores o datos erróneos (un LED fundido, el contraste `V0` del LCD mal ajustado o un precio mal puesto), ya que no detienen la CPU.

#### Puntos del código donde podría saltar el Watchdog
Revisando el proyecto, los sitios donde un fallo eléctrico o de software podría congelar la CPU y activar el Watchdog son:
1. **`UltrasonidosThread.h` (`pulseIn(pinEcho, HIGH, 25000)`):** es una función bloqueante que espera la llegada del eco. Aunque está limitada a `25000 us` para no colgar el programa, un fallo grave en `ECHO` sin timeout o un bucle de reintentos mal planteado detendría la ejecución hasta que saltase el Watchdog.
2. **`DHTThread.h` (`dht.readTemperature()` / `dht.readHumidity()`):** la librería de Adafruit lee la trama de 40 bits con bucles `while` cronometrados en microsegundos y con las interrupciones desactivadas. Un fallo eléctrico en `PIN_DHT` que impidiera salir de esa espera provocaría el reset a los 4 segundos.
3. **`Interrupciones.h` (`isrBotonExterno`):** si un mal contacto o ruido en el Pin 2 disparase `INT0` sin parar, la CPU no podría volver al `loop()` para hacer `wdt_reset()`.
4. **Bloqueo en `MaquinaEstados.h`:** si en algún cambio futuro se metiera un bucle de espera (`while`) del que nunca se saliera, a los 4 segundos el Watchdog reiniciaría la placa y la devolvería al **Modo Arranque**.

---

## 5. Funcionamiento de los Modos de Operación

### 1. Modo Arranque (`MODO_ARRANQUE`)
Al encender o reiniciar el Arduino, el LCD muestra **`CARGANDO ...`** mientras `led1Thread` (configurado cada `1000 ms`) va cambiando el estado de `LED1` sin bloquear el resto del programa. Cuando `led1Thread->cambiosRealizados` llega a `6` (`ON-OFF-ON-OFF-ON-OFF`), se han completado los **3 parpadeos de 1 segundo**. En ese momento se llama a `entrarModoServicio()`, que desactiva el hilo (`led1Thread->enabled = false`) para ahorrar CPU, apaga `LED1` y pasa al Modo Servicio.

### 2. Modo Servicio (`MODO_SERVICIO`)
Es el modo normal de uso de la máquina y está dividido en cinco subestados controlados con `millis()`:

- **Detección de presencia (`SERV_ESPERA`):** mientras la distancia leída por `usThread` sea mayor o igual a 1 metro (`usThread->distanciaCm >= 100.0`), el LCD muestra **`ESPERANDO CLIENTE`**. En cuanto detecta algo a menos de 1 metro (`< 100.0 cm`), guarda `tInicioFase = millis()` y pasa al siguiente estado.
- **Lectura del entorno (`SERV_ENTORNO`):** el LCD muestra la temperatura (`°C`) y la humedad (`%`) medidas por `dhtThread`, actualizándose cada segundo durante **5 segundos** (`millis() - tInicioFase >= 5000`).
- **Catálogo y selección (`SERV_CATALOGO`):** pasados los 5 segundos, aparecen las 5 bebidas definidas en `Catalogo.h` con su precio a dos decimales:
  - `Café Solo`: **1.00 €**
  - `Café Cortado`: **1.10 €**
  - `Café Doble`: **1.25 €**
  - `Café Premium`: **1.50 €**
  - `Chocolate`: **2.00 €**
  Se recorre la lista de forma circular moviendo el joystick arriba o abajo (`JOY_UP` / `JOY_DOWN`) y se elige pulsando el botón integrado (`JOY_CLICK`).
- **Preparación (`SERV_PREPARANDO`):** al confirmar la bebida se sortea un tiempo aleatorio entre **4 y 8 segundos** con `tPreparacion = random(4000, 8001)` (usando `randomSeed(analogRead(PIN_SEED))` en `setup()`). El LCD muestra **`Preparando Cafe...`** y, en cada vuelta del bucle, se calcula el tiempo transcurrido (`transcurrido = millis() - tInicioFase`) para mapearlo al rango PWM de 8 bits con `map(transcurrido, 0, tPreparacion, 0, 255)`, subiendo el brillo de `LED2` (Pin 6, Timer0) del 0% al 100%.
- **Retirada (`SERV_RETIRE`):** al terminar `tPreparacion`, se apaga `LED2` y el LCD enseña **`RETIRE BEBIDA`** durante **3 segundos** (`3000 ms`), volviendo después a `SERV_ESPERA`.
- **Cancelación (Botón externo 2–3 s):** en cualquier punto entre `SERV_ENTORNO` y `SERV_RETIRE`, mantener pulsado el botón externo entre **2 y 3 segundos** activa `flagCancelarServicio` desde la interrupción `INT0`. El programa corta la operación actual, apaga `LED2` y vuelve a `SERV_ESPERA` sin reiniciar físicamente la placa.

### 3. Modo Administración (`MODO_ADMIN`)
Sirve para consultar los sensores y modificar los precios de las bebidas sin tocar el código:

- **Acceso y salida (Botón externo ≥ 5 s):** se puede entrar desde cualquier modo manteniendo el botón externo pulsado al menos **5 segundos seguidos** (`>= 5000 ms`). Con `verificarPulsacionLargaAdmin()`, el cambio salta justo al cumplirse los 5 segundos. Para salir y volver al Modo Servicio se hace exactamente la misma pulsación.
- **Indicadores:** al entrar mediante `entrarModoAdmin()`, tanto `LED1` como `LED2` se quedan encendidos fijos (`HIGH`) mientras se esté dentro de este modo.
- **Menú (`ADM_MENU`):** el LCD muestra las 4 opciones (`a.Ver temp`, `b.Ver distancia`, `c.Ver contador`, `d.Modif precios`), que se recorren con arriba/abajo y se abren con `JOY_CLICK`. Mover el joystick a la izquierda (`JOY_LEFT`) vuelve atrás desde cualquier pantalla:
  1. **Ver temperatura (`ADM_TEMP`):** muestra las lecturas del DHT11 (`Temp: XX C` y `Hum: YY %`), refrescando cada `500 ms`.
  2. **Ver distancia (`ADM_DIST`):** enseña la distancia actual del HC-SR04 (`Distancia: XX cm`), refrescando cada `250 ms`.
  3. **Ver contador (`ADM_CONT`):** muestra los segundos transcurridos desde que se encendió el Arduino (`contadorSegundos`), actualizados cada segundo por `Timer1`.
  4. **Modificar precios (`ADM_SEL_PRECIO` → `ADM_EDIT_PRECIO`):** deja elegir un producto de la lista y, al pulsar el botón del joystick, copia su precio en `precioTemp`. Moviendo arriba o abajo se sube o baja en pasos de **5 céntimos (`0.05 €`)**. Al pulsar el botón se guarda el cambio en `catalogo[indiceProd].precio` (se aplica al instante al volver al Modo Servicio, sin usar EEPROM), mientras que mover a la izquierda descarta los cambios y vuelve a la lista.
---

## 6. Conclusiones

A lo largo de esta práctica se han trabajado los principales recursos del ATmega328P del Arduino UNO: entradas analógicas (ADC), salidas PWM, interrupciones, temporizadores, hilos cooperativos y el Watchdog.

Dividir el código en 9 ficheros (encapsulando cada periférico en su propia clase heredada de `Thread`) ha facilitado bastante la depuración y ha permitido dejar un `loop()` muy limpio. Gracias a `ThreadController` y a los controles de tiempo con `millis()` se ha eliminado por completo `delay()`, logrando leer los sensores, actualizar el LCD y variar el brillo de `LED2` por PWM sin congelar el programa, además de desactivar los hilos cuando ya no hacen falta (`enabled = false`).

Por otro lado, usar la interrupción externa `INT0` (Pin 2) con anti-rebote ha permitido distinguir sin fallos entre la pulsación de cancelación (2–3 s) y la de administración (≥ 5 s), mientras que `TimerOne` se encarga de llevar el contador de segundos con precisión. Para evitar que `Timer1` interfiriera con el PWM, se llevó `LED2` al Pin 6 (gobernado por `Timer0`). A esto se suma el Watchdog configurado a 4 segundos (`WDTO_4S`), que actúa como respaldo por si algún sensor o interrupción llegase a colgar la placa.

Por último, pasar de la simulación en CirkitDesigner al montaje real en protoboard sacó a la luz problemas prácticos que en el simulador no aparecen, como el ajuste fino del contraste del LCD (`V0`) con el pin `RW` a masa, los rebotes físicos del pulsador o la necesidad de meter una zona muerta en los ejes del joystick para que el menú no salte solo.
