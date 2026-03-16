
# BMP280 en ESP8266 (Wemos D1) con Arduino y MQTT

Proyecto de clase (IoT / meteorología) para **1º Bachillerato**: montamos una mini estación meteorológica con un ESP8266 y un sensor BMP280, y enviamos los datos por WiFi usando MQTT en formato JSON.

---

## Descripción

Este repositorio contiene un proyecto para **ESP8266 (NodeMCU / Wemos D1 mini)** programado con Arduino IDE. El ESP8266 lee un sensor barométrico **BMP280** por I2C (temperatura, presión y una altitud *estimada*) y envía esos datos a un **broker MQTT** cada 5 segundos en **JSON**.

Objetivo didáctico: entender un proyecto IoT completo de principio a fin:
- Hardware básico (cableado I2C).
- Conexión a WiFi.
- Mensajes MQTT (publicar/suscribirse con topics).
- Datos en JSON (estructura clara para luego visualizar en Node-RED).

---

## Funcionalidad (qué hace el programa)

- Se conecta a una red **WiFi** (si falla, vuelve a intentarlo).
- Se conecta a un **broker MQTT** (si se corta, reconecta).
- Publica un estado de conexión **Online/Offline** usando **LWT** (Last Will): así se puede saber desde fuera si el ESP “está vivo” o se ha caído. 
- Detecta el sensor BMP280 por I2C (dirección típica `0x76` o `0x77`) y lo configura.
- Cada `SEND_PERIOD_MS` (por defecto, 5000 ms):
  - Lee `temperatura_c`, `presion_hpa` y `altitud_m` (estimada).
  - Construye un JSON con información del ESP (IP, intensidad WiFi/RSSI, `millis()`) y valores del sensor.
  - Publica ese JSON en un **topic MQTT**.

---

## 1) Material necesario

- 1 × ESP8266 (NodeMCU o Wemos D1 mini)
- 1 × Sensor BMP280 (I2C)
- Cables Dupont (macho-macho o según tu módulo)
- Cable USB (micro-USB o USB-C según la placa)
- Driver CH340 (si tu placa lo necesita):  
  [Descargar aquí](https://sparks.gogo.co.nz/ch340.html)

**NOTA (importante):** si el PC no detecta el puerto COM o falla el driver, prueba a instalar el CH340 con la placa conectada y cambia de cable USB (algunos cables solo cargan y no sirven para datos).

---

## 2) Cableado (Wemos D1 / NodeMCU + BMP280 por I2C)

El bus I2C usa 2 cables de datos:
- **SCL** (reloj)
- **SDA** (datos)

En ESP8266 suele usarse:
- **D1 = GPIO5 = SCL**
- **D2 = GPIO4 = SDA** 

> Si tu placa es NodeMCU/Wemos, normalmente en la serigrafía ya aparece D1/D2.

### Conexiones (I2C)

![Cableado I2C Wemos D1 mini + BMP280](https://github.com/user-attachments/assets/03c42897-e598-430f-830e-7facc8c6fbce)

- Wemos **3V3** → BMP280 **VCC / VIN** (usa 3.3V)
- Wemos **G (GND)** → BMP280 **GND**
- Wemos **D1 (GPIO5 / SCL)** → BMP280 **SCL**
- Wemos **D2 (GPIO4 / SDA)** → BMP280 **SDA**

### Pines extra del BMP280 (si tu placa los tiene)

Muchos módulos BMP280 traen pines **CSB** y **SDO**:
- **SDO** cambia la dirección I2C: suele ser `0x76` cuando SDO está a GND y `0x77` cuando está a 3V3. 
- **CSB** se usa en SPI; para I2C normalmente se deja como está en el módulo (si tu módulo da problemas, hay módulos donde ayuda fijarlo a 3V3, según diseño).

---

## 3) Software y librerías

### 3.1 Arduino IDE y librerías
En Arduino IDE (Library Manager) instala:
- **Adafruit BMP280 Library**

  <img width="283" height="335" alt="Screenshot_1" src="https://github.com/user-attachments/assets/6b11d460-e529-4686-8a65-f4bb8497915e" />

- **ArduinoJson**
  
  <img width="275" height="345" alt="Screenshot_2" src="https://github.com/user-attachments/assets/9398869a-a109-44e6-b39c-2a28cfe0212e" />

- **PubSubClient**
  
<img width="276" height="316" alt="Screenshot_3" src="https://github.com/user-attachments/assets/8c7517c4-06b1-4b96-a5eb-c9abf4f67af7" />

### 3.2 Instalar soporte ESP8266 (Board Manager)
Para programar un ESP8266, Arduino IDE necesita el “core” del ESP8266.

1) Abre **Archivo / File → Preferencias / Preferences**.  
   (En esta ventana se configuran URLs de placas.)

<img width="891" height="615" alt="Preferencias Arduino IDE" src="https://github.com/user-attachments/assets/bf18bc3e-4550-4b64-a274-acbb32c2fe17" />

2) En “Additional Boards Manager URLs” pega esta URL:

