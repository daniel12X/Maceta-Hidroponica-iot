# Maceta-Hidroponica-iot
Este proyecto de maceta hidropónica con IoT utiliza sensores para medir humedad, temperatura y nutrientes, optimizando el cultivo de plantas. Con un sistema de procesamiento de datos, busca promover la agricultura urbana sostenible, con beneficios económicos, sociales y ambientales.

## Codigo-funcionando

#include <WiFi.h>
#include <ThingSpeak.h>
#include <OneWire.h>
#include <DallasTemperature.h>

// Definir los pines y constantes
#define ONE_WIRE_BUS 5   // Pin GPIO para el sensor de temperatura DS18B20
const int trigPin = 19;  // Pin trig del sensor ultrasónico
const int echoPin = 18;  // Pin echo del sensor ultrasónico
const int uvPin = 39;    // Pin ADC del sensor UV

// Configurar el objeto OneWire y DallasTemperature
OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);

// Datos de la red Wi-Fi
const char* ssid = "???";            // Reemplaza por el nombre de tu red Wi-Fi
const char* password = "Daniel12X";  // Reemplaza por la contraseña de tu Wi-Fi

// Datos de ThingSpeak
unsigned long myChannelNumber = 2672642;          // Reemplaza con tu número de canal
const char* myWriteAPIKey = "6F29LVBGQKVWCV5X";  // Reemplaza con tu clave API de ThingSpeak

// Cliente WiFi y ThingSpeak
WiFiClient espClient;

// Constantes para el vaso cónico
const float vasoAltura = 9.5;         // Altura total del vaso en cm
const float vasoRadioSuperior = 4.0;  // Radio superior en cm
const float vasoRadioInferior = 2.5;  // Radio inferior en cm

// Función para conectar al Wi-Fi
void setup_wifi() {
  delay(10);
  Serial.println();
  Serial.print("Conectando a ");
  Serial.println(ssid);

  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println("");
  Serial.println("WiFi conectado");
  Serial.println("Dirección IP: ");
  Serial.println(WiFi.localIP());
}

// Función para leer la distancia del sensor ultrasónico
long readUltrasonicDistance() {
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);

  long duration = pulseIn(echoPin, HIGH);
  float distance = (duration * 0.0343) / 2;
  return distance;
}

// Función para calcular el promedio de varias lecturas de distancia
float getPreciseDistance(int samples = 3) {
  float sum = 0;
  for (int i = 0; i < samples; i++) {
    sum += readUltrasonicDistance();
    delay(50);  // Pausa entre lecturas para mayor precisión
  }
  return sum / samples;
}

// Función para calcular el volumen de agua basado en el nivel medido
float calcularVolumenAgua(float nivelAgua) {
  if (nivelAgua < 0) nivelAgua = 0;
  float radioEnNivelAgua = vasoRadioInferior + ((vasoRadioSuperior - vasoRadioInferior) * (nivelAgua / vasoAltura));
  float volumenAgua = (1.0 / 3.0) * 3.1416 * nivelAgua * (vasoRadioSuperior * vasoRadioSuperior + vasoRadioSuperior * radioEnNivelAgua + radioEnNivelAgua * radioEnNivelAgua);
  if (volumenAgua > 266.16) volumenAgua = 266.16;  // Limitar al volumen máximo del vaso
  return volumenAgua;
}

// Función para obtener el índice UV
float getUVIndex() {
  int uvValue = analogRead(uvPin);
  float voltage = (uvValue / 4095.0) * 3.3;
  float uvIndex = voltage * 3.0;  // Aproximadamente 1V = UV Index 1, 3V = UV Index 3
  return uvIndex;
}

// Setup
void setup() {
  Serial.begin(115200);

  pinMode(trigPin, OUTPUT);
  pinMode(echoPin, INPUT);

  setup_wifi();  // Conectar a la red Wi-Fi
  sensors.begin();  // Iniciar el sensor de temperatura
  ThingSpeak.begin(espClient);  // Iniciar ThingSpeak
  analogReadResolution(12);  // Configurar la resolución del ADC
  analogSetAttenuation(ADC_11db);  // Para el rango de voltaje de 0 a 3.3V
}

// Loop
void loop() {
  // --- Leer temperatura ---
  sensors.requestTemperatures();
  float temperatureC = sensors.getTempCByIndex(0);

  // --- Publicar temperatura en Field 1 ---
  ThingSpeak.setField(1, temperatureC);

  // --- Leer nivel de agua (3 mediciones promedio) ---
  float distance = getPreciseDistance();  // Se realizan 3 mediciones y se promedia
  float nivelAgua = vasoAltura - distance - 0.50;  // Se ajusta el valor del nivel de agua
  float volumenAgua = calcularVolumenAgua(nivelAgua);

  // --- Publicar nivel de agua en Field 2 ---
  ThingSpeak.setField(2, volumenAgua);

  // --- Leer índice UV ---
  float uvIndex = getUVIndex();

  // --- Publicar índice UV en Field 3 ---
  ThingSpeak.setField(3, uvIndex);

  // --- Enviar los datos a ThingSpeak ---
  int httpCode = ThingSpeak.writeFields(myChannel
  --}}}--Number, myWriteAPIKey);
  if (httpCode == 200) {
    Serial.println("Datos enviados correctamente.");
  } else {
    Serial.println("Error al enviar datos. Código: " + String(httpCode));
  }

  // Esperar 15 segundos antes de la siguiente lectura para respetar el límite de ThingSpeak
  delay(15000);
}

