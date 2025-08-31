# Proyecto IoT: ESP32 LED RGB con MQTT y Servidor Web


Esta es la guia para realizar la demo de  un sistema Iot basico donde un ESP32-C6 controla un LED RGB que cambia de colores cada 3 segundos, envía la información por MQTT a un servidor en AWS EC2, y una página web muestra los cambios de color en tiempo real.


## 📋 Componentes del Sistema

- **ESP32-C6-DevKitC-1 v1.2** con LED RGB WS2812 integrado
- **Servidor AWS EC2** con Ubuntu
- **Broker MQTT Mosquitto** con soporte WebSocket
- **Servidor Web Apache** 
- **Página Web** con visualización en tiempo real

## 🏗️ Arquitectura del Sistema

```
ESP32-C6 ──(WiFi)──> Internet ──> AWS EC2 Server
    │                                │
    └─ LED RGB WS2812               ├─ Mosquitto MQTT (puerto 1883)
                                    ├─ WebSocket MQTT (puerto 9001)
                                    └─ Apache Web (puerto 80)
                                           └─ Página Web
```

---

## 🚀 Guía de Instalación Paso a Paso

### Parte 1: Configuración del Servidor AWS EC2

#### 1.1 Crear Instancia EC2

1. **Acceder a AWS Console**:
   - Ve a [AWS Console](https://aws.amazon.com/console/)
   - Navega a **EC2 > Instances**

  ![AWS EC2 Instance](https://drive.google.com/file/d/155OIdVFIBjbC9M_az7C33ndBVkHtx5FY/view?usp=drive_link)

2. **Lanzar Nueva Instancia**:
   - Click en **"Launch Instance"**
   
  ![Lanzar instancia](https://drive.google.com/file/d/1KsmVoDUVva6_cD9U2vwYWw1qdt23LKVk/view?usp=sharing)


   - **Name**: `esp-webserver`

   ![Nombre server](https://drive.google.com/file/d/1r6W-NnYiei_XSrN2-oPW5_Kc4phu5rW0/view?usp=sharing)

   - **AMI**: Ubuntu Server 22.04 LTS (Free tier eligible)
   - **Instance Type**: t2.micro (Free tier eligible)

3. **Configurar Security Groups**:
   - **SSH (22)**: Tu IP (para acceso remoto)
   - **HTTP (80)**: 0.0.0.0/0 (para la página web)

    ![Grupo de seguridad 1](https://drive.google.com/file/d/1J1x6sIn0GQfFgD3MkQE7eZjF7XVsXBpr/view?usp=sharing)

   - **MQTT (1883)**: 0.0.0.0/0 (para el ESP32)
   - **WebSocket (9001)**: 0.0.0.0/0 (para la página web)

   ![Grupo de seguridad 2](https://drive.google.com/file/d/1YQQ4VMH-6w52Fr4fggVbDjCO5tVakzwa/view?usp=sharing)

   ![Grupo de seguridad 3](https://drive.google.com/file/d/1tD4k7NFHZsZwpLkgSl1nUCG2CPb9nd7p/view?usp=sharing)

4. **Configurar Key Pair**:

  ![Key-par 1](https://drive.google.com/file/d/1tD4k7NFHZsZwpLkgSl1nUCG2CPb9nd7p/view?usp=sharing)
   - Crear nuevo Key Pair o usar existente


   ![Key-par 2](https://drive.google.com/file/d/1WInfO19BwOqGEjt24B689RRF2Xo3-r6P/view?usp=sharing)



   - Descargar archivo `.pem` para acceso SSH


   ![Key-par 3](https://drive.google.com/file/d/1s8-bWCufL0JACIJkdMwakB6106TaICSw/view?usp=sharing)


5. **Lanzar Instancia** y anotar la **IP pública**

![Launch instance 1](https://drive.google.com/file/d/1QQsbhHSKzdzeS5czW2XZDGCT5SBHPyK3/view?usp=sharing)

![Launch instance 2](https://drive.google.com/file/d/1MOEXaDMneSLZQzOcl8q3mhrGXcwUp3ap/view?usp=sharing)


#### 1.2 Conectar al Servidor

```bash
# Cambiar permisos del archivo .pem
chmod 400 tu-key.pem

# Conectar por SSH
ssh -i tu-key.pem ubuntu@TU_IP_PUBLICA
```

#### 1.3 Actualizar Sistema

```bash
sudo apt update
sudo apt upgrade -y
```

### Parte 2: Instalación y Configuración de Apache Web Server

#### 2.1 Instalar Apache

```bash
sudo apt install apache2 -y
```

#### 2.2 Verificar Instalación

```bash
sudo systemctl status apache2
sudo systemctl start apache2
sudo systemctl enable apache2
```

#### 2.3 Verificar Acceso Web

- Abrir navegador y ir a `http://IP_PUBLICA_SERVER`
- Deberías ver la página por defecto de Apache

### Parte 3: Instalación y Configuración de Mosquitto MQTT

#### 3.1 Instalar Mosquitto

```bash
sudo apt install mosquitto mosquitto-clients -y
```

#### 3.2 Configurar Mosquitto

```bash
# Crear archivo de configuración
sudo nano /etc/mosquitto/conf.d/mosquitto.conf
```

**Contenido del archivo:**
```
listener 1883
protocol mqtt
password_file /etc/mosquitto/passwd
allow_anonymous false

listener 9001
protocol websockets
allow_anonymous true
```

#### 3.3 Crear Usuario y Contraseña

```bash
# Crear usuario 'esp32' con contraseña '12345678'
sudo mosquitto_passwd -c /etc/mosquitto/passwd esp32
# Cuando pregunte, ingresar: 12345678

# Configurar permisos
sudo chown mosquitto:mosquitto /etc/mosquitto/passwd
sudo chmod 640 /etc/mosquitto/passwd
```

#### 3.4 Reiniciar y Verificar Mosquitto

```bash
sudo systemctl restart mosquitto
sudo systemctl enable mosquitto
sudo systemctl status mosquitto
```

#### 3.5 Probar MQTT

```bash
# Terminal 1 - Suscribirse
mosquitto_sub -h localhost -t esp32/led -u esp32 -P 12345678

# Terminal 2 - Publicar
mosquitto_pub -h localhost -t esp32/led -m "test" -u esp32 -P 12345678
```

### Parte 4: Crear Página Web

#### 4.1 Crear archivo HTML

```bash
sudo nano /var/www/html/index.html
```

**Contenido del archivo:**
```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <title>ESP32 LED Monitor</title>
  <style>
    body { font-family: Arial, sans-serif; text-align: center; margin-top: 40px; }
    #circle {
      width: 120px;
      height: 120px;
      border-radius: 50%;
      background: #222;
      margin: 0 auto 20px auto;
      box-shadow: 0 0 20px #888;
      transition: background 0.5s;
    }
    #status {
      margin: 20px 0;
      font-weight: bold;
    }
    .connected { color: green; }
    .disconnected { color: red; }
    .connecting { color: orange; }
  </style>
</head>
<body>
  <h2>ESP32 LED Monitor</h2>
  <div id="circle"></div>
  <div id="status" class="connecting">Conectando a MQTT...</div>
  <div>Color actual: <span id="colorText">---</span></div>
  <div>Mensajes recibidos: <span id="messageCount">0</span></div>

  <script src="https://cdnjs.cloudflare.com/ajax/libs/paho-mqtt/1.0.1/mqttws31.min.js"></script>
  <script>
    window.onload = function() {
      const brokerHost = "IP_PUBLICA_SERVER"; // CAMBIAR  LA IP
      const brokerPort = 9001;
      const topic = "esp32/led";
      const username = "esp32";
      const password = "12345678";

      const colorMap = {
        red: "#ff0000",
        green: "#00ff00",
        blue: "#0000ff"
      };

      let messageCount = 0;
      let client = new Paho.MQTT.Client(brokerHost, brokerPort, "webclient_" + Math.random().toString(16).substr(2, 8));

      client.onConnectionLost = function(responseObject) {
        console.log("Conexión perdida: " + responseObject.errorMessage);
        document.getElementById('status').textContent = "Desconectado de MQTT";
        document.getElementById('status').className = "disconnected";
      };

      client.onMessageArrived = function(message) {
        console.log("Mensaje recibido: " + message.payloadString);
        const color = message.payloadString.toLowerCase();
        messageCount++;
        
        document.getElementById('colorText').textContent = color;
        document.getElementById('messageCount').textContent = messageCount;
        document.getElementById('circle').style.background = colorMap[color] || "#222";
      };

      const connectOptions = {
        userName: username,
        password: password,
        timeout: 10,
        keepAliveInterval: 30,
        onSuccess: function() {
          console.log("Conectado exitosamente a MQTT broker");
          document.getElementById('status').textContent = "Conectado a MQTT";
          document.getElementById('status').className = "connected";
          client.subscribe(topic);
          console.log("Suscrito al topic: " + topic);
        },
        onFailure: function(error) {
          console.log("Error de conexión: " + error.errorMessage);
          document.getElementById('status').textContent = "Fallo la conexión MQTT: " + error.errorMessage;
          document.getElementById('status').className = "disconnected";
        }
      };

      console.log("Intentando conectar a: " + brokerHost + ":" + brokerPort);
      client.connect(connectOptions);
    };
  </script>
</body>
</html>
```



#### 4.2 Verificar Página Web

- Ir a `http://IP_PUBLICA_SERVER/index.html`
- Debería mostrar la página con estado "Conectando a MQTT..."

---

### Parte 5: Configuración del Proyecto ESP32

#### 5.1 Prerequisitos

- **ESP-IDF v5.5** instalado
- **ESP32-C6-DevKitC-1 v1.2**
- **Cable USB-C**

#### 5.2 Estructura del Proyecto

El proyecto ya está configurado con los archivos necesarios:

```
main/
├── CMakeLists.txt           # Configuración de dependencias
├── idf_component.yml        # Dependencia led_strip
├── Kconfig.projbuild        # Configuración del proyecto
└── station_example_main.c   # Código principal
```

#### 5.3 Configurar Variables en el Código

Editar `main/station_example_main.c` y cambiar las siguientes líneas:

```c
// Configuración WiFi - CAMBIAR POR TUS DATOS
#define EXAMPLE_ESP_WIFI_SSID      "RED_WIFI"
#define EXAMPLE_ESP_WIFI_PASS      "PASSWORD_WIFI"

// Configuración MQTT - CAMBIAR POR TU IP
#define MQTT_BROKER_URL "mqtt://_IP_PUBLICA_SERVER:1883"
```

⚠️ **IMPORTANTE**: Reemplazar:
- `RED_WIFI`: Nombre de tu red WiFi
- `PASSWORD_WIFI`: Contraseña de tu red WiFi  
- `IP_PUBLICA_SERVER`: IP pública de tu servidor AWS EC2

#### 5.4 Compilar y Flashear

```bash
# Configurar target para ESP32-C6
idf.py set-target esp32c6

# Configurar proyecto (opcional)
idf.py menuconfig

# Compilar
idf.py build

# Conectar ESP32 por USB y flashear
idf.py flash

# Monitor serial para ver logs
idf.py monitor
```

#### 5.5 Configuración de Red (opcional)

Si prefieres configurar WiFi mediante menuconfig:

```bash
idf.py menuconfig
```

Navegar a: **Example Configuration** y establecer:
- WiFi SSID
- WiFi Password

---

## 🧪 Verificación del Sistema

### 1. Verificar ESP32

**En el monitor serial (`idf.py monitor`) deberías ver:**
```
I (2450) ESP32_MQTT_LED: Connected to WiFi SSID:RED_WIFI
I (2460) ESP32_MQTT_LED: Got IP: 192.168.x.x
I (2470) ESP32_MQTT_LED: MQTT Broker: mqtt://IP_PUBLICA_SERVER:1883
I (2890) ESP32_MQTT_LED: MQTT Connected
I (2900) LED: Color: ROJO REAL - LED RGB encendido por 3 segundos
I (5910) LED: Color: VERDE REAL - LED RGB encendido por 3 segundos
I (8920) LED: Color: AZUL REAL - LED RGB encendido por 3 segundos
```

**LED RGB físico debe:**
- Cambiar colores cada 3 segundos: Rojo → Verde → Azul
- Mostrar colores puros y brillantes

### 2. Verificar Servidor MQTT

```bash
# En el servidor AWS, monitorear mensajes en tiempo real
mosquitto_sub -h localhost -t esp32/led -u esp32 -P 12345678
```

**Deberías ver:**
```
red
green
blue
red
green
blue
...
```

### 3. Verificar Página Web

- Ir a `http://TU_IP_PUBLICA/index.html`
- **Estado**: "Conectado a MQTT" (verde)
- **Círculo**: Debe cambiar de color sincronizado con el ESP32
- **Contador**: Debe incrementar con cada mensaje
- **Color actual**: Debe mostrar "red", "green", "blue"

---



## 📊 Diagrama de Flujo del Sistema

```
Inicio
  ↓
Inicializar NVS
  ↓
Conectar WiFi
  ↓
Inicializar LED RGB
  ↓
Conectar MQTT
  ↓
Crear Tarea LED
  ↓
Loop infinito:
  ├─ Cambiar color LED (rojo/verde/azul)
  ├─ Publicar color por MQTT
  ├─ Esperar 3 segundos
  ├─ Apagar LED brevemente
  └─ Repetir con siguiente color
```

---



### Servidor
- **Cloud Provider**: AWS EC2
- **OS**: Ubuntu 22.04 LTS
- **Broker MQTT**: Mosquitto
- **Web Server**: Apache2
- **Puertos**: 22 (SSH), 80 (HTTP), 1883 (MQTT), 9001 (WebSocket)

---


