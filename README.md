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

| ![Vista general del montaje](img/montaje_general.jpg) |

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