```

http://arduino.esp8266.com/stable/package_esp8266com_index.json

```

<img width="809" height="550" alt="URL del core ESP8266" src="https://github.com/user-attachments/assets/8a6d64c7-042d-4f54-9f75-841802146e36" />

> Consejo: en GitHub, el icono de “dos cuadrados” permite copiar fácilmente el texto de un bloque.

3) Una vez hayas copiado el enlace y le hayas dado a aceptar, nos vamos a `Boards Manager` en el menú de la izquiera o en `Tools-Boards-Boards Manager` e instalamos el soporte para `ESP8266`

   <img width="279" height="320" alt="Screenshot_4" src="https://github.com/user-attachments/assets/fcc589ef-b762-4dc0-a14a-5e58897b98a4" />

---

## 4) Estructura del código (qué funciones importan)

En la carpeta `bmp280` abre `bmp280.ino`. Estas son las partes más importantes:

- `wifiConnect()`: conecta el ESP al WiFi (si falla, reintenta).
- `reconnectMQTT()`: conecta al broker MQTT, configura LWT y publica “Online”.
- `i2cScan()`: busca dispositivos I2C y muestra direcciones encontradas (sirve para comprobar cableado).
- `iniciaSensor()`: inicializa el BMP280 y configura el muestreo.
- `sendToBroker()`: lee el sensor → crea el JSON → publica por MQTT.
- `loop()`: mantiene WiFi/MQTT y envía datos cada cierto tiempo.

**Recomendación de clase:** lee primero `setup()` y `loop()` para entender el “mapa” general, y luego entra a las funciones.

---

## 5) Configuración rápida (valores que sí puedes cambiar)

En la parte de arriba del código verás constantes como estas (cámbialas por tus datos):

```cpp
#define WIFI_SSID     "TU_WIFI"
#define WIFI_PASSWORD "TU_PASS"

#define SET_SECURE_WIFI false

// Si SET_SECURE_WIFI == false:
#define MQTT_SERVER "136.112.103.14"
#define MQTT_PORT   1883
```

Qué significa cada una:

- `WIFI_SSID` y `WIFI_PASSWORD`: la WiFi a la que se conectará el ESP8266 (la del aula, tu móvil en modo hotspot, etc.).
- `MQTT_SERVER` y `MQTT_PORT`: el “servidor de mensajes” (broker MQTT) y su puerto.

Otros parámetros útiles:

- `TYPE_NODE`: etiqueta para organizar los topics (`meteorologia`, `aula1`, `grupo4`, etc.).
- `SEND_PERIOD_MS`: cada cuánto se envían datos (por defecto 5000 ms = 5 s).

**Importante para trabajos en grupo:** en `sendToBroker()` puedes cambiar el **topic de envío** para identificar mejor al alumno o al grupo (por ejemplo añadir `grupo4`, `mesa2`, `alumno_carla`). Así, al ver los datos en Node‑RED, sabréis de quién es cada sensor sin confusiones.

<img width="1083" height="732" alt="Cambiar topic de envío en sendToBroker" src="https://github.com/user-attachments/assets/e3a386e3-c3f0-42d9-b6ea-fbc342030d76" />

---

## 6) Node‑RED: montar tu primer flow para ver y enviar datos

<img title="" src="img/Logo%20de%20node-RED.png" alt="Logo de node-RED.png" width="147" data-align="center">

Vas a usar bloques o ***nodos*** interconectados en un flujo o ***flow*** para:

1. Recibir los datos que envía tu ESP8266 por MQTT.
2. Comprobar que esos datos llegan bien.
3. Extraer solo la parte que nos interesa, por ejemplo la temperatura.
4. Mostrar esos datos en una página web llamada **Dashboard**.
5. Enviar órdenes al ESP8266, por ejemplo para encender o apagar un LED.

Piensa en Node‑RED como una cadena de montaje:

![Diagrama cadena de montaje Node-RED](img/cadena_montaje.png)

- Un nodo **recibe** los datos.
- Otro nodo los **muestra** para revisarlos.
- Otro nodo los **transforma**.
- Otro nodo los **enseña en pantalla**.
- Y otro nodo puede **enviar órdenes** de vuelta al ESP8266.

