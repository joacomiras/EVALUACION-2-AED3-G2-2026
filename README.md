# 🌡️ EVALUACIÓN DIGITALES 3: EL MISTERIO DEL 1-WIRE 👻

Buenas tardes queridos alumnos.

Después de sobrevivir a timers, GPIO, comunicaciones y algún que otro registro mal configurado, llega una nueva misión.

Esta vez deberán comunicarse con un dispositivo que aparentemente necesita muy poco para usarse...

**Un solo cable.**

Suena sencillo.

Obviamente, lo es.

Su misión será:

**Crear una librería para el STM32 capaz de comunicarse con un sensor DS18B20.** 👻

El protagonista de esta historia será el **DS18B20**, un sensor digital de temperatura capaz de medir y transmitir su información utilizando un protocolo llamado:

**1-Wire**

Como su nombre indica, toda la comunicación se realiza mediante **una única línea de datos**.

Y sí...

Por esa misma línea tendrán que transmitir, recibir, generar pulsos y respetar tiempos bastante específicos.

A continuación comienza la aventura.

---

## 🧞‍♂️ El Desafío: Crear el Driver

Su misión es desarrollar una librería en C para el **STM32F103C8** que permita comunicarse correctamente con un **DS18B20**.

La librería deberá ser capaz de:

- Inicializar el GPIO necesario.
- Detectar la presencia del sensor.
- Enviar bits.
- Recibir bits.
- Enviar bytes.
- Recibir bytes.
- Iniciar una conversión de temperatura.
- Leer la temperatura medida.
- Interpretar correctamente los datos entregados por el sensor.

La lógica de comunicación deberá desarrollarse a partir de la **hoja de datos del DS18B20**.

No se utilizará una librería externa que resuelva el protocolo.

---

# 👻 El Protocolo 1-Wire

El DS18B20 utiliza una única línea denominada:

```text
DQ
```

Por esta línea se realiza toda la comunicación entre:

```text
STM32
  ↕
DS18B20
```

No existe un pin separado para transmitir y otro para recibir.

Tampoco existe una señal de clock independiente.

En este protocolo, la información se interpreta principalmente según:

```text
ESTADO DE DQ
+
TIEMPO
```

Por lo tanto, los tiempos indicados en el datasheet son parte fundamental del protocolo.

---

## 🧟 DQ Tiene Dos Personalidades

La línea `DQ` es bidireccional.

En algunos momentos el STM32 necesita controlar la línea.

En otros momentos debe dejar de controlarla para permitir que el DS18B20 responda.

Por lo tanto, deberán decidir cuándo el STM32:

```text
CONTROLA DQ
```

y cuándo:

```text
LEE DQ
```

Esto implica modificar la configuración del GPIO según corresponda.

Presten especial atención a un concepto importante del protocolo:

**liberar la línea.**

Cuando el STM32 deja de mantener `DQ` en LOW, una resistencia externa de **pull-up** permite que la línea vuelva a HIGH.

Para esta evaluación se utilizará una resistencia de aproximadamente:

```text
4.7 kΩ
```

entre `DQ` y la alimentación.

Esto no les cambia mucho la parte codificada, pero no es una mención que saltarse.

A su vez, las configuraciones necesarias serán: 

```text
FLOATING INPUT: 0x4
```
```text
OPEN DRAIN OUTPUT: 0x5
```
Para esta última configuración tienen que tener en cuenta que solamente pueden forzar estados bajos, los estados altos aparecerán cuando el STM32 deje de conducir y la resistencia de pull-up lo lleve a ese punto. Entonces: 
```text
mandar 0:
STM32 fuerza DQ a LOW

mandar 1:
STM32 libera DQ
pull-up lleva DQ a HIGH
```

---

# 🕯️ RESET Y PRESENCE

Toda comunicación con el DS18B20 comienza con una secuencia de inicialización.

Primero el STM32 genera un:

```text
RESET
```

Luego libera la línea.

Si existe un DS18B20 conectado correctamente, el dispositivo responde mediante un:

```text
PRESENCE PULSE
```

Conceptualmente:

```text
STM32 genera RESET
        ↓
STM32 libera DQ
        ↓
DS18B20 responde
        ↓
STM32 detecta PRESENCE
```

Los tiempos exactos de cada etapa se encuentran indicados en el datasheet.

🧠 **Lean los diagramas temporales.**

Si leen la línea demasiado temprano o demasiado tarde, probablemente detecten basura en vez del sensor.

---

# 🕸️ Los Time Slots

El protocolo 1-Wire transmite:

```text
1 bit
```

por cada:

```text
TIME SLOT
```

Existen diferentes situaciones:

```text
WRITE 0
WRITE 1
READ
```

Todas comienzan con el STM32 llevando momentáneamente la línea:

```text
DQ → LOW
```

Lo que cambia es:

- cuánto tiempo permanece en LOW;
- cuándo se libera;
- cuándo debe leerse.

Esto significa que dos pulsos aparentemente muy parecidos pueden representar valores completamente diferentes.

