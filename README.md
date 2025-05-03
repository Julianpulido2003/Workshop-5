
# Workshop 5 - Sistema IoT de Monitoreo de Temperatura

Este proyecto implementa un sistema de monitoreo de temperatura utilizando un microcontrolador **ESP32** y una red de **comunicación I2C** con **Arduino UNO** (master y slave). La temperatura es medida mediante un sensor analógico (TMP36), procesada y visualizada en ThingSpeak a través de la nube.

---

## Tecnologías Utilizadas

- ESP32 (simulado en [Wokwi](https://wokwi.com/))
- ThingSpeak para visualización IoT
- Arduino UNO (master y slave) usando comunicación I2C
- Sensor de temperatura TMP36 
- LCD 16x2
- TinkerCAD para simulación de Arduinos

---

##  Lógica del Proyecto

### 1. Comunicación I2C entre Arduinos

Se utilizaron dos placas Arduino simuladas en TinkerCAD, conectadas vía **I2C**, donde:

- **El Arduino slave** lee el valor analógico del sensor TMP36 desde el pin A0.
- **El Arduino master** solicita los datos por I2C al slave, calcula la temperatura en °C, la muestra en una pantalla LCD 16x2 y activa un LED de alerta si la temperatura supera los 30 °C.

#### Conexiones I2C entre Arduinos

| Señal | Master | Slave |
|-------|--------|-------|
| SDA   | A4     | A4    |
| SCL   | A5     | A5    |
| GND   | GND    | GND   |

---
![image](https://github.com/user-attachments/assets/32391510-06e4-4078-ab45-773d436cc62b)

## Diagrama de funcionalidad
![image](https://github.com/user-attachments/assets/ef3d3319-c3ae-447d-b91b-21b71f6d8175)

##  Códigos Documentados


### Arduino Slave

```cpp
#include <Wire.h>

// Dirección del esclavo I2C (decimal 8 = 0x08)
const uint8_t SLAVE_ADDR = 0x08;
// Pin de entrada analógica para el sensor TMP36 o LM35
const int TMP_PIN = A0;

// Variable global para almacenar la última lectura del sensor (0-1023)
volatile uint16_t adcRaw = 0;
// Variable para controlar el tiempo de muestreo
unsigned long tStamp = 0;

void setup() {
  Wire.begin(SLAVE_ADDR);           // Configura el Arduino como esclavo I2C
  Wire.onRequest(sendData);         // Callback al recibir solicitud del master
  Serial.begin(9600);
}

void loop() {
  // Lee el sensor cada 1 segundo
  if (millis() - tStamp >= 1000) {
    adcRaw = analogRead(TMP_PIN);   // Lectura del sensor
    Serial.print("ADC = ");
    Serial.println(adcRaw);         // Muestra la lectura en el monitor serial
    tStamp = millis();
  }
}

// Envía los datos al master en formato de 2 bytes (MSB primero)
void sendData() {
  Wire.write(highByte(adcRaw));
  Wire.write(lowByte(adcRaw));
}
```

---

### Arduino Master

```cpp
#include <Wire.h>
#include <LiquidCrystal.h>

// Dirección I2C del esclavo
const uint8_t SLAVE_ADDR = 0x08;
const int LED_PIN = 13;                 // LED de alerta si T > 30 °C

// Configuración del LCD: RS,E,D4,D5,D6,D7
LiquidCrystal lcd(7, 6, 5, 4, 3, 2);

void setup() {
  Wire.begin();                         // Configura como master
  Serial.begin(9600);
  pinMode(LED_PIN, OUTPUT);
  lcd.begin(16, 2);                     // LCD 16x2
  lcd.print("TEMP (C):");
}

void loop() {
  // Solicita dos bytes al esclavo
  Wire.requestFrom(SLAVE_ADDR, (uint8_t)2);

  if (Wire.available() == 2) {
    // Combina los dos bytes recibidos
    uint16_t adcRaw = (Wire.read() << 8) | Wire.read();
    // Conversión a temperatura en °C
    float celsius = ((((adcRaw * 5.0) / 1024.0) * 1000.0) - 500.0) / 10.0;

    // Muestra por monitor serial
    Serial.print("ADC: ");
    Serial.print(adcRaw);
    Serial.print("  T: ");
    Serial.println(celsius, 1);

    // Muestra en LCD
    lcd.setCursor(0, 1);
    lcd.print("                ");  // Limpia la línea
    lcd.setCursor(0, 1);
    lcd.print(celsius, 1);

    // Activa LED si la temperatura supera 30 °C
    digitalWrite(LED_PIN, (celsius > 30.0) ? HIGH : LOW);
  }

  delay(1000);  // Espera 1 segundo
}
```

---

###  ESP32 + ThingSpeak (en Wokwi)

```cpp
// ***********************************************
// Configuración de red y API Key de ThingSpeak
// ***********************************************

const char* WIFI_SSID = "Wokwi-GUEST";         // Nombre de la red WiFi simulada en Wokwi
const char* WIFI_PASS = "";                    // Contraseña de la red (vacía en Wokwi)
const char* API_KEY   = "NWP8EHDC05BK7TVD";    // Clave de escritura de tu canal en ThingSpeak

#define SIM_MODE      1    // Modo simulación activado: 1 = usa temperatura simulada; 0 = lectura real vía I2C

// ***********************************************
// Librerías necesarias
// ***********************************************

#include <WiFi.h>             // Para conexión WiFi
#include <HTTPClient.h>       // Para enviar solicitudes HTTP
#include <Wire.h>             // Para comunicación I2C (si se usa Arduino esclavo)

// ***********************************************
// Variables de configuración
// ***********************************************

const uint8_t SLAVE_ADDR = 0x08;   // Dirección I2C del Arduino esclavo
const int LED_PIN = 2;             // Pin del LED que se enciende si la temperatura > 30°C

// ***********************************************
// Setup inicial
// ***********************************************

void setup() {
  Serial.begin(115200);               // Inicializa el puerto serie
  pinMode(LED_PIN, OUTPUT);          // Configura el pin del LED como salida

  // Configuración del bus I2C (si se usa)
  Wire.begin(21, 22, 400000);        // SDA = GPIO21, SCL = GPIO22, velocidad = 400kHz

  // Conexión a la red WiFi
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  Serial.print("Conectando WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    Serial.print('.');
    delay(500);
  }
  Serial.println("  ¡OK!");
}

// ***********************************************
// Bucle principal
// ***********************************************

void loop() {
  float tempC = leerTemperatura();          // Obtiene la temperatura en °C

  if (isnan(tempC)) {                       // Si la lectura falló (NaN), reintenta luego
    delay(5000);
    return;
  }

  // Enciende el LED si la temperatura supera los 30°C
  digitalWrite(LED_PIN, tempC > 30.0);

  // Envío de datos a ThingSpeak vía HTTP GET
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    String url = "https://api.thingspeak.com/update?api_key=";
    url += API_KEY;
    url += "&field1=";
    url += String(tempC, 1);   // Solo 1 decimal

    http.begin(url);                  // Inicializa la conexión
    int rc = http.GET();             // Ejecuta el GET
    Serial.printf("TS rc=%d  T=%.1f °C\n", rc, tempC);
    http.end();                      // Finaliza la conexión
  } else {
    Serial.println("WiFi caída, re-intentando...");
  }

  delay(15000);   // Espera 15 segundos (mínimo entre publicaciones en ThingSpeak)
}

// ***********************************************
// Función para obtener la temperatura
// ***********************************************

float leerTemperatura() {
#if SIM_MODE
  // Modo simulación: genera valores entre 25°C y 35°C como onda triangular
  static float v = 25;
  static int dir = 1;
  v += 0.6 * dir;
  if (v > 35 || v < 25) dir *= -1;
  return v;

#else
  // Lectura real vía I2C desde Arduino esclavo
  Wire.requestFrom(SLAVE_ADDR, (uint8_t)2);
  if (Wire.available() == 2) {
    uint16_t adc = (Wire.read() << 8) | Wire.read();  // Lee 2 bytes (16 bits)
    
    // Convierte la señal analógica del TMP36 a grados Celsius
    return ((((adc * 5.0) / 1024.0) * 1000.0) - 500.0) / 10.0;
  }
  Serial.println("I²C vacío");   // Error de lectura
  return NAN;
#endif
}
```

---

##  Visualización en ThingSpeak

Los datos enviados desde el ESP32 son registrados y graficados automáticamente en el dashboard de ThingSpeak.

![image](https://github.com/user-attachments/assets/cf16d346-e257-4225-a28d-1f96c3c53c39)


- **Field 1 Chart**: Muestra cómo varía la temperatura a lo largo del tiempo.
- **Temp**: Muestra la última lectura registrada.
- **Alerta**: Indicador visual que se activa si la temperatura supera los 30 °C (círculo rojo).

---

##  Resultados y Conclusión

Este sistema permite integrar sensores físicos con plataformas IoT para la visualización y análisis remoto de datos en tiempo real. Se probaron con éxito:

- La lectura y transmisión de temperatura por I2C entre dos Arduinos
- La simulación del sensor en ESP32 con Wokwi
- La conexión y visualización de datos en ThingSpeak

La implementación permite generar alertas visuales locales (LED) y remotas (dashboard IoT), ofreciendo una solución efectiva para monitoreo ambiental o de procesos industriales.

---
## Referencias:
[1] OpenAI, “ChatGPT,” ChatGPT, [En línea]. Disponible en: https://chat.openai.com/. [Accedido: 2-may-2025].

##  Autores y Acta de Reunión

**Julián Pulido**  
Wiki y conexión thingspeak

**Juan Diego García**  
Simulación thinkercad y wokwi

ACTA DE REUNIÓN – Proyecto IoT: Monitoreo de Temperatura con ESP32 y ThingSpeak

Fecha de primera sesión: Lunes 28 de abril de 2025
Hora: 7:00 a.m. – 9:00 a.m.
Lugar: Universidad de La Sabana – Sala de clase / Virtual TinkerCad & Wokwi
Participantes:
	•	Julián Pulido
	•	Juan Diego García
	•	Profesor guía: [Nombre del profesor, si aplica]

⸻

Objetivo de la reunión

Desarrollar el sistema de monitoreo de temperatura basado en ESP32, simulación con Wokwi y diseño estructural del sistema en TinkerCad, para avanzar en el proyecto de IoT propuesto.

⸻


⸻

📅 Segunda reunión (cierre)

Fecha: Viernes 2 de mayo de 2025
Hora: 7:00 p.m. – 8:00 p.m.
Objetivo: Finalizar documentación, subir la Wiki, y alistar informe final con capturas, código comentado y evidencias gráficas.

⸻
<img width="652" alt="image" src="https://github.com/user-attachments/assets/bba28a23-7075-41e4-ac54-6146fd771a1c" />

\