### 6.1 ¿Qué es Node‑RED?

Node‑RED es una herramienta visual para crear programas uniendo bloques llamados **nodos** con líneas.  
Cada nodo hace una tarea concreta, y al unir varios nodos construimos un **flow**, es decir, un flujo de trabajo.

Una forma sencilla de entenderlo es pensar en piezas de LEGO:

![Idea sobre Node-RED](img/que_es_nodered.png)

- Cada pieza hace algo.
- Tú decides en qué orden colocarlas.
- Al final, todas juntas forman un sistema completo.

En este proyecto, Node‑RED será nuestro **panel de control** del ESP8266.

### 6.2 Antes de empezar

Antes de abrir Node‑RED, comprueba lo siguiente:

- El ESP8266 ya tiene MicroPython instalado.
- El programa del ESP8266 ya está funcionando.
- La placa ya se conecta al WiFi.
- La placa ya está enviando datos por MQTT.

Para eso, abre la consola de MicroPython (punto 5.6). Si en la consola del ESP8266 ves mensajes parecidos a estos:

```text
[wifi] connected
[mqtt] publish OK
```

entonces ya puedes seguir con este apartado.

### 6.3 Qué es un flow

Un **flow** es un conjunto de nodos conectados entre sí.

![Flow Node-RED](img/que_es_flow.png)

En esta práctica vamos a crear **dos flows**:

#### Flow 1: recibir y mostrar datos

![Flow 1 Node-RED](img/flow_1.png)

Este flow servirá para recibir los datos del sensor y mostrarlos en una dirección web.

```text
mqtt in  →  debug  →  function  →  ui_gauge / ui_text
```

#### Flow 2: enviar órdenes al ESP8266

![Flow 2 Node-RED](img/flow_2.png)

Este flow servirá para mandar órdenes desde Node‑RED a la placa.

```text
inject o ui_switch  →  mqtt out
```

### 6.4 Una pieza clave: Entendiedo el protocolo MQTT (Message Queuing Telemetry Transport)

![MQTT](img/MQTT_1.jpg)

![MQTT2](img/MQTT_2.jpg)

#### Qué es MQTT en esta práctica

![MQTT](img/diagrama_mqtt.png)

MQTT es un sistema de mensajería.

Los dispositivos envían y reciben mensajes usando “canales” llamados **topics**.

Por ejemplo:

- Un topic puede servir para enviar los datos del sensor.
- Otro topic distinto puede servir para enviar órdenes.

Cada topic es como un buzón:

- A un buzón de entrada llegan los datos del BMP280.
- De un buzón de salida salen órdenes para activar 0 desactivar algo (en este caso un LED).

Con Node‑RED puedes dirigirte al buzón indicado y recoger o dejar mensajes.

### 6.5 Abrir Node‑RED y reconocer la pantalla

Cuando abras Node‑RED verás tres zonas principales:

![noder-red](img/espacios_trabajo_nodered.png)

#### 1) Paleta de nodos

Suele estar en la parte izquierda.
Aquí aparecen los bloques que puedes arrastrar al espacio de trabajo.

Por ejemplo, verás nodos como:

- `inject`

![inject](img/inject.png)

- `debug`

![debug](img/debug_nodo.png)

- `function`

![function](img/function.png)

- `json`

![json](img/json.png)

- `mqtt in`

![mqttin](img/mqttin.png)

- `mqtt out`

![mqttout](img/mqttout.png)

- `ui_text`

![text](img/text.png)

- `ui_gauge`

![gauge](img/gauge.png)

- `ui_switch`

![switch](img/switch.png)


#### 2) Espacio de trabajo

Está en el centro.
Aquí es donde colocas los nodos y los conectas con líneas.

#### 3) Barra lateral derecha

Aquí aparecen varias pestañas con información y herramientas útiles.

La más importante al principio es la pestaña **Debug**, porque ahí verás los mensajes que recibe tu flow.  

![debug](img/debug.png)

Esa pestaña te ayudará a comprobar si los datos están llegando bien desde MQTT y a entender qué contenido tiene `msg.payload`.

Más adelante también usarás otras zonas de la interfaz, como la del **Dashboard**, pero al principio lo más importante es aprender a mirar el panel **Debug** para revisar qué está pasando.

#### 4) Resumen rápido para empezar bien

Antes de arrastrar nodos, haz esto:

1. Abre Node‑RED.
2. Crea un flow nuevo con el botón **`+`**.

![crearflow](img/add_flow.png)

Cuando lo tengas, vamos a configurarlo:

![configflow](img/flow_config.png)

