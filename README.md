# INSTRUMENTO DE RECUPERACIÓN 2 (R2)
# Curso JavaScript Essentials 2 de Cisco NetAcad.
Capturas 
<img width="959" alt="netacat5" src="https://github.com/user-attachments/assets/80553424-4489-462e-8dbb-9a8eb4a77d8c" />
<img width="954" alt="netacat1" src="https://github.com/user-attachments/assets/254697c8-3045-40b8-8d98-25be69a95e54" />
<img width="959" alt="netacat2" src="https://github.com/user-attachments/assets/24087ddb-5086-4414-85ea-be6109a23249" />
<img width="954" alt="netacat3" src="https://github.com/user-attachments/assets/103882f2-30da-4ee4-b321-e615bb62470a" />
<img width="952" alt="netacat4" src="https://github.com/user-attachments/assets/160129cd-3da1-4507-ad83-00b3e181db13" />

# Ejercicio 1: Comunicación Serial con Conversión de Datos
En lugar de solo enviar datos, el ESP32 deberá convertir la señal analógica de
un sensor en datos legibles en la PC.
1. Implementa una comunicación serial donde el ESP32 convierta la señal analógica de
un sensor en datos de temperatura o humedad y los envíe a la PC.

# Codigo:
```python
from machine import ADC, Pin
from time import sleep

# Configuramos el pin 34 como entrada analógica
sensor = ADC(Pin(34))
sensor.atten(ADC.ATTN_11DB)   # Para medir hasta 3.3V
sensor.width(ADC.WIDTH_10BIT)  # Rango de 0-1023

while True:
    valor = sensor.read()  # Leemos el valor ADC (0-1023)
    voltaje = valor * (3.3 / 1023.0)  # Convertimos a voltaje
    temperatura = voltaje * 100  # 10mV = 1°C para el LM35

    print("Temperatura: {:.2f} °C".format(temperatura))
    sleep(1)
```

# Diagrama de conexion:
<img width="195" alt="Captura de pantalla 2025-05-06 155011" src="https://github.com/user-attachments/assets/d862c9d3-3ac3-4bf1-8930-80facfcbb69b" />

# Video Demostrativo:

https://drive.google.com/file/d/1HalKD40sVr0boueGuoXZstCGtutvL9He/view?usp=sharing

////////////////////////////////////////////////////////////////////////////

# Ejercicio 2: Comunicación Bluetooth con Control desde un Chatbot
En lugar de encender un LED, se usará un chatbot en Telegram o WhatsApp
para controlar el Bluetooth.
1. Configura un bot que, al recibir un mensaje en Telegram o WhatsApp, envíe
comandos a un ESP32 para encender/apagar un LED o mover un servo.
# Codigo:

# servidor.py 
```python

import asyncio
import telebot
from bleak import BleakClient, BleakScanner
import threading


TOKEN = '7616505920:AAH-08P7A_5fP0W5_1lbq2NmlL1PdHcZkgU'
bot = telebot.TeleBot(TOKEN)

UART_SERVICE_UUID = "6E400001-B5A3-F393-E0A9-E50E24DCCA9E"
UART_RX_CHAR_UUID = "6E400002-B5A3-F393-E0A9-E50E24DCCA9E"  # Para escribir en ESP32
UART_TX_CHAR_UUID = "6E400003-B5A3-F393-E0A9-E50E24DCCA9E"  # Para leer de ESP32


ble_client = None
device_name = "ESP32-BLE"  # Debe coincidir con el nombre en tu código ESP32


reconnect_loop_running = True

def notification_handler(sender, data):
    message = data.decode()
    print(f"Mensaje recibido del ESP32: {message}")
    # Aquí puedes procesar los mensajes recibidos del ESP32


async def connect_ble():
    global ble_client
    
    print(f"Buscando dispositivo BLE: {device_name}...")
    
    try:
        # Buscar dispositivo por nombre
        device = await BleakScanner.find_device_by_name(device_name)
        
        if not device:
            print(f"No se encontró el dispositivo: {device_name}")
            return False
        
        print(f"Dispositivo encontrado. Intentando conectar a {device.name} [{device.address}]")
        
        # Conectar al dispositivo
        client = BleakClient(device.address)
        await client.connect()
        
        if client.is_connected:
            print(f"Conectado a {device.name}")
            
            # Configurar para recibir notificaciones
            await client.start_notify(UART_TX_CHAR_UUID, notification_handler)
            
            ble_client = client
            return True
        else:
            print("No se pudo conectar al dispositivo")
            return False
            
    except Exception as e:
        print(f"Error al conectar: {e}")
        return False


async def disconnect_ble():
    global ble_client
    if ble_client and ble_client.is_connected:
        await ble_client.disconnect()
        print("Desconectado del dispositivo BLE")


async def send_command(command):
    global ble_client
    
    if not ble_client or not ble_client.is_connected:
        print("No hay conexión BLE activa")
        return False
    
    try:
        # Enviar comando a la característica RX
        await ble_client.write_gatt_char(UART_RX_CHAR_UUID, command.encode())
        print(f"Comando enviado: {command}")
        return True
    except Exception as e:
        print(f"Error al enviar comando: {e}")
        return False


async def reconnection_loop():
    global ble_client, reconnect_loop_running
    
    while reconnect_loop_running:
        if not ble_client or not ble_client.is_connected:
            print("Intentando reconectar...")
            await connect_ble()
        await asyncio.sleep(5)  # Revisar cada 5 segundos


def run_ble_loop():
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)
    
    # Conectar por primera vez
    loop.run_until_complete(connect_ble())
    
    # Iniciar bucle de reconexión
    loop.run_until_complete(reconnection_loop())


ble_thread = threading.Thread(target=run_ble_loop)
ble_thread.daemon = True
ble_thread.start()

@bot.message_handler(commands=['start'])
def send_welcome(message):
    bot.reply_to(message, "Hola! Envíame comandos como SERVO:90 para mover el servo a 90 grados.")

@bot.message_handler(commands=['status'])
def check_status(message):
    if ble_client and ble_client.is_connected:
        bot.reply_to(message, "✅ Conectado al ESP32")
    else:
        bot.reply_to(message, "❌ No hay conexión con el ESP32")

@bot.message_handler(commands=['reconnect'])
def force_reconnect(message):
    bot.reply_to(message, "Intentando reconectar con el ESP32...")
    # La reconexión se maneja automáticamente en el bucle de reconexión

@bot.message_handler(func=lambda message: True)
def handle_message(message):
    command = message.text.strip().upper()
    
    # Validar si el comando es tipo SERVO:90
    if command.startswith("SERVO:"):
        try:
            angle = int(command.split(":")[1])
            if 0 <= angle <= 180:
                # En lugar de usar bluetooth.write, usamos asyncio para enviar el comando BLE
                loop = asyncio.new_event_loop()
                success = loop.run_until_complete(send_command(command))
                
                if success:
                    bot.reply_to(message, f"Ángulo enviado: {angle}°")
                else:
                    bot.reply_to(message, "No se pudo enviar el comando. Verifica la conexión BLE.")
            else:
                bot.reply_to(message, "El ángulo debe estar entre 0 y 180.")
        except ValueError:
            bot.reply_to(message, "Formato incorrecto. Usa SERVO:90, SERVO:45, etc.")
    else:
        bot.reply_to(message, "Comando no válido. Usa SERVO:[0-180]")

try:
    print("Iniciando bot de Telegram...")
    bot.polling()
except Exception as e:
    print(f"Error al iniciar el bot: {e}")
finally:
    # Detener el bucle de reconexión y desconectar BLE
    reconnect_loop_running = False
    loop = asyncio.new_event_loop()
    loop.run_until_complete(disconnect_ble())
    
    input("Presiona Enter para salir...")
```

