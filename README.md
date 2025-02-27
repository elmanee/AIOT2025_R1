# AIOT2025_R1

## Parte Teórica
Basado en **NetAcad Python Fundamentals 2**:

- **Capítulo 1**  
  ![image](https://github.com/user-attachments/assets/034afded-84a2-4912-b2e6-4f6971f1f130)

- **Capítulo 2**  
  ![image](https://github.com/user-attachments/assets/916d5ec7-b405-45d1-8078-2585e2da3f52)

- **Capítulo 3**  
  ![image](https://github.com/user-attachments/assets/aad7a913-9b64-484a-abdb-d7b2f5e342b2)

- **Capítulo 4**  
  ![image](https://github.com/user-attachments/assets/a7836531-9a34-486c-bf66-a91cd210f93d)

- **Prueba Final del curso Fundamentos de Python 2 (PE2)**  
  ![image](https://github.com/user-attachments/assets/b84129a4-ee14-4ebe-8f98-17dbb58c949c)
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

### Libreria HCSR04
```python
from machine import Pin, time_pulse_us
from utime import sleep_us

__version__ = '0.2.1'
__author__ = 'Roberto Sánchez'
__license__ = "Apache License 2.0. https://www.apache.org/licenses/LICENSE-2.0"

class HCSR04:
    """
    Driver to use the untrasonic sensor HC-SR04.
    The sensor range is between 2cm and 4m.

    The timeouts received listening to echo pin are converted to OSError('Out of range')

    """
    # echo_timeout_us is based in chip range limit (400cm)
    def __init__(self, trigger_pin, echo_pin, echo_timeout_us=500*2*30):
        """
        trigger_pin: Output pin to send pulses
        echo_pin: Readonly pin to measure the distance. The pin should be protected with 1k resistor
        echo_timeout_us: Timeout in microseconds to listen to echo pin. 
        By default is based in sensor limit range (4m)
        """
        self.echo_timeout_us = echo_timeout_us
        # Init trigger pin (out)
        self.trigger = Pin(trigger_pin, mode=Pin.OUT, pull=None)
        self.trigger.value(0)

        # Init echo pin (in)
        self.echo = Pin(echo_pin, mode=Pin.IN, pull=None)

    def _send_pulse_and_wait(self):
        """
        Send the pulse to trigger and listen on echo pin.
        We use the method `machine.time_pulse_us()` to get the microseconds until the echo is received.
        """
        self.trigger.value(0) # Stabilize the sensor
        sleep_us(5)
        self.trigger.value(1)
        # Send a 10us pulse.
        sleep_us(10)
        self.trigger.value(0)
        try:
            pulse_time = time_pulse_us(self.echo, 1, self.echo_timeout_us)
            # time_pulse_us returns -2 if there was timeout waiting for condition; and -1 if there was timeout during the main measurement. It DOES NOT raise an exception
            # ...as of MicroPython 1.17: http://docs.micropython.org/en/v1.17/library/machine.html#machine.time_pulse_us
            if pulse_time < 0:
                MAX_RANGE_IN_CM = const(500) # it's really ~400 but I've read people say they see it working up to ~460
                pulse_time = int(MAX_RANGE_IN_CM * 29.1) # 1cm each 29.1us
            return pulse_time
        except OSError as ex:
            if ex.args[0] == 110: # 110 = ETIMEDOUT
                raise OSError('Out of range')
            raise ex

    def distance_mm(self):
        """
        Get the distance in milimeters without floating point operations.
        """
        pulse_time = self._send_pulse_and_wait()

        # To calculate the distance we get the pulse_time and divide it by 2 
        # (the pulse walk the distance twice) and by 29.1 becasue
        # the sound speed on air (343.2 m/s), that It's equivalent to
        # 0.34320 mm/us that is 1mm each 2.91us
        # pulse_time // 2 // 2.91 -> pulse_time // 5.82 -> pulse_time * 100 // 582 
        mm = pulse_time * 100 // 582
        return mm

    def distance_cm(self):
        """
        Get the distance in centimeters with floating point operations.
        It returns a float
        """
        pulse_time = self._send_pulse_and_wait()

        # To calculate the distance we get the pulse_time and divide it by 2 
        # (the pulse walk the distance twice) and by 29.1 becasue
        # the sound speed on air (343.2 m/s), that It's equivalent to
        # 0.034320 cm/us that is 1cm each 29.1us
        cms = (pulse_time / 2) / 29.1
        return cms


```

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
### Imagen
![Imagen de WhatsApp 2025-02-27 a las 03 46 58_93c77da4](https://github.com/user-attachments/assets/94c3dbe4-d906-4670-ac6f-2e5454394cf3)

#### Video
[Link al video y flujo](https://drive.google.com/drive/folders/1oocsk-Ek0dTLc7A9W3XLpgc5RM9qkoKE?usp=drive_link)  