1. Ponle un **nombre** claro.
2. Añade una **descripción** breve.

A partir de aquí, empieza a construir el flow en el espacio central. Recuerda usar la pestaña **Debug** para comprobar que todo funciona.

De esta forma trabajarás de manera más ordenada y te resultará mucho más fácil entender qué hace cada parte del proyecto.

### 6.6 Primer objetivo: comprobar que llegan datos

Antes de construir paneles bonitos, medidores o botones, lo primero es comprobar que los datos **de verdad están llegando**.

Para eso vamos a crear el flow más simple posible:

```text
mqtt in  →  debug
```

![mqttdebug](img/mqtt_debug.png)

Este primer flow no transforma ni muestra nada en el Dashboard. Solo sirve para asegurarnos de que Node‑RED está recibiendo mensajes del ESP8266.

### 6.7 Primer nodo: `mqtt in`

Arrastra un nodo `mqtt in` al espacio de trabajo.

Ahora haz doble clic sobre él para configurarlo.

![mqttconfig](img/mqttin_config.png)

#### Qué debes configurar

##### A) Server

Aquí debes indicar el servidor MQTT, también llamado **broker**.

El broker es el “repartidor” de mensajes.
Si el broker no es correcto, Node‑RED no recibirá nada.

##### B) Topic

Aquí debes escribir el topic donde publica tu ESP8266.

Ejemplo:

```text
cabrerapinto/meteorologia/ecfabca5f251/bmp280
```

Cada placa o cada grupo puede tener un topic diferente, así que copia exactamente el tuyo.

#### Consejo útil

Si al principio no recuerdas el topic exacto, a veces puedes usar un topic más general con `#`, por ejemplo:

```text
cabrerapinto/#
```

Eso significa: “escucha cualquier subtopic que empiece por `cabrerapinto/`”.

**Pero cuidado:** eso está bien para pruebas, pero cuando ya funcione, lo mejor es usar el topic exacto de tu ESP8266 para no mezclar tus mensajes con los de otros compañeros.

### 6.8 Segundo nodo: `debug`

Ahora arrastra un nodo `debug` y conéctalo a la salida del nodo `mqtt in`.

El nodo `debug` sirve para mostrar en la barra lateral derecha el contenido de los mensajes que recibe.

Es muy parecido a usar `print()` en Python:

- no cambia el mensaje,
- no lo arregla,
- solo te enseña qué está pasando.

#### Qué debes hacer ahora

1. Conecta `mqtt in` con `debug`.
2. Pulsa el botón **Deploy** arriba a la derecha.

![btndeploy](img/flow_basico.png)

4. Mira la pestaña **Debug** en la barra lateral.

![datos](img/debug_con_datos.png)

Si todo está bien, deberían empezar a aparecer mensajes.

**NOTA IMPORTANTE**: Cada vez que realicemos un cambio, tenemos que darle a **Deploy** para que todo funcione.

### 6.9 Qué aspecto puede tener el mensaje

Un mensaje típico puede ser parecido a este:

```json
{
  "esp": {
    "ip": "10.53.151.83",
    "rssi": -62
  },
  "sensor": {
    "bmp280": {
      "t_c": 22.45,
      "p_hpa": 1012.80
    }
  }
}
```

Vamos a entenderlo paso a paso.

#### Nivel 1: el mensaje completo

Tiene dos partes grandes:

- `esp`
- `sensor`

#### Nivel 2: la parte `esp`

Dentro de `esp` hay información sobre la placa, por ejemplo:

- `ip`
- `rssi`

#### Nivel 2: la parte `sensor`

Dentro de `sensor` hay otra parte llamada `bmp280`.

#### Nivel 3: la parte `bmp280`

Dentro de `bmp280` están los datos que nos interesan:

- `t_c` → temperatura en grados Celsius
- `p_hpa` → presión en hPa

### 6.10 Cómo leer el JSON sin perderse

Una forma fácil de entender el JSON es imaginar **cajas dentro de cajas**.

![jsonformat](img/json_format.png)

- Hay una caja grande llamada `sensor`.
- Dentro de esa caja hay otra caja llamada `bmp280`.
- Dentro de esa caja hay una etiqueta llamada `t_c`.

Por eso la ruta hasta la temperatura es:

```js
sensor.bmp280.t_c
```

Y como todo eso está dentro de `msg.payload`, en Node‑RED la ruta completa será:

```js
msg.payload.sensor.bmp280.t_c
```

### 6.11 Tercer nodo: `function` para extraer la temperatura

![flowfunction](img/flow_funcion.png)


