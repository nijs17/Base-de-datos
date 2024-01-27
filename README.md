# Base-de-datos
## INTRODUCCION

### DESCRIPCION  
.
La **Esp32**  es un dispositivo electronico en el cual podemos realizar practicas y simulaciones, en esta practica realizaremos   una base de datos  en **XAMPP**, esto entrelasando nuestra tarjeta y  **Node-red**, esta accion nos permitira visualizar en tiempo real nuestras graficas de acuerdo a los valores detectados por nuestros sensores, ademas de llevar un registro de estos datos. Cabe aclarar que para esta practica se usara un simulador llamado **https://wokwi.com/**.

### MATERIAL A OCUPAR

Para realizar esta practica ocuparemos lo siguente:

-WORKI

-XAMPP

-Tarjeta ESP 32

-DTH22

- 1 sensor ultrasonico

-Node red

-Un servidor virtual que nos genere una IP

### Requisitos previos

Para poder realizar esta practica es nesesario contar con conosimientos basicos de programacion y electronica.

## INSTRUCIOES DE ELAVORACION 

1.Buscar un servidor virtual y generar una IP:

 a) Entraremos al siguiente link **(https://www.hivemq.com/mqtt/public-mqtt-broker/)** y copiaremos la direccion señalada
 ![](https://github.com/nijs17/Base-de-datos/blob/main/W1.png)
 b)Abriremos el simbolo del sistema y escribimos **nslookup**  seguido del link anteriormente copiado **broker.hivemq.com** esta accion nos generara una IP, copiamos esa IP
 ![](https://github.com/nijs17/Base-de-datos/blob/main/W2.png)
 c)Prosederemos a pegar el ip como se muestra en la imagen 1 y poner el nombre de nuestro archivo en node-red a enlazar copmo se muestra en la imagen 2.
 ![](https://github.com/nijs17/Base-de-datos/blob/main/X.png)
 
2. Abrir la terminal de programación y colocar el siguente codigo:
 
#include <ArduinoJson.h>
#include <WiFi.h>
#include <PubSubClient.h>
#define BUILTIN_LED 2
#include "DHTesp.h"
const int Ap = 12;    // Pin para A+
const int Am = 13;   // Pin para A-
const int Bp = 14;    // Pin para B+
const int Bm = 27;   // Pin para B-
volatile int velocidad; //Variable de velocidad
volatile int dt;    //Variable de Delay
int L;    //Variable para el calculo de litros
int Humedad;
const int DHT_PIN = 15;
const int Trigger = 4;   //Pin digital 2 para el Trigger del sensor
const int Echo = 2;   //Pin digital 3 para el Echo del sensor
DHTesp dhtSensor;
// Update these with values suitable for your network.

const char* ssid = "Wokwi-GUEST";
const char* password = "";
const char* mqtt_server = "3.65.168.153";
String username_mqtt="educatronicosiot";
String password_mqtt="12345678";

WiFiClient espClient;
PubSubClient client(espClient);
unsigned long lastMsg = 0;
#define MSG_BUFFER_SIZE  (50)
char msg[MSG_BUFFER_SIZE];
int value = 0;

void setup_wifi() {

  delay(10);
  // We start by connecting to a WiFi network
  Serial.println();
  Serial.print("Connecting to ");
  Serial.println(ssid);

  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  randomSeed(micros());

  Serial.println("");
  Serial.println("WiFi connected");
  Serial.println("IP address: ");
  Serial.println(WiFi.localIP());
}

void callback(char* topic, byte* payload, unsigned int length) {
  Serial.print("Message arrived [");
  Serial.print(topic);
  Serial.print("] ");
  for (int i = 0; i < length; i++) {
    Serial.print((char)payload[i]);
  }
  Serial.println();

  // Switch on the LED if an 1 was received as first character
  if ((char)payload[0] == '1') {
    digitalWrite(BUILTIN_LED, LOW);   
    // Turn the LED on (Note that LOW is the voltage level
    // but actually the LED is on; this is because
    // it is active low on the ESP-01)
  } else {
    digitalWrite(BUILTIN_LED, HIGH);  
    // Turn the LED off by making the voltage HIGH
  }

}

void reconnect() {
  // Loop until we're reconnected
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    // Create a random client ID
    String clientId = "ESP8266Client-";
    clientId += String(random(0xffff), HEX);
    // Attempt to connect
    if (client.connect(clientId.c_str(), username_mqtt.c_str() , password_mqtt.c_str())) {
      Serial.println("connected");
      // Once connected, publish an announcement...
      client.publish("outTopic", "hello world");
      // ... and resubscribe
      client.subscribe("inTopic");
    } else {
      Serial.print("failed, rc=");
      Serial.print(client.state());
      Serial.println(" try again in 5 seconds");
      // Wait 5 seconds before retrying
      delay(5000);
    }
  }
}

void setup() {
  pinMode(BUILTIN_LED, OUTPUT);     // Initialize the BUILTIN_LED pin as an output
  Serial.begin(115200);
  setup_wifi();
  client.setServer(mqtt_server, 1883);
  client.setCallback(callback);
  dhtSensor.setup(DHT_PIN, DHTesp::DHT22);
  pinMode(Trigger, OUTPUT); //pin como salida
  pinMode(Echo, INPUT);  //pin como entrada
  digitalWrite(Trigger, LOW);//Inicializamos el pin con 0

  pinMode(Ap, OUTPUT);  //Pin como salida
  pinMode(Am, OUTPUT);  //Pin como salida
  pinMode(Bp, OUTPUT);  //Pin como salida
  pinMode(Bm, OUTPUT);  //Pin como salida
  velocidad = 100;  //Velocidad de la bomba
  dt=(600/(4*abs(velocidad)));  //Delay entre cada paso
  Serial.println(velocidad);
}

void loop() {


delay(1000);
TempAndHumidity  data = dhtSensor.getTempAndHumidity();
Humedad = data.humidity;
long t; //timepo que demora en llegar el eco
long d; //distancia en centimetros

digitalWrite(Trigger, HIGH);
delayMicroseconds(10);          //Enviamos un pulso de 10us
digitalWrite(Trigger, LOW);
  
t = pulseIn(Echo, HIGH); //obtenemos el ancho del pulso
d = t/59;             //escalamos el tiempo a una distancia en cm
L = ((400-(d))*2000*2000)/1000000; //Conversión de los valores de "d" a Litros

if (Humedad>=0 && Humedad<=50){   //Condición para accionamiento de bomba de agua              
  if (d>=0 && d<=350){    //Valores de niel de agua entre los que la bomba de agua funcionara
    delay(dt);
    digitalWrite(Bm,LOW) ; digitalWrite(Ap,HIGH);
    delay(dt);
    digitalWrite(Ap,LOW) ; digitalWrite(Bp,HIGH);
    delay(dt);
    digitalWrite(Bp,LOW) ; digitalWrite(Am,HIGH);
    delay(dt);
    digitalWrite(Am,LOW) ; digitalWrite(Bm,HIGH);
  }
}
else if (Humedad=100){    //A un 100% de humedad la bomba de agua se apagará
  digitalWrite(Bm,LOW) ; digitalWrite(Ap,LOW);
  digitalWrite(Ap,LOW) ; digitalWrite(Bp,LOW);
  digitalWrite(Bp,LOW) ; digitalWrite(Am,LOW);
  digitalWrite(Am,LOW) ; digitalWrite(Bm,LOW);
}
  
Serial.print("Distancia: ");
Serial.print(d);      //Enviamos serialmente el valor de la distancia
Serial.print("cm");
Serial.println();
Serial.print("Nivel: ");
Serial.print(L);      //Enviamos serialmente el valor de los litros
Serial.print("L");
Serial.println();
  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  unsigned long now = millis();
  if (now - lastMsg > 2000) {
    lastMsg = now;
    //++value;
    //snprintf (msg, MSG_BUFFER_SIZE, "hello world #%ld", value);

    StaticJsonDocument<128> doc;

    doc["DEVICE"] = "ESP32";
    //doc["Anho"] = 2022;
    doc["LITROS"] = String(L);
    doc["TEMPERATURA"] = String(data.temperature, 1);
    doc["HUMEDAD"] = String(data.humidity, 1);
   

    String output;
    
    serializeJson(doc, output);

    Serial.print("Publish message: ");
    Serial.println(output);
    Serial.println(output.c_str());
    client.publish("SistemaRiego", output.c_str());
  }
}
 
 
3. Hacer las conexiones de la tarjeta con los demas elementos, anteriormente mensionados.
 ![](https://github.com/nijs17/Base-de-datos/blob/main/x1.png)
 4. Armar el proyecto en node red.
    
 Acontinuacion se muestran los cambios que se le tiene que hacer al proyecto en:
 
**mqtt out:** pegar la IP y cambiar el nombre al topico

**json:** Elegir al leguaje de programacion que se va a convertir

**function:** programar las funciones 

**gauge y char:t** editar los valores de los indicadores y sensores como se muestra en la imagen.
 ![](https://github.com/nijs17/Base-de-datos/blob/main/x2.png)

 5. Una vez teniendo el proyecto descargar desde node-red el nodo llamado **node-red-node-mysql** crear una funsion llamada **MySQL** y tomar un nodo MySQL y nombrarlo como llamaremos a nuestra base de datos:
     ![](https://github.com/nijs17/Base-de-datos/blob/main/x3.png)
6. Configurar la funcion y el nodo nuevos de la siguiente manera, para que nos permita conectarse con la base de datos y el WOKWI:
  ![](https://github.com/nijs17/Base-de-datos/blob/main/x4.png)
7. Ejecutar como administrador el programa de **XAMPP**, correr **Apache** y **MySQL**
   ![](https://github.com/nijs17/Base-de-datos/blob/main/X5.png)
8.Crear una nueva base de datos con el nombre seleccionado anteriormente, configurar la base de datos para 5 columnas e introducir los valores que se muestran en la siguente imagen:
 ![](https://github.com/nijs17/Base-de-datos/blob/main/x6.png)
9. Dar clik en examinar y visualizar la tabla de dtos en tiempo real
     ![](https://github.com/nijs17/Base-de-datos/blob/main/x7.png)
## INSTRUCCIONES DE OPERACION 


    1. Iniciar simulador.
    
    2. Realizar las conexiones mostradas en la imagen anterior.
    
    3. Mover la sencibilidad del sensor para poder visualizar los resultados de manera grafica.

    4. Abri XAMPP y visuakizar la base de datos
    
## RESUSLTADOS
Al finalizar tendremos nuestra tarjeta conectada con nuestro servidor en linea y podremos visualizar en tiempo real los datos de los sensores, en la base de datos.
