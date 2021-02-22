# IoT Dashboard - Sensores, MQTT, Telegraf, InfluxDB y Grafana
## Descripción

En cada aula del instituto vamos a tener un [Wemos D1 mini](https://wemos.cc), un [sensor de CO2](https://wiki.keyestudio.com/KS0457_keyestudio_CCS811_Carbon_Dioxide_Air_Quality_Sensor) y un [sensor de temperatura/humedad DHT11](https://learn.adafruit.com/dht/overview) que van a ir tomando medidas de forma constante y las van a ir publicando en un *topic* de un [*broker* MQTT](http://mqtt.org). Podríamos seguir la siguiente estructura de nombres para los *topics* del edificio:

```
iescelia/aula<número>/temperature
iescelia/aula<número>/humidity
iescelia/aula<número>/co2
```

Por ejemplo para el `aula20` tendríamos los siguientes *topics*:

```
iescelia/aula20/temperature
iescelia/aula20/humidity
iescelia/aula20/co2
```

También existirá un agente de [Telegraf](https://www.influxdata.com/time-series-platform/telegraf/) que estará suscrito a los  *topics* del [*broker* MQTT](http://mqtt.org) donde se publican los valores recogidos por los sensores.  El agente de [Telegraf](https://www.influxdata.com/time-series-platform/telegraf/) insertará los valores que recoge del [*broker* MQTT](http://mqtt.org) en una base de datos [InfluxDB](https://www.influxdata.com/), que es un sistema gestor de bases de datos diseñado para almacenar series temporales de datos.Finalmente tendremos un servicio web [Grafana](https://grafana.com) que nos permitirá visualizar los datos en un panel de control.

## Sensores
### Wemos D1 mini
Wemos D1 mini es una placa de prototipado que incluye el SoC (System on Chip) ESP8266 que integra un microcontrolador de propósito general de 32 bits y conectividad WiFi.

### Sensor de temperatura/humedad DHT11
####  Lectura del sensor de temperatura/humedad DHT11 y publicación de datos en el broker MQTT
Vamos a hacer uso de la librería Adafruit MQTT para conectar con el broker MQTT y publicar los datos que vamos obteniendo de los sensores DHT11. Podemos utilizar el siguiente código de ejemplo:

```
#include "DHT.h"
#include <ESP8266WiFi.h>
#include "Adafruit_MQTT.h"
#include "Adafruit_MQTT_Client.h"

#define DHTPIN D4
#define DHTTYPE DHT11

DHT dht(DHTPIN, DHTTYPE);

#define WLAN_SSID       "PUT_YOUR_WLAN_SSID_HERE" 
#define WLAN_PASS       "PUT_YOUR_WLAN_PASS_HERE" 

#define MQTT_SERVER      "PUT_YOUR_MQTT_SERVER_HERE" 
#define MQTT_SERVERPORT  1883
#define MQTT_USERNAME    ""
#define MQTT_KEY         ""
#define MQTT_FEED_TEMP   "iescelia/aula22/temperature" 
#define MQTT_FEED_HUMI   "iescelia/aula22/humidity" 

WiFiClient client;

Adafruit_MQTT_Client mqtt(&client, MQTT_SERVER, MQTT_SERVERPORT, MQTT_USERNAME, MQTT_USERNAME, MQTT_KEY);

Adafruit_MQTT_Publish temperatureFeed = Adafruit_MQTT_Publish(&mqtt, MQTT_FEED_TEMP);

Adafruit_MQTT_Publish humidityFeed = Adafruit_MQTT_Publish(&mqtt, MQTT_FEED_HUMI);

//----------------------------------------------

void connectWiFi();

//----------------------------------------------

void setup() {
  Serial.begin(115200);
  Serial.println("IoT demo");
  dht.begin();
  connectWiFi();
  connectMQTT();
}

//----------------------------------------------

void loop() {
  delay(2000);

  float h = dht.readHumidity();
  float t = dht.readTemperature();

  if (isnan(h) || isnan(t)) {
    Serial.println("Failed to read from DHT sensor!");
    return;
  }

  Serial.print("Humidity: ");
  Serial.print(h);
  Serial.print(" %\t");
  Serial.print("Temperature: ");
  Serial.print(t);
  Serial.println(" *C ");

  temperatureFeed.publish(t);
  humidityFeed.publish(h);
}

//----------------------------------------------

void connectWiFi() {
  WiFi.begin(WLAN_SSID, WLAN_PASS);
  while (WiFi.status() != WL_CONNECTED) {
     delay(500);
     Serial.print(".");
  }

  Serial.println("WiFi connected");
  Serial.println("IP address: ");
  Serial.println(WiFi.localIP());
}

//----------------------------------------------

void connectMQTT() {
  if (mqtt.connected())
    return;

  Serial.print("Connecting to MQTT... ");
  while (mqtt.connect() != 0) {
       Serial.println("Error. Retrying MQTT connection in 5 seconds...");
       mqtt.disconnect();
       delay(5000);
  }
}

```
### Sensor de CO2
#### Lectura del sensor de CO2 y publicación de datos en el broker MQTT
Vamos a hacer uso de la librería Adafruit MQTT para conectar con el broker MQTT y publicar los datos que vamos obteniendo de los sensores CCS811. Podemos utilizar el siguiente código de ejemplo:

```
#include <ESP8266WiFi.h>
#include <Adafruit_Sensor.h>
#include "Adafruit_MQTT.h"
#include "Adafruit_MQTT_Client.h"
#include <CCS811.h>

// WiFi configuration
#define WLAN_SSID       "PUT_YOUR_WLAN_SSID_HERE" 
#define WLAN_PASS       "PUT_YOUR_WLAN_PASS_HERE" 

// MQTT configuration
#define MQTT_SERVER      "PUT_YOUR_MQTT_SERVER_HERE" 
#define MQTT_SERVERPORT  1883
#define MQTT_USERNAME    ""
#define MQTT_KEY         ""
#define MQTT_FEED_CO2   "iescelia/aula22/co2" 
#define MQTT_FEED_TVOC  "iescelia/aula22/tvoc" 

// WiFi connection
WiFiClient client;

// MQTT connection
Adafruit_MQTT_Client mqtt(&client, MQTT_SERVER, MQTT_SERVERPORT, MQTT_USERNAME, MQTT_USERNAME, MQTT_KEY);

// Feed to publish CO2
Adafruit_MQTT_Publish co2Feed = Adafruit_MQTT_Publish(&mqtt, MQTT_FEED_CO2);

// Feed to publish TVOC
Adafruit_MQTT_Publish tvocFeed = Adafruit_MQTT_Publish(&mqtt, MQTT_FEED_TVOC);

CCS811 sensor;

//----------------------------------------------

void connectWiFi();
void connectMQTT();
void initSensor();

//----------------------------------------------

void setup() {
  Serial.begin(115200);
  Serial.println("CO2 IES Celia");
  connectWiFi();
  connectMQTT();
  initSensor();
}

//----------------------------------------------

void loop() {
  // Wait a few seconds between measurements.
  delay(1000);

  if(sensor.checkDataReady() == false){
    Serial.println("Data is not ready!");
    return;
  }

  float co2 = sensor.getCO2PPM();
  float tvoc = sensor.getTVOCPPB();

  Serial.print("CO2: ");
  Serial.print(co2);
  Serial.print(" ppm\t");
  Serial.print("TVOC: ");
  Serial.print(tvoc);
  Serial.println(" ppb");

  co2Feed.publish(co2);
  tvocFeed.publish(tvoc);
}

//----------------------------------------------

void initSensor() {
  // Wait for the chip to be initialized completely
  while(sensor.begin() != 0){
    Serial.println("Failed to init chip, please check if the chip connection is fine");
    delay(1000);
  }

  // eClosed      Idle (Measurements are disabled in this mode)
  // eCycle_1s    Constant power mode, IAQ measurement every second
  // eCycle_10s   Pulse heating mode IAQ measurement every 10 seconds
  // eCycle_60s   Low power pulse heating mode IAQ measurement every 60 seconds
  // eCycle_250ms Constant power mode, sensor measurement every 250ms
  sensor.setMeasCycle(sensor.eCycle_250ms);
}

//----------------------------------------------

void connectWiFi() {
  WiFi.begin(WLAN_SSID, WLAN_PASS);
  while (WiFi.status() != WL_CONNECTED) {
     delay(500);
     Serial.print(".");
  }

  Serial.println("WiFi connected");
  Serial.println("IP address: ");
  Serial.println(WiFi.localIP());
}

//----------------------------------------------

void connectMQTT() {
  if (mqtt.connected())
    return;

  Serial.print("Connecting to MQTT... ");
  while (mqtt.connect() != 0) {
       Serial.println("Error. Retrying MQTT connection in 5 seconds...");
       mqtt.disconnect();
       delay(5000);
  }
}
```
## MQTT Broker
MQTT (MQ Telemetry Transport) es un protocolo de mensajería estándar utilizado en las aplicaciones de Internet de las cosas (IoT). Se trata de un protocolo de mensajería muy ligero basado en el patrón publish/subscribe, donde los mensajes son publicados en un topic de un MQTT Broker que se encargará de distribuirlos a todos los suscriptores que se hayan suscrito a dicho topic.

Eclipse Mosquitto será el MQTT Broker que vamos a utilizar en este proyecto.

### Descripción del servicio en docker-compose.yml
docker-compose.yml

```
mosquitto: 
    image: eclipse-mosquitto:2 
    ports:
      - 1883:1883 
    volumes:
      - ./mosquitto/mosquitto.conf:/mosquitto/config/mosquitto.conf 
      - mosquitto_data:/mosquitto/data 
      - mosquitto_log:/mosquitto/log 
```

###  Archivo de configuración mosquitto.conf
Este archivo estará almacenado en el directorio ./mosquitto/mosquitto.conf de nuestra máquina local. Como vamos a utilizar la versión 2 del broker MQTT eclipse-mosquitto vamos a necesitar crear un archivo de configuración llamado mosquitto.conf que incluya las siguientes opciones de configuración.

mosquitto.conf

````
listener 1883 
allow_anonymous true
````
En este punto, el contenido de nuestro directorio de trabajo debe tener los siguientes archivos.

````
.
├── docker-compose.yml
├── mosquitto
    └── mosquitto.conf
````
### Cliente MQTT para publicar en un topic (publish)

En esta sección vamos a explicar cómo podemos hacer uso de un cliente MQTT para publicar mensajes de prueba en el broker MQTT que acabamos de crear.

Esta comprobación deberíamos realizarla antes de empezar a trabajar directamente con los sensores sobre el broker MQTT. Una vez que hayamos comprobado que podemos publicar y suscribirnos a los mensajes del broker con este cliente, ya podríamos realizar el siguiente paso para publicar mensajes con los sensores y recibirlos con el agente de Telegraf.

Ejemplo:

En este ejemplo vamos a publicar el valor 30 en el topic iescelia/aula22/co2 del broker MQTT test.mosquitto.org. Este broker es de prueba, por lo tanto, tendremos que cambiar este broker por la dirección IP de nuestro broker MQTT.

````
docker run --init -it --rm efrecon/mqtt-client pub -h test.mosquitto.org -p 1883 -t "iescelia/aula22/co2" -m 30
````
### Cliente MQTT para suscribirse a un topic (subscribe)
Podemos hacer uso de un cliente MQTT para suscribirnos a los mensajes que se publican en un topic específico del broker MQTT.

Esta comprobación deberíamos realizarla antes de empezar a trabajar directamente con los sensores sobre el broker MQTT. Una vez que hayamos comprobado que podemos publicar y suscribirnos a los mensajes del broker con este cliente, ya podríamos realizar el siguiente paso para publicar mensajes con los sensores y recibirlos con el agente de Telegraf.

Ejemplo:

En este ejemplo nos vamos a suscribir al topic iescelia/ del broker MQTT test.mosquitto.org. Con el wildcard estamos indicando que queremos suscribirnos a todos los topics que existan dentro de iescelia/, es decir, todos los mensajes que se publiquen en cada una de las aulas.

Tenga en cuenta que tendrá que cambiar el broker test.mosquitto.org por la dirección IP de nuestro broker MQTT.

````
docker run --init -it --rm efrecon/mqtt-client sub -h test.mosquitto.org -t "iescelia/#"
````
## Telegraf
Telegraf es un agente que nos permite recopilar y reportar métricas. Las métricas recogidas se pueden enviar a almacenes de datos, colas de mensajes o servicios como: InfluxDB, Graphite, OpenTSDB, Datadog, Kafka, MQTT, NSQ, entre otros.

### Descripción del servicio en docker-compose.yml
docker-compose.yml

````
telegraf: 
    image: telegraf 
      volumes:
        - ./telegraf/telegraf.conf:/etc/telegraf/telegraf.conf 
      depends_on: 
        - influxdb
````

### Creación del archivo de configuración telegraf.conf
En primer lugar tenemos que crear el archivo de configuración telegraf.conf en nuestra máquina local. Para crear el archivo de configuración haremos uso del comando telegram config dentro del contenedor de Telegraf.

````
docker run --rm telegraf telegraf config > telegraf.conf
````
Una vez que hemos creado el archivo telegraf.conf lo movemos al directorio telegraf de nuestro proyecto. El contenido de nuestro directorio de trabajo debe tener los siguientes archivos.

````
.
├── docker-compose.yml
├── mosquitto
│   └── mosquitto.conf
└── telegraf
    └── telegraf.conf
````
###  Configuración del archivo telegraf.conf para suscribirnos a un topic MQTT
Tendremos que buscar la sección inputs.mqtt_consumer dentro del archivo telegraf.conf y configurar los siguientes valores:

* servers

* topics

* data_format

Existen más directivas de configuración (qos, client_id, username, password, etc.), pero en este proyecto sólo vamos a utilizar los valores que hemos indicado anteriormente.

servers  
En esta directiva de configuración indicamos la URL del broker MQTT al que queremos conectarnos. En nuestro caso pondremos el nombre del servicio mosquitto que es como lo hemos definido en nuestro archivo docker-compose.yml.

topics  
En esta directiva indicamos los topics a los que queremos suscribirnos. En nuestro caso pondremos el topic iescelia/#. El carácter # al final del topic indica que nos vamos a suscribir a cualquier topic que exista después de la cadena iescelia/.

data_format  
En esta directiva indicamos cómo es el formato de los datos que vamos a recibir por MQTT.

Telegraf contiene muchos plugins de propósito general que permiten parsear datos de entrada para convertirlos en el modelo de datos de métricas que utiliza InfluxDB.

### Formato: Value
El formato value permite convertir valores simples en métricas de Telegraf.

Es necesario indicar el tipo de dato al que queremos convertir el valor leído, utilizando la opción de configuración data_type. Los tipos de datos disponibles son:

1. integer

2. float o long

3. string

4. boolean

#### Ejemplo de configuración de la sección inputs.mqtt_consumer
Si los mensajes que vamos a recibir por MQTT sólo contienen valores numéricos de tipo real que representan valores de temperatura, tendríamos que indicar:

data_format = "value" y

data_type = "float".

Una posible configuración para la sección inputs.mqtt_consumer del archivo telegraf.conf podría ser la siguiente.

````
 [[inputs.mqtt_consumer]]
   servers = ["tcp://mosquitto:1883"] 

   topics = [
     "iescelia/#" 
   ]

  data_format = "value" 
  data_type = "float" 
````
### Formato: InfluxDB Line Protocol
El formato influx no necesita ninguna configuración adicional. Los valores que se reciben son parseados automáticamente a métricas de Telegraf.

Si utilizamos data_format = "influx" los mensajes de entrada tienen que cumplir la sintaxis del protocolo InfluxDB line protocol.

A continuación se muestra un ejemplo de un mensaje con las sintaxis de InfluxDB line protocol.

````
weather,location=us-midwest temperature=82 1465839830100400200
  |    -------------------- --------------  |
  |             |             |             |
  |             |             |             |
+-----------+--------+-+---------+-+---------+
|measurement|,tag_set| |field_set| |timestamp|
+-----------+--------+-+---------+-+---------+
````
El mensaje se compone de cuatro componentes:

* measurement: Indica la estructura de datos donde vamos a almacenar los valores en InfluxDB.

* tag_set: Es opcional. Son tags opcionales asociados al dato. Van detrás de una coma después del nombre de la measurement, sin espacios en blanco.

* field_set: Son los datos. Aparecen después de un espacio en blanco después de la meausrement. Podemos indicar varios datos separados por comas.

* timestamp: Es opcional. Es una marca de tiempo en Unix con precisión de nanosegundos.

#### Ejemplos de mensajes válidos con la sintaxis del protocolo InfluxDB line protocol
````
weather,location=us-midwest temperature=82 
````
````
1465839830100400200
````
````
weather temperature=82 1465839830100400200
````
````
weather temperature=82
````
````
weather temperature=82,humidity=71 1465839830100400200
````
````
weather temperature=82,humidity=71
````
#### Ejemplo de configuración de la sección inputs.mqtt_consumer
Una posible configuración para la sección inputs.mqtt_consumer del archivo telegraf.conf podría ser la siguiente.

````
 [[inputs.mqtt_consumer]]
   servers = ["tcp://mosquitto:1883"] 

   topics = [
     "iescelia/#" 
   ]

   data_format = "influx" 
````
### Configuración del archivo telegraf.conf para almacenar los datos en InfluxDB (outputs.influxdb)

Tendremos que buscar la sección outputs.influxdb dentro del archivo telegraf.conf y configurar los siguientes valores:

* urls

* database

* skip_database_creation

* username

* password

Existen más directivas de configuración, pero en este proyecto sólo vamos a utilizar los valores que hemos indicado anteriormente.

#### urls
En esta directiva de configuración indicamos la URL de la instancia de InfluxDB donde vamos a almacenar los datos. En nuestro caso pondremos influxdb que es el nombre que le hemos puesto al servicio en el archivo docker-compose.yml

#### database
En esta directiva indicamos el nombre de la base de datos donde vamos a almacenar las métricas.

#### skip_database_creation
Permite evitar la creación de bases de datos en InfluxDB cuando está configurada como true. Se recomienda configurarla a true cuando estemos trabajando con usuarios que no tienen permisos para crear bases de datos o cuando la base de datos ya exista.

#### username y password
En esta directiva configuramos los parámetros de autenticación para conectarnos a InfluxDB.

#### Ejemplo de configuración de la sección outputs.mqtt_consumer
Una posible configuración de la sección outputs.influxdb podría ser la siguiente:

````
[[outputs.influxdb]]
  urls = ["http://influxdb:8086"] 

  database = "iescelia_db" 

  skip_database_creation = true 

  username = "root" 
  password = "root"
````

## InfluxDB 
InfluxDB es un sistema gestor de bases de datos diseñado para almacenar bases de datos de series temporales (TSBD - Time Series Databases). Estas bases de datos se suelen utilizar en aplicaciones de monitorización, donde es necesario almacenar y analizar grandes cantidades de datos con marcas de tiempo, como pueden ser datos de uso de cpu, uso memoria, datos de sensores de IoT, etc.

### Descripción del servicio en docker-compose.yml
docker-compose.yml

````
influxdb: 
    image: influxdb 
    ports:
      - 8086:8086 
    volumes:
      - influxdb_data:/var/lib/influxdb 
    environment:
      - INFLUXDB_DB=iescelia_db 
      - INFLUXDB_ADMIN_USER=root 
      - INFLUXDB_ADMIN_PASSWORD=root 
      - INFLUXDB_HTTP_AUTH_ENABLED=true 
````
### Conectar a InfluxDB desde un terminal
Podemos conectarnos al contenedor de InfluxDB desde un terminal para comprobar si los datos que estamos recogiendo de los sensores se están insertando de forma correcta en la base de datos.

En primer lugar necesitamos obtener el ID del contenedor que se está ejecutando con InfluxDB. Para obtener el listado de todos los contenedores que están en ejecución podemos ejecutar el siguiente comando:

````
docker ps
````
Ahora sólo tenemos que buscar el ID del contenedor con InfluxDB entre la lista de contenedores. En el ejemplo que se muestra a continuación sería el valor 27b06d552719.

````
CONTAINER ID        IMAGE                 COMMAND                  ...
27b06d552719        influxdb              "/entrypoint.sh infl…"   ...
...
````
Una vez que tenemos el ID del contenedor de InfluxDB, creamos un nuevo proceso con el comando /bin/bash dentro del contenedor para obtener un terminal que nos permita interactuar con el contenedor.

````
docker exec -it 27b06d552719 /bin/bash
````
Una vez que tenemos un terminal en el contenedor de InfluxDB, utilizamos el cliente influx para conectarnos al sistema gestor de bases de datos con el usuario y contraseña que hemos creado en el archivo docker-compose.yml.

````
influx -username root -password root
````
Después de autenticarnos, tendremos acceso al shell de InfluxDB donde podemos interaccionar con la base de datos con sentencias InfluxQL (Influx Query Language), que tienen una sintaxis muy similar a SQL.

## Grafana
Grafana es un servicio web que nos permite visualizar en un panel de control los datos almacenados en InfluxDB y otros sistemas gestores de bases de datos de series temporales.
### Descripción del servicio en docker-compose.yml
docker-compose.yml

````
grafana: 
    image: grafana/grafana:7.4.0 
    ports:
      - 3000:3000 
    volumes:
      - grafana_data:/var/lib/grafana 
    depends_on: 
      - influxdb
````
### Configuración de un data source
Grafana permite utilizar diferentes data sources, en nuestro proyecto utilizaremos InfluxDB pero también es posible utilizar AWS CloudWatch, Elasticsearch, MySQL o PostgreSQl entre otros.

#### Configuración de forma manual
Antes de crear un dashboard es necesario crear un data source. Sólo los usuarios que tengan rol de administrador podrán crear un data source.

Para crear un data source debe seguir los siguientes pasos.

1. Accede al menu lateral haciendo clien en el icono de Grafana del encabezado superior.

2. Accede a la sección "Data Sources".

3. Añada un nuevo data source.

4. Seleccione InfluxDB en el desplegable donde se indica el tipo de data source.

5. Configure los parámetros url, database, user y password de su servicio InfluxDB.

En la documentación oficial puede encontrar más información sobre cómo utilizar InfluxDB con Grafana.

#### Configuración con aprovisionamiento automático
Es posible configurar Grafana para que utilize un archivo de configuración de forma automática para evitar tener que realizar la configuración de forma manual.

En nuestro proyecto, vamos a crear un archivo con el nombre datasource.yml dentro de la siguiente estructura de directorios.

````
├── grafana-provisioning
│   └── datasources
│       └── datasource.yml`
````
El contenido del archivo datasource.yml será el siguiente.

````
apiVersion: 1
datasources:
  - name: InfluxDB
    type: influxdb
    access: proxy
    database: iescelia_db 
    user: root 
    password: root 
    url: http://influxdb:8086 
    isDefault: true
    editable: true
````

Para que el aprovisionamiento se realice forma correcta, tenemos que definir un nuevo volumen en el servicio de Grafana en el archivo docker-compose.yml.

docker-compose.yml

````
grafana: 
    image: grafana/grafana:7.4.0 
    ports:
      - 3000:3000 
    volumes:
      - grafana_data:/var/lib/grafana 
      - ./grafana-provisioning/:/etc/grafana/provisioning 
    depends_on: 
      - influxdb
````
### Configuración de un dashboard
#### Configuración con aprovisionamiento automático
Es posible configurar Grafana para que utilize un archivo de configuración de forma automática para evitar tener que realizar la configuración del dashboard de forma manual.

En nuestro proyecto, vamos a crear los archivos dashboard.yml y CO2Dashboard.json dentro de la siguiente estructura de directorios.

````
├── grafana-provisioning
│   ├── dashboards
│   │   ├── CO2Dashboard.json
│   │   └── dashboard.yml
│   └── datasources
│       └── datasource.yml
````
El contenido del archivo dashboard.yml será el siguiente.
````
apiVersion: 1
providers:
- name: InfluxDB
  folder: ''
  type: file
  disableDeletion: false
  editable: true
  options:
    path: /etc/grafana/provisioning/dashboards
````
Un dashboard de Grafana se representa por un objeto JSON, que almacena todos los metadatos del dashboard. Estos metadatos incluyen propiedades de los paneles, variables, etc. El archivo CO2Dashboard.json contiene los metadatos del los paneles que hemos creado para el dashboard de este proyecto.

A continuación se muestra una imagen del dashboard almacenado en el archivo CO2Dashboard.json.
![grafana](images/grafana.PNG)