Ahora vamos a crear un nodo `function` para quedarnos solo con la temperatura.

#### Paso 1: arrastrar el nodo

Arrastra un nodo `function` al espacio de trabajo.

#### Paso 2: conectarlo

Conéctalo después del `mqtt in`.

Puedes dejar también el `debug` conectado en paralelo para seguir viendo el mensaje completo.

Por ejemplo:

```text
mqtt in ──→ debug
   │
   └──→ function
```

#### Paso 3: abrir la configuración

Haz doble clic sobre el nodo `function`.

![funcionconfig](img/function_config.png)

Ponle un nombre, por ejemplo:

```text
Extraer temperatura
```

#### Paso 4: escribir el código

Escribe este código:

```js
var p = msg.payload;
msg.payload = p.sensor.bmp280.t_c;
return msg;
```

Ahora vamos a entenderlo **línea por línea**.

### 6.12 Explicación línea por línea del código de temperatura

#### Línea 1

```js
var p = msg.payload;
```

Esta línea significa:

> “voy a guardar el contenido de `msg.payload` en una variable llamada `p`”.

¿Por qué hacemos esto?

Porque `msg.payload` es más largo de escribir.
Si lo guardamos en `p`, el código queda más limpio y más fácil de leer.

Es como si dijéramos:

> “a partir de ahora, llamaré `p` a todo este paquete de datos”.

#### Línea 2

```js
msg.payload = p.sensor.bmp280.t_c;
```

Esta es la línea más importante.

Lo que hace es:

1. entrar en `p`,
2. buscar la parte `sensor`,
3. luego la parte `bmp280`,
4. luego la etiqueta `t_c`,
5. y colocar ese valor como nuevo `msg.payload`.

Si la temperatura era `22.45`, entonces después de esta línea `msg.payload` ya no será el JSON completo, sino solo:

```text
22.45
```

![debug_temp](img/debug_temp.png)

Es como si tuvieras una mochila llena de cosas y sacaras solo el **termómetro**, dejando lo demás aparte.

#### Línea 3

```js
return msg;
```

Esta línea significa:

> “ya he preparado el mensaje; ahora envíalo al siguiente nodo”.

Si olvidas esta línea, el mensaje no seguirá avanzando por el flow.

### 6.13 Qué hacer si aparece `undefined`

Si en vez de un número aparece `undefined`, significa que la ruta no coincide con el mensaje real.

![undefinded](img/undefinde.png)

Puede pasar porque:

- el mensaje no trae `sensor`,
- o no trae `bmp280`,
- o el nombre es distinto,
- o estás intentando acceder a un campo que no existe.

En ese caso, vuelve al nodo `debug` y mira bien cómo llega el JSON.

**Regla de oro:**
primero mirar el `debug`, después escribir la ruta.

### 6.14 Y si el mensaje llega como texto

A veces el mensaje MQTT no llega como un objeto ya organizado, sino como texto.

![mqttintxt](img/mqttin_texto.png)

Por ejemplo, podrías ver algo así:

```text
"{\"sensor\":{\"bmp280\":{\"t_c\":22.45}}}"
```

Eso significa que el JSON está “metido en una cadena de texto”.

Si te pasa eso, añade un nodo `json` entre `mqtt in` y `function`:

![str2json](img/json_format_flow.png)

```text
mqtt in  →  json  →  function
```

El nodo `json` convierte ese texto en un objeto que ya se puede recorrer por dentro.

#### Cómo saber si necesitas el nodo `json`

Por defecto, el nodo `mqtt in` recibe los mensajes y los formatea a JSON. Aquí te van algunas pistas para ver si es necesario o no:

