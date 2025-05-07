# R2-Unidad-2-IOT
# INSTRUMENTO DE RECUPERACIÓN 2 (R2)

# Ejercicio 1: Comunicación Serial con Conversión de Datos
En lugar de solo enviar datos, el ESP32 deberá convertir la señal analógica de
un sensor en datos legibles en la PC.
1. Implementa una comunicación serial donde el ESP32 convierta la señal analógica de
un sensor en datos de temperatura o humedad y los envíe a la PC.

# Evidencias requeridas:
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

# Diagrama de conexion:
<img width="195" alt="Captura de pantalla 2025-05-06 155011" src="https://github.com/user-attachments/assets/d862c9d3-3ac3-4bf1-8930-80facfcbb69b" />

# Video Demostrativo:

# Ejercicio 2: Comunicación Bluetooth con Control desde un Chatbot
En lugar de encender un LED, se usará un chatbot en Telegram o WhatsApp
para controlar el Bluetooth.
1. Configura un bot que, al recibir un mensaje en Telegram o WhatsApp, envíe
comandos a un ESP32 para encender/apagar un LED o mover un servo.
# Evidencias requeridas:
# Codigo:

# servidor.py 

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

# esp32
# Control de Servo por Bluetooth BLE para ESP32 en MicroPython
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


# Diagrama de conexion:
![image](https://github.com/user-attachments/assets/b3899046-a898-4fcd-82e3-e4e09de723ca)

# Video Demostrativo:

# Ejercicio 3: Comunicación TCP/IP con Base de Datos:
En lugar de solo enviar datos, estos deben almacenarse en una base de datos
(SQLite, Firebase, PostgreSQL).
1. Implementa una comunicación TCP/IP donde un ESP32 registre datos en una base
de datos en la nube o local.

# Evidencias requeridas:
# Codigo:
# Diagrama de conexion:
# Video Demostrativo:

# Ejercicio 4: Envío de Notificaciones con Discord Webhooks:
Se reemplaza el correo por una integración con Discord.
1. Configura un webhook de Discord para recibir notificaciones desde un ESP32
cuando un sensor detecte un evento.
# Evidencias requeridas:
# Codigo:
# Diagrama de conexion:
# Video Demostrativo:

# Autores 

Juana Jaqueline Camarrillo Olaez
<br>
Jose Andres Gutierrez Vargas
<br>
Princes Rocio Guerrero Sánchez 
<br>