---

## 🧙‍♂️ Escritura de un Bit

Para transmitir un bit, el STM32 comienza llevando `DQ` a LOW.

Luego deberá generar un comportamiento diferente según quiera transmitir:

```text
0
```

o:

```text
1
```

La diferencia se encuentra principalmente en **cuánto tiempo se mantiene la línea en LOW antes de liberarla**.

No se brindarán esos tiempos en este README.

Se encuentran claramente especificados en el datasheet.

---

## 🧛‍♂️ Lectura de un Bit

Para leer un bit, el STM32 también debe comenzar generando un pulso sobre `DQ`.

Pero después deberá:

```text
liberar la línea
        ↓
esperar el momento adecuado
        ↓
leer el GPIO
```

El DS18B20 decidirá durante ese intervalo si mantiene la línea en LOW o permite que permanezca en HIGH.

⚠️ **IMPORTANTE**

No pueden leer el pin en cualquier momento.

Existe una ventana determinada dentro del time slot donde el dato es válido.

Si leen fuera de esa ventana, el valor obtenido puede ser incorrecto.

---

# 🦇 De Bits a Bytes

Los comandos del DS18B20 poseen normalmente un tamaño de:

```text
8 bits
```

Por lo tanto necesitarán enviar y recibir bytes completos utilizando las operaciones de bit.

Una función para escribir un byte deberá conceptualmente realizar:

```text
obtener bit
    ↓
enviar bit
    ↓
obtener siguiente bit
    ↓
enviar bit
    ↓
...
```

Y para recibir:

```text
leer bit
    ↓
guardarlo en su posición
    ↓
leer siguiente bit
    ↓
...
    ↓
formar byte
```

Presten atención al orden en el que el protocolo transmite los bits.

No necesariamente comienza por el bit que ustedes elegirían primero.

👻 **La respuesta vuelve a estar en el datasheet.**

---

# 🧠 La Secuencia de Comunicación

El DS18B20 no acepta comandos en cualquier orden.

Toda transacción sigue conceptualmente una estructura:

```text
INICIALIZACIÓN
      ↓
COMANDO ROM
      ↓
COMANDO DE FUNCIÓN
      ↓
DATOS
```

Para esta evaluación se utilizará:

```text
UN SOLO DS18B20
```

conectado al bus.

Por lo tanto no será necesario implementar:

- búsqueda de dispositivos;
- identificación mediante número de serie;
- múltiples sensores;
- Search ROM.

Existe un comando ROM que permite continuar la comunicación sin enviar la dirección individual del dispositivo.

Encuéntrenlo en el datasheet.

---

# 🌡️ Convertir Temperatura

El DS18B20 no está midiendo continuamente para ustedes.

Para solicitar una nueva medición deberán enviar el comando correspondiente a:

```text
CONVERT T
```

Conceptualmente:

```text
RESET
   ↓
PRESENCE
   ↓
ROM COMMAND
   ↓
CONVERT T
   ↓
esperar conversión
```

La conversión de temperatura no es instantánea.

El tiempo máximo depende de la resolución utilizada por el sensor.

Para esta evaluación se trabajará con la configuración estándar del dispositivo.

Revisen el datasheet para determinar cuánto puede tardar una conversión.

---

# 📖 El Scratchpad

El DS18B20 almacena temporalmente información dentro de una memoria denominada:

```text
SCRATCHPAD
```

Allí se encuentra, entre otras cosas, la última temperatura medida.

Para leerla deberán realizar una nueva comunicación:

```text
RESET
   ↓
PRESENCE
   ↓
ROM COMMAND
   ↓
READ SCRATCHPAD
   ↓
DATOS
```

La temperatura ocupa **dos bytes** dentro del Scratchpad.

Deberán identificar:

```text
qué byte corresponde al LSB
```

y:

```text
qué byte corresponde al MSB
```

No es necesario leer todo el Scratchpad para esta evaluación.

---

# 🔮 El Misterio de la Temperatura

Los dos bytes obtenidos no representan directamente un número decimal como:

```text
25
```

El DS18B20 entrega la temperatura utilizando un formato binario con parte fraccionaria.

Por lo tanto deberán:

```text
leer LSB
    ↓
leer MSB
    ↓
unir ambos bytes
    ↓
interpretar el valor
    ↓
obtener temperatura en °C
```

La hoja de datos contiene ejemplos donde se muestra qué valores binarios corresponden a temperaturas como:

```text
+25.0625 °C
+0.5 °C
0 °C
-0.5 °C
-25.0625 °C
```

También deberán contemplar que la temperatura puede ser:

```text
POSITIVA
```

o:

```text
NEGATIVA
```

Presten atención a cómo el sensor representa números con signo.

---



# ⏱️ Funciones de Tiempo

El protocolo 1-Wire depende fuertemente de temporizaciones del orden de:

```text
microsegundos
```

Para evitar transformar esta evaluación en otra evaluación de timers, podrán disponer de funciones capaces de generar retardos en:

```c
delay_us(...);
delay_ms(...);
```

La tarea de ustedes será determinar:

**cuándo utilizarlas y durante cuánto tiempo.**

Los valores necesarios se encuentran en los diagramas temporales del DS18B20.

---

# 🧪 Funciones a Implementar — Los Hechizos

La librería deberá contener como mínimo las siguientes funciones:

```c
void ds18b20_init();
```

**Propósito:**  
Inicializar el GPIO y dejar la línea 1-Wire preparada para funcionar.

---

```c
uint8_t ds18b20_reset();
```

**Propósito:**  
Generar la secuencia de inicialización del protocolo y detectar si el DS18B20 respondió mediante un `presence pulse`.

La función deberá permitir determinar si existe un dispositivo conectado al bus.

---

```c
void ds18b20_write_bit(uint8_t bit);
```

**Propósito:**  
Transmitir un único bit mediante un time slot de escritura.

---

```c
uint8_t ds18b20_read_bit();
```

**Propósito:**  
Generar un time slot de lectura y devolver el bit entregado por el DS18B20.

---

```c
void ds18b20_write_byte(uint8_t data);
```

**Propósito:**  
Transmitir un byte completo utilizando el protocolo 1-Wire.

---

```c
uint8_t ds18b20_read_byte();
```

**Propósito:**  
Recibir un byte completo utilizando los bits obtenidos desde el bus.

---

```c
void ds18b20_start_conversion();
```

**Propósito:**  
Enviar la secuencia de comandos necesaria para indicarle al sensor que realice una nueva medición de temperatura.

---

```c
float ds18b20_get_temperature();
```

**Propósito:**  
Leer los bytes correspondientes a la temperatura desde el Scratchpad, interpretarlos correctamente y devolver:

```text
temperatura en °C
```

---

# 🦇 Funciones Recomendadas — No Son Obligatorias, Pero Les Van a Servir

Además de las funciones obligatorias, pueden separar algunas tareas en funciones auxiliares.

Por ejemplo:

```c
void dq_output();
void dq_input();

void dq_low();
uint8_t dq_read();
```

Estas funciones pueden ayudar a separar:

```text
configuración del GPIO
       ↓
control de DQ
       ↓
generación de time slots
       ↓
envío y recepción de bits
       ↓
envío y recepción de bytes
       ↓
comandos del sensor
```

⚠️ **IMPORTANTE**

Estas funciones:

- No son obligatorias.
- No tienen que llamarse de esta manera.
- No tienen que seguir exactamente esta estructura.
- No forman parte de lo que deberá utilizar directamente `main.c`.

Son simplemente una recomendación para organizar mejor la librería.

Cada grupo puede resolver internamente el problema de la manera que considere más adecuada mientras respete correctamente el protocolo indicado por el datasheet.

---

# 🧟 Estructura de la Librería

El proyecto deberá contener como mínimo:

```text
ds18b20.h
ds18b20.c
main.c
```

En:

```text
ds18b20.h
```

deberán encontrarse:

- Los `#define` necesarios.
- Los comandos utilizados.
- La configuración de pin y puerto.
- Los prototipos de las funciones públicas de la librería.

En:

```text
ds18b20.c
```

deberá desarrollarse toda la lógica correspondiente al protocolo 1-Wire.

El:

```text
main.c
```

deberá utilizar la librería para demostrar que el sensor funciona correctamente.

Por ejemplo, deberán poder observar desde el debugger una variable que contenga:

```text
temperatura
```

con el valor medido en grados Celsius.

---

# 👹 Forma de Entrega

Deberán sintetizar toda su magia en un repositorio de GitHub.

El repositorio deberá llamarse exactamente:

```text
APELLIDO_NOMBRE_EV_AED3G1
```

Deberán subir desde VSCode el **workspace completo del proyecto**, como vienen trabajando durante el año.

El repositorio deberá incluir correctamente:

```text
main.c
ds18b20.c
ds18b20.h
```

y todos los archivos necesarios para poder revisar el proyecto.

### ⚠️ ATENCIÓN

Si la forma de entrega, la estructura del proyecto o el nombre del repositorio no respetan lo solicitado, el alumno podrá ser penalizado con hasta:

**3 (tres) puntos menos.**

No quiero encontrar repositorios llamados:

```text
prueba
sensor temperatura
codigo final
ds18b20 ahora si
proyecto nuevo 2
```

Ya saben cómo funciona esto.

---

# 🍬 El Trato

Dentro del repositorio encontrarán:

- La hoja de datos del **DS18B20**.
- Las funciones de delay necesarias para trabajar con microsegundos.
- El manual de referencia del **STM32F103**.

Pero no habrá una librería que resuelva el protocolo.

Deberán interpretar:

- El protocolo 1-Wire.
- La secuencia de comunicación.
- Los diagramas temporales.
- Los time slots.
- El cambio de dirección del GPIO.
- El orden de los bits.
- Los comandos.
- El formato de la temperatura.

Y transformarlo en código.

La respuesta está en los manuales. 🔮