- Si en `debug` ves un árbol desplegable con campos, normalmente no hace falta.
- Si ves una sola línea llena de comillas y barras `\`, entonces sí conviene usarlo.

### 6.15 Cómo crear y organizar el Dashboard

El **Dashboard** es la página web donde verás los datos de forma visual.  
Ahí puedes colocar textos, medidores, interruptores, botones y otros elementos.

Para organizarlo todo, el Dashboard usa **tres niveles** que debes crear **en este orden**:

![dashboard](img/DASHBOARD.png)

#### 1) Tab (Pestaña/Página)

Una **Tab** es como una pestaña de navegador. Cada Tab es una página distinta del Dashboard.

**Cómo crear una Tab:**

1. En Node‑RED, mira la **barra lateral derecha**.
2. Busca la pestaña **Dashboard** (icono de panel o monitor).

![irdashboard](img/ir_a_dashboard.png)

3. Pulsa el **+** junto a **"Tabs"**.

![creatab](img/crear_tab.png)

Vamos a configurar nuestro tablero:

![iraconfigtab](img/edit_tab.png)

4. Escribe el nombre, por ejemplo: `Mi estación meteorológica`.

![tabconfig](img/tab_config.png)

5. Pulsa **Update**.

**Ejemplos de Tabs útiles:**

- `Mi estación meteorológica` (página principal)
- `Sensores` 
- `Control`
- `Logs`

Para acceder a los tableros que se crean, en el Dashboard hay que darle click a las 3 lineas horizontales que hay en la esquina superior izquierda:

![accesotabs](img/Acceso_tabs.png)

#### 2) Group (Caja/Sección)

Un **Group** es una caja o bloque **dentro** de una Tab. Sirve para agrupar widgets relacionados.

**Cómo crear un Group:**

1. En la misma pestaña **Dashboard** (barra lateral derecha).
2. Pulsa el **+ Group** junto a tu Tab `Mi estación meteorológica`.

![addgroup](img/add_group.png)

3. Escribe el nombre del grupo, por ejemplo: `Estado`.

![groupconfig](img/group_config.png)

4. Selecciona el **With** (normalmente `col-6` o `col-12`). Esto es el ancho que quieres que ocupe tu grupo de objetos en el Dashboard.
5. Pulsa **Update**.

**Ejemplo de organización dentro de `ESP8266`:**

```
Tab: ESP8266
├── Group: Estado        ← IP, RSSI, Online/Offline
├── Group: BMP280       ← Temperatura, Presión
└── Group: Control LED  ← Interruptor del LED
```

#### 3) Widget (Elemento visual)

Un **Widget** es cada elemento que ves en la web: texto, medidor, botón, etc.

**Regla de oro: nunca asignes un widget sin antes tener Tab y Group.**

**Cómo configurar un widget (ejemplo con `ui_gauge`):**

![flowgauge](img/flow_con_gauge.png)

1. Arrastra `ui_gauge` al espacio de trabajo.

![arrastrogauge](img/arrastrar_uigauge.png)

2. Haz **doble clic** sobre él.

![gaugeconfig](img/ui_gauge_config.png)

3. En **Group**, **selecciona** el grupo que creaste (ej: `BMP280`).
4. **Label**: `Temperatura`
5. **Units**: `ºC`
6. **Min**: `0` / **Max**: `50`
7. Pulsa **Done**.

#### **Orden correcto para no perderse**

```
1️⃣ Crear Tab → 2️⃣ Crear Group → 3️⃣ Configurar Widget
```

**Si haces esto al revés, el widget NO aparecerá.**

#### **Ejemplo práctico completo**

**Tu primera Tab y Groups:**

```
Tab: ESP8266
├── Group: Estado (col-6)
│   ├── ui_text: IP
│   └── ui_text: RSSI
├── Group: BMP280 (col-6)
│   ├── ui_gauge: Temperatura
│   └── ui_gauge: Presión
└── Group: Control (col-12)
└── ui_switch: LED
```

#### **Cómo ver el Dashboard**

1. **Deploy** los cambios.
2. En la barra lateral derecha, pestaña **Dashboard**.
3. Puedes darle al icono que tiene forma de cuadrado con una flecha saliendo.

![iradashboard](img/ver_dashboard.png)

También puedes compiar la **URL** que aparece en Node-Red (algo como `http://[TU DIRECCIÓN IP]:1880/ui`).

4. Ábrela en una **pestaña nueva** del navegador.

#### **Errores típicos**

| Problema              | Solución                                             |
| --------------------- | ---------------------------------------------------- |
| **Widget no aparece** | No asignaste **Group** o el Group no tiene **Tab**   |
| **Todo en una línea** | Cambia el **layout** del Group a `col-6` o `col-12`  |
| **No se actualiza**   | Revisa que `msg.payload` sea un **número** (no JSON) |
| **URL no funciona**   | Comprueba que **Deploy** esté hecho                  |

#### **Consejo para clase**

**Crea siempre esta estructura base:**

```
Tab: ESP8266
├── Group: Estado
├── Group: Sensores
└── Group: Control
```

Y asigna **todos** tus widgets a uno de esos tres Groups. Así nunca te pierdes.

### 6.16 Mostrar la temperatura con `ui_gauge`

Una vez que el nodo `function` ya entrega solo un número, podemos mostrarlo en el Dashboard con un medidor.

Para eso vamos a usar el nodo `ui_gauge`.

#### Paso 1

Arrastra un nodo `ui_gauge`.

#### Paso 2

Conéctalo a la salida del nodo `function`.

#### Paso 3

