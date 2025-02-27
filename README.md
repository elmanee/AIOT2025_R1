# AIOT2025_R1

## Parte Teórica

(Colocar aquí la información teórica relacionada con el proyecto, conceptos y fundamentos necesarios.)

## Parte Práctica

### Actividad 1

#### Código Fuente

```python
from machine import Pin, ADC
from time import sleep
from umqtt.simple import MQTTClient
import network

sensor = ADC(Pin(34))
led = Pin(2, Pin.OUT)  

sensor.atten(ADC.ATTN_11DB)  # Permite un rango de lectura de 0 a 3.3V
sensor.width(ADC.WIDTH_12BIT)  # Valores entre 0 y 4095

MQTT_Broker = "broker.emqx.io"
MQTT_USER = ""
MQTT_PASSWORD = ""
MQTT_CLIENT_ID = "esp32_client"
MQTT_TOPIC = "utng/arg/cmpm"
MQTT_PORT = 1883

# Función para conectar a la red WiFi
def wifi_connect():
    print("Conectando", end="")
    sta_if = network.WLAN(network.STA_IF)
    sta_if.active(True)
    try:
        sta_if.connect("INFINITUM0F58", "xDtf3kf5zs")
    except Exception as e:
        print("Error al conectar:", e)
        return
    timeout = 10
    while not sta_if.isconnected() and timeout:
        print(".", end="")
        sleep(1)
        timeout -= 1
    if sta_if.isconnected():
        print(" WiFi conectada")
    else:
        print(" Falló la conexión WiFi")

# Función para manejar la llegada de mensajes MQTT
def llegada_mensaje(topic, msg):
    print("Mensaje recibido:", msg)
    if msg == b'1':
        led.value(1)
    elif msg == b'0':
        led.value(0)

# Función para suscribirse al broker MQTT
def subscribir():
    client = MQTTClient(MQTT_CLIENT_ID, MQTT_Broker, port=MQTT_PORT, user=MQTT_USER, password=MQTT_PASSWORD, keepalive=0)
    client.set_callback(llegada_mensaje)
    client.connect()
    client.subscribe(MQTT_TOPIC)
    print("Conectado a %s, suscrito al tópico %s" % (MQTT_Broker, MQTT_TOPIC))
    return client

wifi_connect()
client = subscribir()

valor_anterior = sensor.read()
while True:
    client.check_msg()  # Revisión de mensajes entrantes
    nuevo_dato = sensor.read()
    if valor_anterior != nuevo_dato:
        client.publish(MQTT_TOPIC, str(nuevo_dato))
        valor_anterior = nuevo_dato
        led.value(1)
        sleep(0.5)
        led.value(0)
    sleep(1)
```

#### Imagen de Conexión

![image](https://github.com/user-attachments/assets/c397e837-4921-4b2d-8d51-fa5de6838bab)

#### Video

[Link al video y flujo](https://drive.google.com/drive/folders/1u0eyGvb6l0lOKrkla-z487WvvUNCj13C?usp=drive_link)  

### Actividad 2

#### Código Fuente

```python
from hcsr04 import HCSR04
from time import sleep
from machine import Pin

sensor = HCSR04(trigger_pin=5, echo_pin=18, echo_timeout_us=24000)

ledRojo = Pin(19, Pin.OUT)
ledAmarillo = Pin(4, Pin.OUT)
ledVerde = Pin(15, Pin.OUT)

while True:
    distancia = sensor.distance_cm()
    if distancia >= 0:
        print(distancia)
        if distancia > 30:
            ledRojo.value(0)
            ledAmarillo.value(0)
            ledVerde.value(1)
        elif distancia >= 15:
            ledRojo.value(0)
            ledAmarillo.value(1)
            ledVerde.value(0)
        else:
            ledRojo.value(1)
            ledAmarillo.value(0)
            ledVerde.value(0)
    else:
        print("Error de medición")

    sleep(1)
```
#### Video

[Link al video y flujo](https://drive.google.com/drive/folders/1u0eyGvb6l0lOKrkla-z487WvvUNCj13C?usp=drive_link)  