# esp32
# Control de Servo por Bluetooth BLE para ESP32 en MicroPython
```python

import bluetooth
from ble_simple_peripheral import BLESimplePeripheral
import time
from machine import Pin, PWM


servo_pin = 13  # Cambia al pin donde conectaste tu servo
servo = PWM(Pin(servo_pin), freq=50)  # Frecuencia estándar para servos


def set_servo_angle(angle):
    # Convertir de 0-180 grados a duty cycle (aprox. 40-115)
    # Los valores exactos pueden variar según el servo
    min_duty = 40
    max_duty = 115
    duty = min_duty + (max_duty - min_duty) * (angle / 180)
    servo.duty(int(duty))
    return angle


ble = bluetooth.BLE()

sp = BLESimplePeripheral(ble)


led = Pin(2, Pin.OUT)  # El pin puede variar según tu placa ESP32

def on_rx(data):
    """Función que se ejecuta cuando se reciben datos"""
    data_str = data.decode().strip()
    print("Datos recibidos:", data_str)
    
    # Procesar comando para el servo
    if data_str.startswith("SERVO:"):
        try:
            # Extraer el ángulo del comando (SERVO:90)
            angle = int(data_str.split(":")[1])
            if 0 <= angle <= 180:
                # Mover el servo al ángulo especificado
                actual_angle = set_servo_angle(angle)
                
                # Parpadear LED para indicar que se recibió comando
                led.value(1)
                time.sleep(0.1)
                led.value(0)
                
                # Enviar confirmación
                if sp.is_connected():
                    sp.send(f"Servo movido a: {actual_angle} grados")
            else:
                if sp.is_connected():
                    sp.send("Error: El ángulo debe estar entre 0 y 180")
        except Exception as e:
            if sp.is_connected():
                sp.send(f"Error: {str(e)}")


set_servo_angle(90)


print("Esperando conexión Bluetooth...")
while True:
    # Si está conectado, el método 'advertise' no hace nada
    if not sp.is_connected():
        # Si no está conectado, comenzar a anunciarse de nuevo
        sp.advertise()
        print("Esperando conexión...")
    
    # Revisar si se han recibido datos
    if sp.is_connected():
        # El callback 'on_rx' manejará los datos recibidos
        sp.on_write(on_rx)
        
        # Enviar un mensaje cada 3 segundos
        sp.send(f"ESP32 Servo Control - {time.ticks_ms()}")
    
    # Esperar un poco antes de la siguiente iteración
    time.sleep(3)
```

