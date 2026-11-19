# Práctica 3: Controlador de Máquina Expendedora en Arduino UNO

**Asignatura:** Sistemas Empotrados y de Tiempo Real (SETR)  
**Grado:** Ingeniería en Robótica Software — Universidad Rey Juan Carlos (URJC)  
**Autor:** Jorge Merás Ordax  

---

## 1. Introducción y Objetivo

En esta práctica se ha diseñado e implementado el sistema hardware y el firmware de control en tiempo real para una **máquina expendedora automática** basada en **Arduino UNO (ATmega328P)**.

El controlador gestiona de forma concurrente y no bloqueante la interacción con el usuario (pantalla LCD 16x2, joystick analógico con pulsador, botón externo y dos indicadores LED) junto con la monitorización ambiental (sensor de temperatura y humedad DHT11) y de proximidad (sensor ultrasónico HC-SR04), estructurando su comportamiento en tres modos de operación: **Arranque**, **Servicio** y **Administración**.

---

## 2. Montaje Hardware y Esquemático

### 2.1. Componentes utilizados
Todos los elementos pertenecen exclusivamente al kit de laboratorio de la asignatura:
* Placa **Arduino UNO R3** (ATmega328P) y protoboard.
* **Pantalla LCD 16x2** (bus de 4 bits) y potenciómetro de 10 kΩ para control de contraste.
* **Módulo Joystick analógico** (ejes X, Y y botón integrado SW).
* **Sensor de Temperatura y Humedad DHT11**.
* **Sensor de Distancia por Ultrasonidos HC-SR04**.
* **Pulsador externo** para eventos de reinicio y acceso a Administración.
* **2 LEDs estándar** (`LED1` y `LED2`) con sus respectivas resistencias limitadoras de 220 Ω.

### 2.2. Asignación de pines y justificación electrónica

La elección de pines se ha planificado cuidadosamente para evitar conflictos de periféricos internos del microcontrolador ATmega328P:

| Componente | Pin del componente | Pin Arduino UNO | Justificación técnica |
| :--- | :--- | :--- | :--- |
| **Pulsador externo** | Terminal 1 / 2 | **Pin 2** / GND | Interrupción externa hardware `INT0` con `INPUT_PULLUP`. |
| **Joystick (Botón)** | SW | **Pin 3** | Lectura digital sin estado flotante mediante `INPUT_PULLUP`. |
| **Joystick (Ejes)** | VRx / VRy | **A0** / **A1** | Entradas al conversor ADC de 10 bits (rango 0–1023). |
| **Sensor DHT11** | DATA | **Pin 4** | Bus digital de trama de 40 bits calibrada. |
| **LED1 (Estado)** | Ánodo (+) | **Pin 5** | Salida digital (con resistencia de 220 Ω a GND). |
| **LED2 (Progreso)** | Ánodo (+) | **Pin 6 (PWM)** | Salida PWM gobernada por **Timer0** (980 Hz), evitando el conflicto con **Timer1** en los pines 9, 10 y 11. |
| **Pantalla LCD 16x2** | RS / E | **Pin 12** / **Pin 11** | Líneas de control del bus LCD (pin `RW` conectado a GND). |
| **Pantalla LCD 16x2** | DB4 / DB5 / DB6 / DB7 | **10 / 9 / 8 / 7** | Bus de datos en modo de 4 bits. |
| **HC-SR04** | TRIG / ECHO | **A2** / **A3** | Pines analógicos empleados como GPIO digitales para liberar los pines serie `0` y `1`. |
| **Semilla aleatoria** | *(Sin conectar)* | **A5** | Lectura de ruido analógico flotante para `randomSeed()`. |

### 2.3. Montaje en el simulador: CirkitDesigner

![Montaje en CirkitDesigner](img/esquematico.png)

### 2.4. Montaje real

 ![Vista general del montaje](img/montaje_general.jpg) 

---

## 3. Arquitectura Software

Para mantener el código limpio, legible y fácilmente reutilizable, la solución se ha modularizado en **9 archivos independientes**: cada periférico periódico se encapsula en su propia clase heredada de `Thread`, mientras que la configuración, el catálogo, las interrupciones y la máquina de estados se separan en ficheros de cabecera específicos sin emplear variables `extern`.

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

## 4. Recursos de Tiempo Real Utilizados: Justificación Técnica

