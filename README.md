# Laboratorio de Integración IoT: Simulador de Telemetría con MQTT Mosquitto, Node-RED y Docker

Este proyecto establece un entorno local de pruebas para simular la recolección, publicación y procesamiento de datos de telemetría en tiempo real. Utiliza el protocolo de comunicación MQTT a través del broker Eclipse Mosquitto, un motor de flujos en Node-RED para la lógica de automatización y Docker Compose para la orquestación de contenedores.

---

## 1. Requisitos del sistema

- Docker Engine instalado y en ejecución.
- Docker Compose v2.x o superior.
- Navegador web moderno.

---

## 2. Arquitectura del sistema

El laboratorio se divide en tres componentes principales:

1. **Broker MQTT (Eclipse Mosquitto):** Actúa como el intermediario de mensajería central bajo el patrón Publicación/Suscripción (Pub/Sub). Escucha solicitudes en el puerto estándar `1883`.
2. **Emisor de Telemetría (Simulador):** Un flujo dentro de Node-RED que genera eventos simulados con cargas útiles en formato JSON hacia el tópico `fabrica/sensores/temperatura`.
3. **Receptor y procesador de eventos:** Un flujo en Node-RED suscrito al tópico `fabrica/sensores/temperatura` que filtra y evalúa las lecturas mediante una función de JavaScript.

---

## 3. Configuración e infraestructura

Para levantar los servicios, cree un archivo nombrado `compose.yaml` con la siguiente configuración:

```yaml
services:
  mqtt:
    image: eclipse-mosquitto:2.1-alpine
    container_name: mosquitto-broker
    ports:
      - "1883:1883"
    restart: unless-stopped
    volumes:
      - ./mosquitto/config:/mosquitto/config
      - ./mosquitto/data:/mosquitto/data
      - ./mosquitto/log:/mosquitto/log
  nodered:
    image: nodered/node-red:latest
    container_name: nodered-container
    ports:
      - "1880:1880"
    restart: unless-stopped
    depends_on:
      - mqtt
    environment:
      - TZ=America/Cancun
    volumes:
      - node-red-data:/data

volumes:
  node-red-data:
```

### Comandos de despliegue

Inicie los contenedores en segundo plano ejecutando el siguiente comando en la terminal:

```bash
docker compose up -d
```

Para verificar el estado de la ejecución:

```bash
docker compose ps
```

---

## 4. Configuración del flujo en Node-RED

Acceda a la interfaz de Node-RED ingresando a la dirección `http://localhost:1880` en su navegador web.

### Estructura de los nodos

#### Componente A: Publicador (Simulador de sensor)

1. **Nodo Inject (`Datos simulados`):**
   - **Payload Type:** JSON
   - **Payload Value:** `{"sensorId": "BLAR", "temperature": 45.5}`
   - **Repeat:** `none` (Manual)
2. **Nodo MQTT Out (`fabrica/sensores/temperatura`):**
   - **Server:** `localhost:1883`
   - **Topic:** `fabrica/sensores/temperatura`
   - **QoS:** `2`
   - **Retain:** `false`

Conexión: `Inject` -> `MQTT Out`

#### Componente B: Suscriptor y procesador

1. **Nodo MQTT In (`fabrica/sensores/temperatura`):**
   - **Server:** `localhost:1883`
   - **Topic:** `fabrica/sensores/temperatura`
   - **Output:** `parsed JSON object`
2. **Nodo Function (`Procesar Alerta`):**
   - **Código:**
```javascript
/**
 * @typedef {Object} SensorData
 * @property {string} sensorId
 * @property {number} temperature
 */

/** @type {SensorData} */
const data = msg.payload;

if(data.temperature > 40){
    msg.payload = {
        estado: "PELIGRO",
        mensaje: `¡Alerta! Temperatura muy alta: ${data.temperature}°C`
    };
} else {
    msg.payload = {
        estado: "NORMAL",
        temperatura: data.temperature
    };
}

return msg;
```
3. **Nodo Debug (`Consola`):**
   - **Output:** `msg.payload`

Conexión: `MQTT In` -> `Function` -> `Debug`

---

## 5. Validación del funcionamiento

1. Haga clic en el botón **Deploy** ubicado en la esquina superior derecha de la interfaz de Node-RED.
2. Abra el panel lateral derecho de depuración (pestaña Debug).
3. Active manualmente el nodo **Inject** haciendo clic en el botón ubicado a su izquierda.
4. Verifique la salida en la consola de depuración. La estructura procesada debe mostrar la evaluación del umbral de temperatura correspondiente.

---
