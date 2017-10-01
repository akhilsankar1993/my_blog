---
title: "Setting Up the NodeMCU"
date: 2017-10-01T14:11:52-04:00
draft: false
---

#### The Project

This is a robot that would allow me to track the moisture level of the soil in my baby tomato plant and pump water to it. I wanted the system to have a WiFi accessible API to be able to send data on the real time moisture level of the soil. I also wanted it to be able to accept JSON on the desired moisture level of the soil.

This is a pretty basic project for now, but I want to be able to connect it to other projects soon.

Alright, enough talk! Lets build!

_Quick disclaimer: This is a writeup just to document the project as it worked for me. If you question the validity/safety of these steps, I encourage you to research them on your own before moving forward. Please use this resource at your own risk._

#### What You need

- [NodeMCU dev board (you'll need a microUSB wire too)](https://www.amazon.com/gp/product/B010O1G1ES/ref=oh_aui_detailpage_o07_s00?ie=UTF8&psc=1)
- [5V water pump](https://www.amazon.com/gp/product/B00JWJIC0K/ref=oh_aui_detailpage_o00_s00?ie=UTF8&psc=1)
- [9V battery clips](https://www.amazon.com/gp/product/B071D98X7K/ref=oh_aui_detailpage_o00_s00?ie=UTF8&psc=1)
- [2N2222 NPN transistor](https://www.amazon.com/Gikfun-Transistor-2N2222A-2N2222-Arduino/dp/B01J7P2XM6/ref=sr_1_1?s=electronics&ie=UTF8&qid=1506886819&sr=1-1&keywords=2n2222+transistor)
- [Jumper wire](https://www.amazon.com/GenBasic-Solderless-Ribbon-Breadboard-Prototyping/dp/B01L5UJ36U/ref=sr_1_1?s=electronics&ie=UTF8&qid=1506886901&sr=1-1&keywords=jumper+wire+male+to+male)
- [Breadboard](https://www.amazon.com/Qunqi-point-Experiment-Breadboard-5-5%C3%978-2%C3%970-85cm/dp/B0135IQ0ZC/ref=sr_1_2_sspa?s=electronics&ie=UTF8&qid=1506887010&sr=1-2-spons&keywords=breadboard&psc=1)
- 150 and 10K ohm resistors

#### Setting Up the Hardware

To make it a little easier to follow, I'll split the hardware into two diagrams, one to power the water pump and one to set up the soil moisture sensor.

This is the circuit to set up the water pump. I forgot to update the resistor in this schematic but I actually used a 150 ohm resistor between the dev board and the base of the transistor:

![Water Pump Circuit](/images/setting_up_nodemcu/power_circuit.png)

This is the circuit to set up the soil moisture. Once again, I actually used a 10K ohm resistor here:

![Soil Moisture Circuit](/images/setting_up_nodemcu/moisture_circuit.png)

You'll need to strip the insulation off the ends of the yellow wiring to get readings in the soil. I'd also recommending soldering galvanized nails to the ends of these wires if you have a soldering iron handy. That'll get you optimal readings.

#### Setting Up the NodeMCU

I was trying to setup the NodeMCU to work on OSX Sierra, which gave me a couple of issues. This is the process I had to run to get the Arduino environment correctly configured to flash to the dev board.

On the Arduino environment, you'll need to go to _Preferences -> Additional Boards Manager URLs -> and add_ `http://arduino.esp8266.com/stable/package_esp8266com_index.json`

Then go to _Tools->Boards Manager->_, search for and install the `esp8266` package (I'm using 2.3.0).

Now you should be able to go to _Tools->Board_ and add `NodeMCU 1.0(ESP-12E Module)`

If you now plug in your dev board, you might notice that the _Tools->Ports_ options are something like:
```
/dev/cu.lpss-serial1
/dev/cu.lpss-serial2
/dev/cu.Bluetooth-Incoming-Port
etc.
```

Apparently, the NodeMCU ships with CH340G USB/Serial ports, which aren't supported by most operating systems. [James Best has a good blog post on this](http://justjibba.net/getting-the-nodemcu-board-to-work-on-mac-os-10-12/).

That solution didn't exactly work for me but it might work for you. For me, I wasn't seeing the port that I actually I needed, and I had install the [CP210X USB to UART Bridge](https://www.silabs.com/products/development-tools/software/usb-to-uart-bridge-vcp-drivers) to get the correct port.  

Finally, my selected port turned out to be `/dev/cu.SLAB_USBtoUART` but yours might look a bit different.

#### The Software

For now, I'll just post the software that I used. This is more like boilerplate code just to get the basic endpoints working. I definitely need to refactor a lot of it, but I'm feeling lazy right now. I'll refactor this and walk through it line by line in another post.

```c

#include <ESP8266WiFi.h>
#include <ESP8266WebServer.h>
#include <ESP8266mDNS.h>
#include <WiFiClient.h>
#include <ArduinoJson.h>

const char* ssid = "{{wifi_network_name}}";
const char* password = "{{wifi_password}}";
ESP8266WebServer server ( 80 );

const int pumpPin = D7;
const int moistureSensorPin = A0;

void setup( void ) {
  pinMode(pumpPin, OUTPUT);

  Serial.begin ( 9600 );
  WiFi.begin ( ssid, password );
  Serial.println ( "" );

  while ( WiFi.status() != WL_CONNECTED ) {
    delay ( 500 );
    Serial.print ( "." );
  }

  Serial.println ( "" );
  Serial.print ( "Connected to " );
  Serial.println ( ssid );
  Serial.print ( "IP address: " );
  Serial.println ( WiFi.localIP() );

  if ( MDNS.begin ( "esp8266" ) ) {
    Serial.println ( "MDNS responder started" );
  }

  server.on( "/test", testPump);
  server.on( "/pump_water", pumpWater);
  server.on( "/moisture_level", getMoistureLevel);

  server.begin();
}

void loop(void) {
  server.handleClient();
}

void testPump() {
  Serial.println("gets to function");
  digitalWrite(pumpPin, HIGH);
  delay(2000);
  digitalWrite(pumpPin, LOW);
  server.send (200, "text/html", "Pump test successful.");
}

void getMoistureLevel() {
  Serial.println("measuring moisture level:");
  int moistureVoltage = analogRead(moistureSensorPin);
  Serial.println(moistureVoltage);
}

void pumpWater() {
  StaticJsonBuffer<200> jsonBuffer;
  JsonObject& root = jsonBuffer.parseObject(server.arg("plain"));
  int desiredMoistureLevel = root["desiredMoistureLevel"];

  int currentMoistureLevel = 1;
  while (currentMoistureLevel < desiredMoistureLevel) {
    digitalWrite(pumpPin, HIGH);
    // currentMoistureLevel = digitalRead(D7);
    Serial.println("Current moisture Level: ");
    Serial.println(currentMoistureLevel);
    currentMoistureLevel++;
    delay(1000);
  }
  digitalWrite(pumpPin, LOW);
  server.send(200, "text/plain", "Pump complete");
}

void postFunction() {
  StaticJsonBuffer<200> jsonBuffer;
  JsonObject& root = jsonBuffer.parseObject(server.arg("plain"));
  int desiredMoistureLevel = root["desiredMoistureLevel"];
  int pumpStatus = root["pumpStatus"];
  Serial.print("Desired Moisture Level: ");
  Serial.println(desiredMoistureLevel);
  Serial.print("Pump Status: ");
  Serial.println(pumpStatus);
}
```

You can _Upload_ this code into your dev board and immediately see the connection to the wifi network made on the Serial Monitor (the magnifying glass icon on the top right of the Arduino IDE).

#### Hitting the Endpoints

You can then use the Serial Monitor to view the data transmitted to and from the Dev Board and your client (Postman, cURL, etc).

I used Postman to make API calls to the dev board server by its IP.

These were my API calls:

```
GET `:dev_board_ip_address/moisture_level`

Headers:

Content-Type: application/json
Content-Length: 20
```

```
POST `:dev_board_ip_address/pump_water`

Headers:

Content-Type: application/json
Content-Length: 20

Body (raw JSON):

{
  "desiredMoistureLevel": "5"
}
```

#### You're Done!

Hopefully, this helped shed some light on how to setup the NodeMCU and make calls to it. In my next post I'll need to document how to expose this API outside the local network.

Thanks for reading!