### 4.1. Multitarea cooperativa con `ArduinoThread` (Por qué Threads frente a `delay()` y `millis()`)

En un microcontrolador AVR mononúcleo como el **ATmega328P** no existe paralelismo real en hardware (solo se ejecuta una instrucción por ciclo de reloj). Sin embargo, la máquina expendedora debe atender de forma concurrente tareas con frecuencias muy distintas: leer el joystick cada 80 ms, medir distancia cada 200 ms, parpadear el LED de arranque cada 1000 ms, consultar el DHT11 cada 1500 ms y actualizar la pantalla LCD y el PWM de manera fluida.

* **Por qué se descarta `delay()`:** Internamente, `delay()` ejecuta un bucle de espera activa (*busy-wait*) consultando `micros()` sin liberar la CPU del microcontrolador. Usar `delay(1000)` en el arranque o `delay(5000)` al mostrar temperatura/humedad impediría atender eventos del usuario, degradaría la reactividad del sistema e incluso podría provocar reinicios no deseados del Watchdog.
* **Por qué se emplea `ArduinoThread` en lugar de múltiples `if (millis() - prev >= T)` en `loop()`:** Gestionar manualmente múltiples contadores temporales en el bucle principal genera código monolítico, difícil de mantener y propenso a errores al escalar el número de periféricos.
* **Ventajas de heredar de `Thread` y planificar con `ThreadController`:**
  1. **Encapsulación y reutilización:** Cada periférico hereda públicamente de `Thread` en su propio `.h` (`LedThread`, `UltrasonidosThread`, `DHTThread`, `JoystickThread`), sobrescribiendo `shouldRun()` y `run()` (llamando siempre al padre `Thread::run()`).
  2. **Planificación cooperativa y bucle limpio:** `controller.run()` comprueba en cada iteración del `loop()` qué hilos han cumplido su intervalo y ejecuta únicamente los que correspondan. Al ser cooperativa, **si un hilo se quedase bloqueado, todo el programa se bloquearía**, lo cual refuerza la necesidad del Watchdog y de evitar bloqueos largos en los sensores.
  3. **Ahorro dinámico de recursos (`enabled`):** Los hilos pueden activarse o desactivarse en ejecución. En nuestro controlador, `led1Thread` se deshabilita (`led1Thread->enabled = false`) en cuanto concluyen los 3 parpadeos del Modo Arranque, y `led2Thread` permanece deshabilitado para no consumir CPU mientras se gobierna directamente mediante PWM o estado fijo.

| Hilo (`Thread`) | Intervalo | Justificación de la periodicidad |
| :--- | :---: | :--- |
| `LedThread` (`led1Thread`) | **1000 ms** | Cumple el intervalo exacto de 1 segundo de los 3 parpadeos del Modo Arranque; después se desactiva (`enabled = false`). |
| `UltrasonidosThread` | **200 ms** | 5 lecturas por segundo ofrecen detección inmediata de presencia (< 1 m) y telemetría fluida en Modo Admin sin generar ecos acústicos cruzados. |
| `DHTThread` | **1500 ms** | El DHT11 tiene una frecuencia máxima física cercana a 1 Hz; muestrearlo más rápido provocaría lecturas inválidas (`NaN`) y malgastaría tiempo en su protocolo serie. |
| `JoystickThread` | **80 ms** | 12.5 muestras por segundo garantizan una navegación ágil en menús sin sobrecargar el conversor ADC de 10 bits ($T_{\text{conv}} \approx 104\ \mu\text{s}$). |

---

### 4.2. Interrupciones Hardware (`INT0`) y Software (`TimerOne`)

#### Por qué interrupciones frente a *polling*
El sondeo continuo (*polling*) desperdicia ciclos de CPU comprobando el pin del pulsador cuando no hay actividad, carece de prioridad y puede perder pulsaciones o medir mal sus tiempos si coincide con la lectura de un sensor. Las interrupciones suspenden momentáneamente el flujo principal, salvan el contexto y el contador de programa, ejecutan la Rutina de Servicio de Interrupción (ISR) desde la tabla de vectores y devuelven el control al punto exacto donde estaba el programa.