Haz doble clic sobre él para configurarlo.

#### Paso 4

Configura estos campos:

- **Group**: el grupo donde aparecerá el medidor.
- **Label**: por ejemplo `Temperatura`.
- **Units**: por ejemplo `ºC`.
- **Min**: por ejemplo `0`.
- **Max**: por ejemplo `50`.

#### Qué debes entender

Este nodo necesita recibir un número.

Si le mandas un JSON completo, no sabrá qué hacer con él. Por eso antes usamos el nodo `function`: para dejar solo la temperatura.

### 6.17 Mostrar otros datos con `ui_text`

Además de la temperatura, a veces interesa enseñar datos como la IP o el RSSI.

![flowtext](img/flow_con_texto.png)

Para eso va muy bien `ui_text`.

#### Ejemplo: mostrar la IP

Primero crea un nodo `function` con este código:

```js
var p = msg.payload;
msg.payload = p.esp.ip;
return msg;
```

Luego conecta ese `function` a un nodo `ui_text`.

En `ui_text` puedes configurar:

- **Label**: `IP`
- **Format**:

```text
{{msg.payload}}
```

Así, en vez de ver un medidor, verás un texto en la web.

### 6.18 Extraer la presión

Ahora vamos a hacer exactamente lo mismo, pero con la presión.

Crea otro nodo `function` con este código:

```js
var p = msg.payload;
msg.payload = p.sensor.bmp280.p_hpa;
return msg;
```

La lógica es la misma que con la temperatura:

- entras en `sensor`,
- luego en `bmp280`,
- luego en `p_hpa`,
- y te quedas solo con la presión.

Después puedes conectarlo a otro `ui_gauge`.

#### Configuración sugerida del medidor de presión

- **Label**: `Presión`
- **Units**: `hPa`
- **Min**: `900`
- **Max**: `1100`

### 6.19 Montar el primer flow completo

Una versión sencilla del flow puede quedar así:

![flowcompleto](img/flow_con_gauge.png)

#### Para temperatura

```text
mqtt in  →  debug
   │
   └──→ function (temperatura)  →  ui_gauge
```

#### Para presión

```text
mqtt in  →  function (presión)  →  ui_gauge
```

#### Para IP

```text
mqtt in  →  function (ip)  →  ui_text
```

### 6.20 Organización recomendada del Dashboard

Para que no quede todo mezclado, puedes organizar el Dashboard así:

#### Tab

`Mi estación meteorológica`

#### Groups

- `Estado`
- `BMP280`
- `Control`

#### Dentro de `Estado`

- `ui_text` con la IP
- `ui_text` con el RSSI

#### Dentro de `BMP280`

- `ui_gauge` con la temperatura
- `ui_gauge` con la presión

#### Dentro de `Control`

- `ui_switch` para el LED

### 6.21 Ver el Dashboard

Cuando ya tengas los nodos colocados y configurados:

1. pulsa **Deploy**,
2. comprueba que los nodos MQTT aparecen como conectados,
3. abre la página del Dashboard.

Si todo está bien, deberías ver cómo los valores cambian conforme el ESP8266 va publicando datos.

### 6.22 Enviar órdenes al ESP8266

Una vez que ya sabes recibir datos, vamos a mandar una orden.

Para eso haremos un flow muy sencillo.

### 6.23 Opción A: usar `inject`

![outinject](img/out_con_inject.png)

El nodo `inject` sirve para lanzar un mensaje manualmente.

Es como un botón de prueba.

#### Qué debes hacer

1. Arrastra un nodo `inject`.
2. Haz doble clic sobre él.

![injectconfig](img/inject_opciones.png)

3. Configura el valor como `true` o `false`, según lo que espere tu programa.
4. Arrastra un nodo `mqtt out`.
5. Conecta `inject` con `mqtt out`.

![mqttoutconfig](img/mqttout_config.png)

7. En `mqtt out`, selecciona el Server correcto.
8. Escribe el topic de control del ESP8266.
9. Pulsa **Deploy**.

Ahora, cada vez que pulses el botón del nodo `inject`, estarás enviando una orden MQTT.

![injectbtn](img/inject_boton.png)

### 6.24 Opción B: usar `ui_switch`

![outswitch](img/out_con_switch.png)

Si quieres controlar la placa desde el Dashboard, usa `ui_switch`.

#### Qué debes hacer

1. Arrastra un nodo `ui_switch`.

![swchco](img/arrastro_switch.png)

2. Asígnalo a una **Tab** y un **Group**.

![swchconfig](img/switch_config.png)