# Diagrama de conexion:
![image](https://github.com/user-attachments/assets/b3899046-a898-4fcd-82e3-e4e09de723ca)

# Video Demostrativo:

https://drive.google.com/file/d/1RHtFUrnR_70dppRgao8QZZ1UtVCI0HhU/view?usp=sharing

////////////////////////////////////////////////////////////////////////////

# Ejercicio 3: Comunicación TCP/IP con Base de Datos:
En lugar de solo enviar datos, estos deben almacenarse en una base de datos
(SQLite, Firebase, PostgreSQL).
1. Implementa una comunicación TCP/IP donde un ESP32 registre datos en una base
de datos en la nube o local.

# Codigo:

# Esp32
```python
from machine import ADC, Pin
import network
import socket
import time

# Conexión Wi-Fi
ssid = 'INFINITUMF116_EXT'
password = 'X4s9xzFrsx'

sta = network.WLAN(network.STA_IF)
sta.active(True)
sta.connect(ssid, password)

while not sta.isconnected():
    print('Conectando...')
    time.sleep(1)

print('Conectado:', sta.ifconfig())

# Sensor LM35 en GPIO34 (puedes usar otro pin ADC)
sensor = ADC(Pin(34))  # Cambia a 32, 33, 35 si prefieres
sensor.atten(ADC.ATTN_11DB)  # Para leer hasta 3.6V

# Envío TCP
while True:
    try:
        s = socket.socket()
        s.connect(('192.168.1.201', 3000))  # IP del servidor
        valor = sensor.read()  # 0 - 4095

        # Convertir la lectura ADC (0-4095) a voltaje (0-3.3V)
        voltaje = (valor / 4095) * 3.3

        # Convertir voltaje a temperatura en Celsius (LM35: 1°C = 10mV)
        temperatura = voltaje * 100

        mensaje = f"temperatura={temperatura:.2f}"  # Formato: temperatura=25.45
        s.send(mensaje.encode())
        print("Enviado:", mensaje)
        s.close()
    except Exception as e:
        print("Error al enviar:", e)
    time.sleep(5)


```
# server.js
```js
const net = require('net');
const sqlite3 = require('sqlite3').verbose();
const db = new sqlite3.Database('./datos.db');

// Crear tabla si no existe
db.run(`CREATE TABLE IF NOT EXISTS lecturas (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  valor TEXT,
  timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
)`);

// Servidor TCP
const server = net.createServer((socket) => {
    console.log('Cliente conectado');

    socket.on('data', (data) => {
        // Guardar solo el valor de temperatura
        const recibido = data.toString().trim(); // Ejemplo de formato: "temperatura=25.45"
        const valor = recibido.split('=')[1]; // Extrae el valor numérico
        db.run('INSERT INTO lecturas (valor) VALUES (?)', [valor], (err) => {
            if (err) {
                return console.error(err.message);
            }
            console.log('Dato guardado en SQLite');
        });
    });

    socket.on('end', () => {
        console.log('Cliente desconectado');
    });
});

// Iniciar el servidor TCP
server.listen(3000, () => {
    console.log('Servidor TCP escuchando en puerto 3000');
});

```
# web.js

```js
const express = require('express');
const sqlite3 = require('sqlite3').verbose();
const app = express();
const port = 3001;  // Cambié el puerto a 3001
const path = require('path');  // Importar el módulo path

// Crear o abrir la base de datos
const db = new sqlite3.Database('./datos.db');
// Configurar para servir archivos estáticos (como HTML, CSS, JS)
app.use(express.static(path.join(__dirname, 'public')));

// Endpoint para obtener las lecturas desde la base de datos
app.get('/api/lecturas', (req, res) => {
  db.all('SELECT * FROM lecturas ORDER BY timestamp DESC', [], (err, rows) => {
    if (err) {
      return res.status(500).json({ error: err.message });
    }
    res.json(rows);  // Enviar los resultados como JSON
  });
});

// Iniciar el servidor Express
app.listen(port, () => {
  console.log(`Servidor HTTP corriendo en http://localhost:${port}`);
});