#### Interrupción por Hardware (`INT0` en Pin 2, modo `CHANGE`)
* **Diseño ultrarrápido de la ISR:** Mientras se ejecuta una ISR en Arduino, las interrupciones globales están deshabilitadas, por lo que `millis()` no avanza y jamás deben usarse `delay()`, `Serial`, I2C ni SPI dentro de ella. Por ello, `isrBotonExterno()` se limita a leer el tiempo actual, aplicar el anti-rebote, registrar la marca de tiempo de pulsación y levantar banderas `volatile`.
* **Eliminación de rebotes (*debounce*):** Al cerrar o abrir un interruptor mecánico, los contactos oscilan durante varios milisegundos generando múltiples flancos falsos. Descartar cualquier transición producida a menos de `40 ms` de la anterior garantiza lecturas estables.
* **Uso estricto de `volatile`:** Solo se declaran `volatile` las variables compartidas entre las ISRs y el código principal (`contadorSegundos`, `tPresionBtnExt`, `tUltimoCambioISR`, `btnExtPresionado`, `flagCancelarServicio`, `flagToggleAdmin`). Esta directiva obliga al compilador a leer siempre su valor desde la memoria RAM y no desde registros internos cacheados, evitando ralentizar el resto de variables donde no es necesaria.

#### Interrupción por Software (`TimerOne`) y prevención de conflictos de Timers
El ATmega328P integra tres temporizadores hardware que se han repartido sin colisiones:
1. **`Timer0` (8 bits, prescaler 64, overflow cada $\approx 1.024\text{ ms}$):** Gestiona `millis()`, `micros()`, `delay()` y el PWM a **980 Hz** de los **pines 5 y 6**. Al conectar **`LED2` al Pin 6**, su modulación PWM durante la preparación del café es totalmente segura junto a `millis()` y ajena a `TimerOne`.
2. **`Timer1` (16 bits):** Reservado en exclusiva para **`TimerOne`**, configurado con un periodo de $1\,000\,000\ \mu\text{s}$ ($1\text{ s}$) para ejecutar `isrTimer1Segundo()` e incrementar `contadorSegundos` con exactitud. Como `TimerOne` interfiere con el PWM de los **pines 9, 10 y 11**, esos pines se destinan únicamente a señales digitales estándar del bus de la pantalla LCD (nunca a `analogWrite()`).
3. **`Timer2` (8 bits):** Queda libre sin interferencias.

---

### 4.3. Mecanismo de Seguridad: Watchdog Timer (`<avr/wdt.h>`)

El **Watchdog Timer (WDT)** es un temporizador independiente integrado en la arquitectura del microcontrolador con su **propio oscilador interno de 128 kHz**, completamente desacoplado del reloj principal de 16 MHz y de los timers `Timer0/1/2`. Si el programa no reinicia su cuenta mediante `wdt_reset()` antes de que venza el límite configurado, el Watchdog provoca un **reset físico automático del Arduino**.

#### Por qué se configura en `WDTO_4S` (4 segundos)
Al inicio de `setup()` se ejecuta `wdt_disable()` seguido de `wdt_enable(WDTO_4S)`, y en cada ciclo de `loop()` se invoca `wdt_reset()`.
* En el peor caso operativo —cuando coinciden en la misma vuelta el timeout del HC-SR04 (25 ms), una lectura del DHT11 (~20–25 ms), el muestreo del joystick y un borrado/escritura del LCD (~2–5 ms)— una iteración completa consume menos de **60 ms**.
* Un umbral de **4 segundos** proporciona un margen holgado para descartar falsos reinicios durante el funcionamiento normal, pero asegura una recuperación rápida si el microcontrolador sufre un bloqueo real.

#### ¿En qué partes del código SÍ influye y protege el Watchdog?
1. **En cada iteración completa de `loop()`:** Garantiza que el conjunto `controller.run()` + `actualizarMaquinaExpendedora()` termine siempre por debajo de 4 segundos y llegue a `wdt_reset()`.
2. **En todos los hilos de sensores (`UltrasonidosThread`, `DHTThread`, `JoystickThread`, `LedThread`):** Dado que `ArduinoThread` es cooperativo, si cualquier `run()` se quedara colgado, `controller.run()` nunca devolvería el control al `loop()` y el Watchdog resetearía la placa automáticamente.
3. **En el tramo de `setup()` posterior a `wdt_enable(WDTO_4S)`:** Si la inicialización de algún periférico (`lcd.begin()`, `dht.begin()`, etc.) bloqueara el arranque más de 4 segundos, el Watchdog forzaría un nuevo intento de arranque.
4. **En bloqueos dentro de ISRs o tormentas de interrupciones:** Incluso si el procesador quedase atrapado en una ISR o con las interrupciones globales deshabilitadas (`noInterrupts()`), el Watchdog opera por hardware de forma independiente y reseteará el sistema al cumplirse los 4 segundos.
5. **Ante corrupción del flujo de ejecución por interferencias eléctricas (EMI) o desbordamiento de pila:** Si el Contador de Programa saltase a una zona inválida de memoria y dejase de ejecutar el `loop()`, el oscilador de 128 kHz recuperaría la placa.