3. Indicale qué tipo de datos quieres que envíe cuando lo pulses.
4. Conéctalo a un nodo `mqtt out`.
5. Configura `mqtt out` con el topic de control correcto.
6. Pulsa **Deploy**.

Ahora verás un interruptor en la web.

![switchdash](img/switch_en_dashboard.png)

Cuando lo cambies, Node‑RED enviará una orden al ESP8266.

### 6.25 Cómo revisar un flow sin perderse

Si algo no funciona, sigue siempre este orden:

1. **`mqtt in`**: comprueba que el broker y el topic están bien.
2. **`debug`**: mira si realmente llegan mensajes.
3. **`json`**: comprueba si hace falta convertir texto a objeto.
4. **`function`**: revisa si la ruta del dato es correcta.
5. **`ui_*`**: comprueba si el widget recibe el tipo de dato correcto.
6. **Dashboard**: revisa que Tab y Group estén bien asignados.

### 6.26 Errores típicos

#### No llega nada al `debug`

Revisa:

- el broker,
- el topic,
- y que el ESP8266 esté publicando datos.

#### Sale `undefined`

La ruta del dato es incorrecta.
Vuelve al `debug` y revisa la estructura del JSON.

#### El widget no aparece

Seguramente no has seleccionado bien la **Tab** o el **Group**.

#### El medidor no funciona

Probablemente no está recibiendo un número, sino un JSON completo o un texto.

#### No se enciende ni apaga el LED

Revisa el topic del `mqtt out` y el tipo de dato que estás enviando.

### 6.27 Objetivo final

Al terminar este apartado deberías tener:

- un flow que reciba los datos del ESP8266,
- un nodo `debug` para inspeccionarlos,
- uno o varios nodos `function` para extraer valores,
- un Dashboard con textos y medidores,
- y un control para enviar órdenes desde Node‑RED al ESP8266.

Si has llegado hasta aquí, ya has montado tu **primer sistema IoT visual**:
tu placa mide, publica, Node‑RED recibe, procesa y muestra los datos, y además puede enviar órdenes de vuelta.

---


## Actividades sugeridas (aula)

- Cambiar `SEND_PERIOD_MS` y observar carga de red/broker.
- Añadir una variable nueva en `values` (y verla en el suscriptor MQTT).
- Calibrar altitud: ajustar `SEALEVELPRESSURE_HPA` y comprobar cambios.
- Diseñar un “dashboard” (Node-RED / Home Assistant / Grafana) con el topic.

***

## Anexo: Cómo modificar sensores y enviar otros datos

La parte “educativa” del proyecto es que el alumnado pueda **cambiar el sensor** o **añadir más variables**. Estas son las zonas que deben tocar:

### 1) Añadir una nueva librería / objeto del sensor

- En la parte de `#include` (arriba).
- Crear una instancia global (igual que `Adafruit_BMP280 bmp;`).


### 2) Inicialización del sensor

- En `setup()` o en una nueva función tipo `iniciaSensorX()`.
- Patrón recomendado:
    - `i2cScan();` (si es I2C)
    - `sensor.begin(...)` y logs por Serial.


### 3) Lectura y publicación (la parte clave)

En `sendToBroker()`:

- Sustituir estas lecturas:

```cpp
float tempC   = bmp.readTemperature();
float pressHP = bmp.readPressure() / 100.0;
float altM    = bmp.readAltitude(SEALEVELPRESSURE_HPA);
```

- Y modificar el JSON en:

```cpp
JsonObject values = doc.createNestedObject("values");
values["temperatura_c"] = tempC;
values["presion_hpa"]   = pressHP;
values["altitud_m"]     = altM;
```


#### Ejemplo: añadir “luz” (LDR analógico) o “humedad”

- Lees el nuevo dato (p.ej. `int luz = analogRead(A0);`)
- Lo añades al JSON:

```cpp
values["luz_raw"] = luz;
```


### 4) Cambiar el topic de publicación (opcional)

También en `sendToBroker()`:

```cpp
String pub_topic = "orchard/" + TYPE_NODE + "/" + String(ESP.getChipId(), HEX) + "/bmp280";
```

Así cada sensor/proyecto queda bien organizado.

***

## Resolución de problemas

- Si `i2cScan()` no encuentra `0x76` o `0x77`:
    - Revisa SDA/SCL, alimentación 3.3V y GND común.
- Si MQTT conecta pero no llegan datos:
    - Revisa `MQTT_SERVER`, puerto, y que el broker esté accesible desde la WiFi.
- Si el JSON es grande y falla `publish()`:
    - Asegura `client.setBufferSize(1024);` y reduce campos si hace falta.

***