```
# Public/index.html
```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <title>Lecturas del Sensor</title>
  <style>
    body { font-family: Arial, sans-serif; padding: 20px; }
    table { width: 100%; border-collapse: collapse; }
    th, td { padding: 8px; border: 1px solid #ccc; text-align: left; }
  </style>
</head>
<body>
  <h1>Lecturas del Sensor</h1>
  <table id="tabla">
    <thead>
      <tr>
        <th>ID</th>
        <th>Temperatura (°C)</th>
        <th>Fecha</th>
      </tr>
    </thead>
    <tbody></tbody>
  </table>

  <script>
    // Realiza una solicitud a la API para obtener las lecturas
    fetch('http://localhost:3001/api/lecturas')
      .then(res => res.json())
      .then(data => {
        const tbody = document.querySelector('#tabla tbody');
        data.forEach(fila => {
          // Si el valor recibido es una temperatura en grados Celsius, se puede usar directamente
          const tr = document.createElement('tr');
          tr.innerHTML = `
            <td>${fila.id}</td>
            <td>${fila.valor} °C</td>  <!-- Mostrando la temperatura -->
            <td>${fila.timestamp}</td>
          `;
          tbody.appendChild(tr);
        });
      })
      .catch(err => {
        console.error('Error al obtener los datos:', err);
      });
  </script>
</body>
</html>


```


# Diagrama de conexion:
![image](https://github.com/user-attachments/assets/26885679-c106-4110-90e8-0c2db3cc3c69)

# Video Demostrativo:

https://drive.google.com/file/d/1MJ8UJMRHiPm69dkWAtC7kAxAvEi7xygj/view?usp=sharing

////////////////////////////////////////////////////////////////////////////
# Ejercicio 4: Envío de Notificaciones con Discord Webhooks:
Se reemplaza el correo por una integración con Discord.
1. Configura un webhook de Discord para recibir notificaciones desde un ESP32
cuando un sensor detecte un evento.

# Codigo:
```Python
import network
import urequests
import time
import json
from machine import Pin
import dht  # Importamos sensor DHT

# Configuración
SSID = "INFINITUMF116_EXT"
PASSWORD = "X4s9xzFrsx"
WEBHOOK_URL = "https://discord.com/api/webhooks/1369521823653302422/gkEHkoV1pZBmu_2RYUSKhrNQh3Lc7hx_xl_LmSAycb48Xgsv05I1INmPiq626lJJI4BY"

# Sensor de temperatura y humedad DHT22 en pin 15 (puedes cambiarlo)
sensor_dht = dht.DHT22(Pin(14))

def conectar_wifi():
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    if not wlan.isconnected():
        print("Conectando a WiFi...")
        wlan.connect(SSID, PASSWORD)
        while not wlan.isconnected():
            time.sleep(1)
    print("Conectado a WiFi:", wlan.ifconfig())

def enviar_discord(mensaje):
    headers = {'Content-Type': 'application/json'}
    cuerpo = json.dumps({"content": mensaje})
    try:
        respuesta = urequests.post(WEBHOOK_URL, data=cuerpo.encode('utf-8'), headers=headers)
        print("Mensaje enviado. Código:", respuesta.status_code)
        print("Respuesta:", respuesta.text)
        respuesta.close()
    except Exception as e:
        print("Error al enviar:", e)

# Programa principal
conectar_wifi()

while True:
    try:
        sensor_dht.measure()
        temp = sensor_dht.temperature()
        hum = sensor_dht.humidity()
        mensaje = f"🌡️ Temp: {temp:.1f}°C | 💧 Humedad: {hum:.1f}%"
        print("Enviando datos:", mensaje)
        enviar_discord(mensaje)
    except Exception as e:
        print("Error leyendo sensor:", e)
    time.sleep(60)  # Espera 60 segundos antes de enviar otra lectura
```
# Diagrama de conexion:
<img width="209" alt="Captura de pantalla 2025-05-06 225552" src="https://github.com/user-attachments/assets/3dff6079-871f-47f1-b0a5-01e79c6f4c69" />


# Video Demostrativo:

https://drive.google.com/file/d/1tfPAfppA5DrKD99I_d8A06hbACWEsEQ9/view?usp=sharing

////////////////////////////////////////////////////////////////////////////

# Autor 
Jose Andres Gutierrez Vargas

# Autoevaluación y Coevaluación
Durante el desarrollo de los ejercicios propuestos a lo largo del proyecto, considero que realicé un trabajo comprometido y completo, cumpliendo con cada uno de los objetivos planteados tanto a nivel técnico como en la parte de responsabilidad personal y trabajo colaborativo. Me esforcé por entender el funcionamiento de cada componente involucrado, desde el uso de sensores y comunicación por Bluetooth, hasta la integración con bases de datos y plataformas modernas como Discord mediante webhooks.

Pude aplicar de forma efectiva los conocimientos adquiridos en clase, y además me mantuve en constante búsqueda de información adicional para resolver los problemas que se presentaron. En particular, me resultó muy enriquecedor enfrentar retos como la configuración de bots, la interacción entre el ESP32 y servicios en la nube, y el uso de diferentes protocolos de comunicación. Cada desafío fue una oportunidad para fortalecer mis habilidades en programación, electrónica y comunicación de dispositivos IoT.

En cuanto a mi participación, considero que fui puntual en la entrega de todas las evidencias requeridas y mostré responsabilidad en la organización del trabajo. Además, mantuve una actitud colaborativa con mis compañeros, ofreciendo ayuda cuando surgían dudas o problemas técnicos, y compartiendo experiencias y soluciones para mejorar el aprendizaje grupal. Esta interacción me permitió también aprender de otros enfoques y validar mis propias soluciones.

Estoy satisfecho con el desempeño que tuve en este proyecto, ya que no solo logré cumplir con los requisitos establecidos, sino que también profundicé en el funcionamiento real de los sistemas conectados. Pienso que este trabajo refleja tanto mi compromiso como mi interés genuino por seguir desarrollándome en el área de tecnologías integradas y sistemas embebidos.