#### ¿En qué partes del código NO influye el Watchdog?
El Watchdog supervisa *disponibilidad temporal* (que el flujo no esté congelado), pero **no puede detectar**:
1. **Errores lógicos no bloqueantes en `MaquinaEstados.h`:** Si un error de programación impidiera salir de un estado (por ejemplo, una condición mal escrita en `SERV_ESPERA`), el `loop()` seguiría ejecutándose con normalidad y refrescando `wdt_reset()` miles de veces por segundo, por lo que el Watchdog consideraría el sistema sano.
2. **Sensores desconectados que fallan devolviendo error sin bloquearse:** Si se suelta el cable del DHT11 (`readTemperature()` retorna `NaN`) o el del HC-SR04 (`pulseIn()` agota su timeout de 25 ms y retorna `0`), el hilo termina en milisegundos, el `loop()` llega a `wdt_reset()` y el Watchdog no interviene.
3. **Durante el Bootloader y antes de `wdt_enable(WDTO_4S)`:** En el arranque previo a la activación del WDT, el temporizador no está vigilando.
4. **Fallos puramente físicos en actuadores o datos en RAM:** Un LED fundido, un mal contraste en `V0` del LCD o un precio mal configurado no detienen la CPU y quedan fuera del alcance del Watchdog.

#### ¿Dónde podría quedarse "pillado" exactamente nuestro código para que entre en acción el Watchdog?
Analizando el firmware completo, los puntos críticos donde un fallo eléctrico, de bus o de software podría congelar la CPU y disparar el Watchdog son:
1. **`UltrasonidosThread.h` → `pulseIn(pinEcho, HIGH, 25000)`:** Es una llamada intrínsecamente bloqueante a la espera del eco. Aunque en nuestro código se ha acotado con un timeout de `25000 us` para evitar bloqueos prolongados, cualquier fallo grave del pin `ECHO` sin timeout o bucles de reintento mal controlados detendrían aquí el flujo principal hasta disparar el Watchdog.
2. **`DHTThread.h` → `dht.readTemperature()` / `dht.readHumidity()`:** La librería de Adafruit lee la trama de 40 bits mediante bucles `while` cronometrados en microsegundos con secciones críticas donde desactiva interrupciones. Un fallo eléctrico severo en la línea de datos `PIN_DHT` que impidiera salir de esas esperas de sincronización haría que el Watchdog reiniciase la placa a los 4 segundos.
3. **Tormenta de interrupciones en `Interrupciones.h` (`isrBotonExterno`):** Si un cortocircuito intermitente o ruido eléctrico de alta frecuencia en el Pin 2 disparase `INT0` de manera incesante, la CPU no lograría volver al `loop()` para ejecutar `wdt_reset()`, activándose el reinicio de seguridad.
4. **Bloqueo accidental en `MaquinaEstados.h`:** Si alguna futura modificación introdujese un bucle de espera bloqueante (`while(...)`) que nunca cumpliese su condición de salida, transcurridos 4 segundos exactos el Watchdog reiniciaría el ATmega328P y devolvería la máquina automáticamente al **Modo Arranque** (`CARGANDO`).

---

## 5. Funcionamiento Detallado de los Modos de Operación

### 5.1. Diagrama de Estados del Controlador

