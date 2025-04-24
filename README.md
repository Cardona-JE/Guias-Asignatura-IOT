# Guía MQTT con ESP32, DHT11 y OLED en Windows

**Guía Técnica:** *Comunicación MQTT con ESP32, Sensor DHT11 y Pantalla OLED en Windows*

## Índice

1. [Materiales necesarios](#1-materiales-necesarios)  
2. [Configurar Arduino IDE en Windows](#2-configurar-arduino-ide-en-windows)  
3. [Instalar librerías necesarias](#3-instalar-librerías-necesarias)  
4. [Conexiones de hardware](#4-conexiones-de-hardware)  
5. [Paso 1: Probar conexión WiFi](#5-paso-1-probar-conexión-wifi)  
6. [Paso 2: Leer DHT11 y enviar a test.mosquitto.org](#6-paso-2-leer-dht11-y-enviar-a-testmosquittoorg)  
7. [Opcional: Crear un broker local en Windows](#7-opcional-crear-un-broker-local-en-windows)  
8. [Agregar pantalla OLED: explicación](#8-agregar-pantalla-oled-explicación)  
9. [Código final completo con OLED](#9-código-final-completo-con-oled)  
10. [Cuadro de resolución de problemas](#10-cuadro-de-resolución-de-problemas)

## 1. Materiales necesarios

- ESP32 (ESP-WROOM-32)
- Sensor DHT11 (temperatura y humedad)
- Pantalla OLED I2C (SSD1306 128×64)
- Cables jumper
- Protoboard (opcional)
- Cable USB para programar el ESP32
- PC con Windows 10/11

## 2. Configurar Arduino IDE en Windows

1. Descarga e instala Arduino IDE desde el siguiente [enlace](https://www.arduino.cc/en/software/).
2. Abre Arduino IDE y ve a `Archivo > Preferencias`.
En Gestor de URLs adicionales de tarjetas, añade:
`https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json`

> [!NOTE]  
> Este paso es opcional dado el caso de no encontrar los ESP32 en el gestor de boards

3. Ve a `Herramientas > Placa > Gestor de tarjetas...`, busca esp32 e instala `esp32 by Espressif Systems`.
4. Selecciona ***ESP32 Dev Module*** en `Herramientas > Placa > ESP32 Arduino > ESP32 Dev Module.`
5. En `Herramientas > Puerto`, elige el COM que corresponde al ESP32.

## 3. Instalar librerías necesarias

Desde `Programa > Incluir librería > Gestionar bibliotecas...`, instala:

- DHT sensor library (Adafruit)
- Adafruit Unified Sensor
- PubSubClient (Nick O'Leary)
- Adafruit SSD1306
- Adafruit GFX Library

## 4. Conexiones de hardware

### 4.1 Sensor DHT11

| DHT11 Pin | ESP32 Pin |
|-|-|
| VCC | 3.3 V |
| GND | GND |
| DATA | GPIO 4 |

### 4.2 Pantalla OLED SSD1306

| OLED Pin | ESP32 Pin |
|-|-|
| VCC | 3.3 V |
| GND | GND |
| SDA | GPIO 21 |
|SCL | GPIO 22 |

## 5. Paso 1: Probar conexión WiFi

**Objetivo:** *Confirmar que el ESP32 se conecta a tu red y obtener la IP asignada.*

### Código de prueba WiFi

```cpp
#include <WiFi.h>

auto ssid = "TU_SSID"; // Nombre de tu red WiFi
auto password = "TU_PASSWORD"; // Contraseña de tu red WiFi

void setup() {
    Serial.begin(115200);
    delay(1000);
    Serial.println("Conectando a WiFi...");
    WiFi.begin(ssid, password);
    // Espera hasta estar conectado
    while (WiFi.status() != WL_CONNECTED) {
        delay(500);
        Serial.print(".");
    }
    Serial.println("✅ Conectado a WiFi");
    Serial.print("📶 IP asignada: ");
    Serial.println(WiFi.localIP());
}

void loop() {
 // nada en el loop
}
```

1. Cambia `TU_SSID` y `TU_PASSWORD` por tus datos de red.
2. Sube el sketch al ESP32 usando **ESP32 Dev Module**.
3. Abre el **Monitor Serie** a 115200 bps.
4. Verifica que se muestre la IP asignada.

## 6. Paso 2: Leer DHT11 y enviar a test.mosquitto.org

**Objetivo:** *Capturar temperatura y humedad y publicarlos cada 5s en el broker público.*

### Explicación

- **PubSubClient** gestiona la conexión MQTT.
- Se leen datos con `dht.readTemperature()` y `dht.readHumidity()`.
- Se convierten los valores a cadena y se publican en tópicos separados.

### Código completo DHT11 + MQTT

```cpp
#include <WiFi.h>
#include <PubSubClient.h>
#include <DHT.h>
// --- Ajustes WiFi ---
const char* ssid = "TU_SSID";
const char* password = "TU_PASSWORD";
// --- Ajustes MQTT --
const char* mqtt_server = "test.mosquitto.org";
const int mqtt_port = 1883;
const char* topicTemp = "esp32/dht/temperature";
const char* topicHum = "esp32/dht/humidity";
// --- Sensor DHT11 --#define DHTPIN 4
#define DHTTYPE DHT11
DHT dht(DHTPIN, DHTTYPE);
// --- Clientes ---
WiFiClient espClient;
PubSubClient client(espClient);
void reconnect() {
    while (!client.connected()) {
        Serial.print("🔁 Conectando a MQTT...");
        if (client.connect("ESP32_DHT11")) {
            Serial.println("✅ Conectado");
        } else {
            Serial.print("❌ rc=");
            Serial.print(client.state());
            Serial.println(" reintentando en 5s");
            delay(5000);
        }
    }
}
void setup() {
    Serial.begin(115200);
    dht.begin();
    WiFi.begin(ssid, password);
    while (WiFi.status() != WL_CONNECTED) {
        delay(500);
        Serial.print(".");
    }
    Serial.println("\n✅ WiFi OK, IP: " + WiFi.localIP().toString());
    client.setServer(mqtt_server, mqtt_port);
}

void loop() {
    if (!client.connected()) reconnect();
    client.loop();
    float h = dht.readHumidity();
    float t = dht.readTemperature();
    if (isnan(h) || isnan(t)) return;
    char bufT[8], bufH[8];
    dtostrf(t, 1, 2, bufT);
    dtostrf(h, 1, 2, bufH);
    client.publish(topicTemp, bufT);
    client.publish(topicHum, bufH);
    Serial.printf("Temp: %s°C Hum: %s%%\n", bufT, bufH);
    delay(5000);
}

```

1. Sustituye tus credenciales WiFi.
2. Abre **MQTT Explorer** e introduce los datos de conexión:
    - **Host:** `test.mosquitto.org`
    - **Puerto:** `1883`
    - **Protocolo:** MQTT (TCP), sin TLS
    - Haz clic en **Connect**.
3. En el panel lateral, localiza el botón **+** o **Add new connection** para crear una nueva conexión si no existe.
4. Una vez conectado, en el árbol de topics, haz clic en **Subscribe to topic** y escribe `esp32/dht/#` para recibir tanto temperatura como humedad.
5. Observa cómo aparecen nuevos mensajes cada 5s mostrando valores de
temperatura y humedad.
6. Para ver los detalles de cada mensaje, selecciónalo y revisa el panel de
payload y timestamp.

## 7. Opcional: Crear un broker local en Windows

Este paso es necesario en el caso de estar usando una red publica, y es una
buena opcion si se desea tener el Broker en local.

1. Descarga el instalador de **Mosquitto** desde el siguiente [enlace](https://mosquitto.org/download/.)
2. Ejecuta el `.exe` y marca **Install as service**.
3. Abre `C:\Program Files\mosquitto\mosquitto.conf` y añade:

    >```conf
    >listener 1883
    >allow_anonymous true
    >```

4. Reinicia el servicio **Mosquitto** en **Servicios de Windows**.
5. Prueba en PowerShell:

    >```ps
    >mosquitto_sub -h localhost -t test -v &
    >mosquitto_pub -h localhost -t test -m "¡Broker local OK!"
    >```

Para usar este broker, modifica en el **Código DHT11 + MQTT**:

>```cpp
>const char* mqtt_server = "localhost"; // o la IP de tu PC
>```

## 8. Agregar pantalla OLED: explicación

Añadir una **OLED SSD1306** te permite mostrar en tiempo real:

- Temperatura
- Humedad
- Estado de conexión MQTT

Pasos:

1. Conecta la pantalla según sección 4.2.
2. Añade las librerías Adafruit SSD1306 y Adafruit GFX.
3. Inicializa el display en setup() .
4. Dibuja texto tras cada publicación MQTT.

## 9. Código final completo con OLED

```cpp
#include <WiFi.h>
#include <PubSubClient.h>
#include <DHT.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
// --- WiFi ---
const char* ssid = "TU_SSID";
const char* password = "TU_PASSWORD";
// --- MQTT --
const char* mqtt_server = "test.mosquitto.org";
const int mqtt_port = 1883;
const char* topicTemp = "esp32/dht/temperature";
const char* topicHum = "esp32/dht/humidity";
// --- DHT11 --#define DHTPIN 4
#define DHTTYPE DHT11
DHT dht(DHTPIN, DHTTYPE);
// --- OLED --#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);
// --- Cliente MQTT --WiFiClient espClient;
PubSubClient client(espClient);
void reconnect() {
    while (!client.connected()) {
        if (client.connect("ESP32_DHT11")) break;
        delay(5000);
    }
}
void setup() {
    Serial.begin(115200);
    dht.begin();
    // Inicializar OLED
    display.begin(SSD1306_SWITCHCAPVCC, 0x3C);
    display.clearDisplay();
    display.setTextSize(1);
    display.setTextColor(SSD1306_WHITE);
    display.setCursor(0, 0);
    display.println("Init...");
    display.display();
    // Conectar WiFi
    WiFi.begin(ssid, password);
    while (WiFi.status() != WL_CONNECTED) delay(500);
    // Configurar MQTT
    client.setServer(mqtt_server, mqtt_port)
}

void loop() {
    if (!client.connected()) reconnect();
    client.loop();
    float h = dht.readHumidity();
    float t = dht.readTemperature();
    if (isnan(h) || isnan(t)) return;
    char bufT[8], bufH[8];
    dtostrf(t, 1, 2, bufT);
    dtostrf(h, 1, 2, bufH);
    client.publish(topicTemp, bufT);
    client.publish(topicHum, bufH);
    // Mostrar en OLED
    display.clearDisplay();
    display.setCursor(0, 0);
    display.printf("Temp: %s C", bufT);
    display.setCursor(0, 10);
    display.printf("Hum: %s %%", bufH);
    display.setCursor(0, 20);
    display.println(client.connected() ? "MQTT: OK" : "MQTT: OFF");
    display.display();
    delay(5000);
}
```

## 10. Cuadro de resolución de problemas

| Problema | Posible causa | Posible solución |
|-|-|-|
| ESP32 no conecta a WiFi | SSID/contraseña incorrectos | Verifica los datos en el código |
|| Red es de 5 GHz | Usa una red de 2.4 GHz |
|| Señal débil |Acerca el ESP32 al router |
| No se publican datos en MQTT | IP incorrecta o broker no disponible | Asegúrate de que el broker esté activo |
|| MQTT Explorer mal configurado | Verifica host, puerto y tópicos |
| OLED no muestra nada | Dirección I2C incorrecta | Prueba con scanner I2C para verificar |
|| Librerías no instaladas | Asegúrate de tener Adafruit GFX y SSD1306|
| Error al compilar en Arduino IDE | Falta de librerías | Instálalas desde el gestor de bibliotecas |
|| Placa incorrecta seleccionada | Usa "ESP32 Dev Module" |