```text
                          [Encendido / Reset / Watchdog]
                                        |
                                        v
                        +-------------------------------+
                        |        MODO ARRANQUE          |
                        |  LCD: "CARGANDO"              |
                        |  LED1: 3 parpadeos (1 s)      |
                        +---------------+---------------+
                                        | (6 conmutaciones de led1Thread)
                                        v
      +===================================================================+
      |                           MODO SERVICIO                           |
      |                                                                   |
      |  [SERV_ESPERA]  ---(HC-SR04 < 100 cm)--->  [SERV_ENTORNO (5 s)]   |
      |  "ESPERANDO CLIENTE"                       "Temp: XX C / Hum: YY%"|
      |        ^                                             |            |
      |        | (3 s)                                       v (5 s)      |
      |  [SERV_RETIRE]  <---(t = random 4..8s)---  [SERV_CATALOGO]        |
      |  "RETIRE BEBIDA"    [SERV_PREPARANDO]      Joystick Arr/Aba + SW  |
      |                     "Preparando Cafe..."             |            |
      |                     LED2: PWM 0 -> 255               v (Click)    |
      |                                            (inicia preparación)   |
      |                                                                   |
      |  * Botón Externo (2 a 3 s) en cualquier fase b -> vuelve a ESPERA |
      +===================================================================+
                       |                                 ^
        Botón Externo  | (>= 5 s)                        | Botón Externo (>= 5 s)
                       v                                 |
      +===================================================================+
      |                      MODO ADMIN (LED1=ON, LED2=ON)                |
      |                                                                   |
      |                       [ADM_MENU PRINCIPAL]                        |
      |                       (Joystick Arr / Aba)                        |
      |        +---------------+--------+-----------------+               |
      |        v (Click)       v        v                 v               |
      |   [ADM_TEMP]      [ADM_DIST] [ADM_CONT]    [ADM_SEL_PRECIO]       |
      |   DHT11 real      HC-SR04    Timer1 (s)           | (Click)       |
      |                                                   v               |
      |                                            [ADM_EDIT_PRECIO]      |
      |                                            +/- 0.05 € (Arr/Aba)   |
      |                                            Click: Guarda          |
      |   * Joystick IZQUIERDA en cualquier submenú -> retrocede 1 nivel  |
      +===================================================================+
```

---

## 6. Conclusiones

El desarrollo de esta práctica ha permitido aplicar de forma integrada los principios fundamentales de los sistemas empotrados y de tiempo real sobre un microcontrolador de recursos limitados como el **ATmega328P** de Arduino UNO:

* **Arquitectura modular y reutilizable:** Separar el proyecto en 9 ficheros independientes —encapsulando cada periférico periódico en su propia clase heredada de `Thread` (`LedThread`, `UltrasonidosThread`, `DHTThread`, `JoystickThread`) y manteniendo la configuración, el catálogo, las interrupciones y la máquina de estados en módulos estructurados sin recurrir a `extern`— ha dado como resultado un código claro, fácil de depurar y con un bucle principal (`loop()`) minimalista.
* **Reactividad sin retardos bloqueantes:** La planificación cooperativa mediante `ThreadController` y la gestión temporal con `millis()` eliminan por completo el uso de `delay()`, permitiendo atender de forma concurrente los sensores, el joystick, el refresco optimizado del LCD y la modulación PWM del `LED2` sin congelar la CPU, además de ahorrar recursos desactivando los hilos que ya no son necesarios (`enabled = false`).
* **Determinismo mediante interrupciones y gestión de timers:** El uso de la interrupción externa hardware `INT0` (Pin 2) con filtrado anti-rebote garantiza una discriminación precisa entre la pulsación de cancelación (2–3 s) y la de administración (≥ 5 s), mientras que la interrupción software de `TimerOne` proporciona un contador exacto segundo a segundo. Planificar el pinout evitando el conflicto entre `Timer1` y el PWM (asignando `LED2` al Pin 6, controlado por `Timer0`) resultó clave para que ambos subsistemas convivieran sin interferencias.
* **Tolerancia a fallos con Watchdog Timer:** Configurar el temporizador independiente Watchdog (`WDTO_4S`) aporta una capa real de seguridad hardware frente a bloqueos imprevistos en lecturas de sensores, líneas de datos o interrupciones anómalas, asegurando que la máquina expendedora pueda recuperarse automáticamente reiniciando en el Modo Arranque.
* **Validación sobre hardware real:** Contrastar la simulación con el montaje físico en protoboard evidenció detalles críticos del entorno real que no aparecen en software, como la calibración del contraste (`V0`) y puesta a masa de `RW` en la pantalla LCD 16x2, los rebotes mecánicos en los pulsadores o la necesidad de añadir una zona muerta en la lectura analógica del joystick para lograr una navegación precisa.

En conjunto, la solución cumple rigurosamente todas las especificaciones funcionales y técnicas de los tres modos de operación (**Arranque**, **Servicio** y **Administración**), dando lugar a un controlador autónomo robusto, reactivo y escalable.
