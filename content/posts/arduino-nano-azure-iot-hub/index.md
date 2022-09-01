---
title: "Upload sensors data from Arduino nano RP2040 to Azure IoT Hub"
date: 2022-08-23
categories:
- Arduino
- azure
author: Hugo Alves
resources:
- name: "featured-image"
  src: "images/banner.jpg"
- name: featured-image-preview
  src: images/banner.jpg
subtitle: Build a room temperature dashboard using Azure and Arduino Nano.
tags:
- Arduino
- azure
- iot
---

In this series of tutorials, we will build a room temperature dashboard with the help of Arduino and Azure. For measuring the temperature, we will use a temperature sensor connected to the Arduino nano board. Then, Arduino will periodically send the temperature values to Azure IoT Hub using the azure sdk c for Arduino and the mqtt protocol. Finally, we will transform this stream of data using Azure stream analytics and create some nice dashboards using Azure Data Explorer, also deployed in Azure.

We will start with only one temperature sensor to simplify the process, later I will add also a light sensor to have more data to play with, but feel free to adapt the code to add more/new sensors. Or even use some mocked data if you don't have any sensor available.\
\
In the end the solution should resemble something like this:
{{< image src="images/flow_diagram.jpg" alt="upload temperature sensor data to azure" height="auto"  >}}

{{< admonition type=tip title="" >}}
All the necessary Arduino code mentioned in this tutorial is available in this [Github repo](https://github.com/hugo-ma-alves/Arduino-nanorp2040-azure-iot-hub-demo).
{{< /admonition >}}


## Required Hardware

 - Arduino, I used the [nano RP2040](https://docs.Arduino.cc/hardware/nano-rp2040-connect)
 - [Grove Shield](https://www.seeedstudio.com/Grove-Shield-for-Arduino-Nano-p-4112.html) - Optional
 - [Temperature sensor](https://www.seeedstudio.com/Grove-Temperature-Sensor.html) - Or any other sensor
 - Azure account - All the necessary tools should have no cost under the free tier

{{< image src="images/material.jpg" alt="Arduino nano rp2040" height="auto" caption="Arduino nano, grove shield and temperature sensor"  >}}

{{< admonition type=warning title="Important information if you are using the Arduino nano RP2040 with the grove shield" open=true >}}
As you can check [here](https://forum.seeedstudio.com/t/grove-shield-for-Arduino-nano-v1-0-can-it-be-used-with-a-nano-rp2040/258699/4) there is an incompatibility between the grove shield and the Arduino nano RP2040.
The Grove shield was designed for the Arduino nano 33 (and its variants), although being very similar the pinout is not the same on the nano RP2040. The RP2040 only has one RESET PIN, the other is used for the REC (BOOTSELECT). The GROVE shield have these two pins connected, on the nano 33 it works correctly because they have the same function. But if you try to use this shield on the RP2040 you won't be able to connect the Arduino to the pc, it won't be recognized as a device. So as suggested on the previous link you can just cut the grove shield connection between the 2 RESET pins, you don't have to drill, a sharp knife is enough to break the link.

[Nano 33 Pinout](https://content.Arduino.cc/assets/Pinout-NANOble_latest.pdf)

[Nano RP2040](https://docs.Arduino.cc/static/97fbbd7b8b0d0f12efd7fc3e3cd5de35/ABX00053-full-pinout.pdf)

[Grove shield remove RESET connection link](images/grove_shield_nano_rp2040.jpg)
{{< /admonition >}}


## Hardware setup

If you are using the grove shield the connections are quite easy to do. Just put the Arduino in the grove shield and connect it to the usb port.
Then connect the sensor using the grove cable to the A0 pin.  
If you aren't using the grove shield, or if you are using another sensor that doesn't have the grove connection, just connect the sensor VCC and GND port to the corresponding Arduino ports and the the signal cable to the A0 pin.  

{{< admonition type=tip title="" >}}
The code I provide in this tutorial is adapted to the seedstudio temperature sensor. If you use other sensor you may have to adapt the code to correctly read its value.
{{< /admonition >}}

{{< admonition type=warning title="" >}}
Don't forget to move the VCC switch of the grove shield to 3.3V if you are using a Arduino nano (all models).
{{< /admonition >}}

{{< image src="images/connections.jpg" alt="connections" height="auto" caption="Arduino nano rp2040 and seeedstudio temperature sensor">}}

## Some theory before writing code

{{< admonition type=tip >}}
This section  gives a brief explanation about the math behind the code that we will implement later. Since the focus of this tutorial is the azure Iot, it isn't critical to understand this section, you can jump directly to the next section.

But if you are interested in knowing the details of the sensor readings implementation, please read this section.
{{< /admonition >}}


### Thermistors
The temperature sensor I'm using uses a thermistor to read the ambient temperature. A thermistor is a resistor whose resistance is dependent on the temperature. 
By measuring the resistance we can calculate the temperature by approximation using one of two formulas, [beta equations](https://www.ametherm.com/thermistor/ntc-thermistor-beta) or the most accurate [Steinhart and Hart Equation](https://www.ametherm.com/thermistor/ntc-thermistors-steinhart-and-hart-equation)

Despite being more accurate, the Steinhart and Hart Equation is a bit more complex to implement. Since the objective of this tutorial isn't having the most accurate temperature measuring I'll implement the Beta equation.
We can find the Beta equation in the page 6 of the thermistor [datasheet](https://files.seeedstudio.com/wiki/Grove-Temperature_Sensor_V1.2/res/NCP18WF104F03RC.pdf). All of the constants required to complete the formula are specified on the datasheet.


{{< image src="images/beta_equation.jpg" alt="thermistor beta" height="auto"  >}}


| Variable |  |
| ------ | ----------- |
| T | Calculated temperature in **Kelvin** - The objective|
| T0 | Reference temperature - 25 Celsius in Kelvin - 298.15K - Defined in the datasheet|
| R<sub>T</sub>   | Measured thermistor resistance (ohm), calculate using the voltage divider circuit. **Check next section** |
| R<sub>T0</sub>   | Resistance at 25C - (100K ohm) - Datasheet page 11 |
| β | 4250 - Datasheet page 11|

On the next section we will see how we can calculate the RT value.

### Measure the thermistor resistance

From the previous equation only the Rt value is missing, this Rt corresponds to the measured thermistor resistance. Invoking the analogRead() function will read the voltage on the pin A0 and convert it to a digital value with a 10 bit resolution, it isn't exactly what we need. Remember we need the value of the resistance (ohm) and not the voltage(V) measured on the pin A0.

By inspecting the [temperature sensor internal schema](https://files.seeedstudio.com/wiki/Grove-Temperature_Sensor_V1.2/res/Grove_-_Temperature_sensor_v1.1.pdf) we can see that we basically have a [voltage divider circuit](https://ohmslawcalculator.com/voltage-divider-calculator). With the value of a known resistance (100k Ohm, R1 in the schema) and the voltage measured in the A0 pin (SIG in the schema) we can calculate the unknown resistance. In this case it is the resistance of the thermistor, the last piece missing for the temperature calculation formula. 

{{< image src="images/temp_sensor_schema.jpg" alt="voltage divider" height="auto"  caption="Temperature sensor internal schema">}}

After some math we will end up with the following formula to calculate the value of the resistance of the thermistor:

{{< image src="images/thermistor_resistance.jpg" alt="thermistor resistance voltage divider equation" height="auto"  caption="Voltage divider equations" >}}
| Variable |  |
| ------ | ----------- |
| Rthermistor == R2 | Resistance(ohm) of the thermistor, R2 in the schema|
| Vcc   | Arduino Vcc, 3.3V for the nano, 5V for the uno|
| R1   | 100K ohm - R1 in the schema, the known resistor of the voltage divider circuit |
| V | Voltage measured on the A0 pin - In the schema it is the voltage measured between SIG and GND. Calculated using the value returned by the analogRead function (formula 2)|
| analogReadValue | Value returned by the analogRead function. A Integer between 0 and 1023 representing the voltage read on pin A0 after the analog to digital conversion. |


With the formula to obtain the resistance of the thermistor we can implement the beta equation in the Arduino to get temperature read by the sensor.


## Azure Configuration

Before proceeding, we have to create the necessary resources on Azure. For this project we will need the following resources:
* Resource group
* Iot Hub
* Device

To create these resources we can either use azure cli or the web interface, in this tutorial I'll mainly use the web version but I'll document the Az cli commands whenever possible.

### Create a resource group
Navigate to Azure resources group page and create a new resource group.

{{< image src="images/step1-create-resource-group.jpg" alt="create azure resource group" height="auto"  caption="Create azure resource group">}}  

    az group create  --location eastus --resource-group Arduino-iot-demo


### Create IoT Hub
Navigate to Azure IoT Hub page and create a new Hub.

{{< image src="images/step2-create-iot-hub.jpg" alt="create azure iot hub" height="auto"  caption="Create azure IoT Hub">}}
{{< admonition type=tip title="This is a tip" >}}
To avoid costs you can choose the Free tier during the hub creation.
{{< /admonition >}}
{{< image src="images/step2-2-create-iot-hub-free-tier.jpg" alt="create azure iot hub - Free tier" height="auto"  caption="Create azure IoT Hub - Free tier">}}

```az iot hub create --name iot-hub-name --resource-group Arduino-iot-demo --sku F1 --partition-count 2```

### Create Device
After the Hub is created we must add a new device to it. To do that you should navigate to the newly created hub page, and click on the "Devices" tab.
{{< image src="images/step3-1-create-device.jpg" alt="create iot device" height="auto"  caption="Create IoT Device 1">}}
{{< image src="images/step3-2-create-device-done.jpg" alt="create iot device" height="auto"  caption="Create IoT Device 2">}}

#### Get keys to generate the SAS token
To authenticate on Azure Iot the device will need to generate a SAS token, we will see the details later in the code, but for now you should copy and save the device primary key.
{{< image src="images/step4-1-get-sas-string.jpg" alt="Open the device details" height="auto"  caption="Open the device details">}}
{{< image src="images/step4-2-get-sas-string.jpg" alt="Get Device primary key" height="auto"  caption="Get Device primary key">}}


## Configure Arduino root trusted certificates

{{< admonition type=warning title="Important" >}}
Arduino nano rp2040 stores the trusted root certificates on the wifi module hardware. You must install the Azure IoT Hub Root certificate in the wifi module.

If you fail to do so you may get the following error: MQTT connecting ... [ERROR] failed, status code =-2. Trying again in 5 seconds.
{{< /admonition >}}
{{< admonition type=warning title="Important" >}}
These instructions only apply to the Arduinos equipped with NINA wifi modules, for example the nano rp2040. If you want to run this code on another hardware please verify how the root certificates are configured before continuing.
{{< /admonition >}}
{{< admonition type=tip title="IoT Hub root certificate migration after 2023" >}}
Microsoft will change the Root certificate of all IoT Hubs domains in 2023.
Currently the root certificate is "Baltimore CyberTrust Root", after 2023 it will be "DigiCert Global Root G2". This means that you'll have to repeat this step after the migration, otherwise your code will stop working.

However, if you want to keep your project running without issues after the migration you can already load the new root certificate. You can fetch it from this domain g2cert.azure-devices.net.

More info [here](https://techcommunity.microsoft.com/t5/internet-of-things-blog/azure-iot-tls-critical-changes-are-almost-here-and-why-you/ba-p/2393169).
{{< /admonition >}}


Contrary to a typical browser, the Arduino wifi module doesn't have capacity to store hundred of Root certificates, you can only store around 10 certificates.
So you'll have to pick them wisely according with your project needs. In this case we only want to connect to azure IoT Hub, so it is enough to add the root certificate of Iot Hub certificate chain.

To install/delete root certificates you have to use the WiFiNina firmware update [tool](https://docs.Arduino.cc/tutorials/generic/WiFiNINAFirmwareUpdater). It is bundled by default in Arduino IDE.

1. Open the firmware update sketch  
To run the firmware update tool you'll have to upload the FirmwareUpdate sketch to the Arduino.
{{< image src="images/root-cert-1.jpg" alt="Open the WifiNina firmware update example" height="auto"  caption="Open the firmware update sketch">}}

1. Upload the FirmwareUpdater sketch to Arduino
{{< image src="images/root-cert-2.jpg" alt="Upload the WifiNina firmware update sketch" height="auto"  caption="Upload the WifiNina firmware update sketch">}}

1. Open the firmware update tool
{{< image src="images/root-cert-3.jpg" alt="Open the firmware update tool" height="auto"  caption="Open the firmware update tool">}}

1. Find the hostname associated with your IoT Hub  
{{< image src="images/root-cert-4.jpg" alt="Find the Iot Hub url" height="auto"  caption="Find the Iot Hub url">}}

1. Install the azure Iot Hub root certificate on the wifi module  
You don't need to download the certificate to your machine. The firmware update tool fetches the certificate associated with the hostname and installs it in the wifi module.  
{{< image src="images/root-cert-5.jpg" alt="Upload the Iot Hub root certificate to the wifi module" height="auto"  caption="Upload the Iot Hub root certificate to the wifi module">}}


## Code Implementation

In this tutorial we aren't going to reinvent the wheel, so we will use some libraries already created specifically for Arduino.

|Library|Location|
| ------ | ----------- |
|Azure SDK for Embedded C - Arduino port|https://www.Arduino.cc/reference/en/libraries/azure-sdk-for-c/|
|WiFiNINA|https://www.Arduino.cc/reference/en/libraries/wifinina/|
|Arduino Client for MQTT|https://www.Arduino.cc/reference/en/libraries/pubsubclient/|

All of these libraries can easily be installed using the Arduino Ide library manager. You should be able to find all of them on the Arduino IDE library manager.
{{< image src="images/arduino-ide-library-manager.jpg" alt="Arduino IDE library manager" height="auto" caption="Arduino IDE library manager">}}

{{< admonition type=warning title="Code compatibility" >}}
This code is **not** generic enough to be used without modifications on other Arduino boards. The first limitation is the Wifi module, I'm explicitly using the WifiNina library, if you have other hardware for the network interface you must adapt your code to use the corresponding library. You can see some code samples for different hardware [here](https://github.com/Azure/azure-sdk-for-c-Arduino/tree/main/examples).

I also have some usages of mbed core modules. I am not sure if all boards have the mbed core included or not, I believe it only applies to the nano 33 boards and the rp2040?
{{< /admonition >}}

The code to read the temperature from the sensor and upload it to azure is available in the [Github repo](https://github.com/hugo-ma-alves/Arduino-nanorp2040-azure-iot-hub-demo).
Before uploading it to Arduino you must open the secrets.h file and change the variables accordingly. This file defines the values that are specific to your environment, for example your wifi ssid and password and the azure credentials.

```C
// Wifi
#define WIFI_SSID "{REPLACE_WITH_WIFI_SSID}" //WIFI SSID
#define WIFI_PASSWORD "{REPLACE_WITH_WIFI_PASSWORD}" //WIFI PASSWORD

// Azure IoT
#define IOT_CONFIG_IOTHUB_FQDN "{REPLACE_WITH_AZURE_IOT_URL}" //Azure IoT Hub url
#define IOT_CONFIG_DEVICE_ID "Arduino" //Device Id
#define IOT_CONFIG_DEVICE_KEY "{REPLACE_WITH_THE_DEVICE_KEY}" //Device Primary key
```

After configuring all the secrets you can compile and deploy the application to Arduino. The easiest way is to use Arduino IDE:
{{< image src="images/compile.jpg" alt="Compile and deploy to Arduino" height="auto" caption="Compile and deploy to Arduino">}}

### Check that the telemetry is being received by IoT Hub

We can check if everything is working correctly by looking to the Serial output of Arduino. If everything is ok, the message "Metric sent to Iot hub" should appear every 5 seconds.
Apart from the serial monitor you can also know if the everything is working by looking to the Arduino RGB led, it should blink with the green colour every 5 seconds (every time a metric is sent to azure). If the red led starts to blink fast it means that something failed, and in this case you should open the serial monitor to see what is going on. 

On Azure side we can also monitor the telemetry that is being received. By running the following commands in powershell we should start seeing the telemetry data appearing in azure.

```powershell
$CONNECTION_STRING=$(az iot hub connection-string  show --hub-name ${Replace by your hub name} --output tsv) 

az iot hub monitor-events -n "${Replace by your hub name}" --login $CONNECTION_STRING --output table
```  

{{< image src="images/az_events_output.jpg" alt="Azure IOT monitor" height="auto" caption="Azure IoT events monitor">}}


## Next steps

At this point we have the raw data available in azure IoT hub. Now we just need to parse it and start creating useful dashboards with it. This is the objective if the part 2 of this tutorial, we will use azure stream analytics jobs and Azure data explorer to parse and show the data collected by the Arduino sensors.

## Common errors

|Error|Solution|
| ------ | ----------- |
|[ERROR] failed, status code =-2. Trying again in 5 seconds.|Check if the azure IoT Hub root certificate is [trusted](#configure-ssl-root-trusted-certificates).|


# Extras

Code to generate a SAS token in Python. It may be useful if you want to troubleshoot its generation on Arduino.
You can run this code locally and the compare the token with the one generated by Arduino.

```python
from base64 import b64encode, b64decode
from hashlib import sha256
from time import time
from urllib import parse
from hmac import HMAC

#DEVICE_KEY_IN_BASE64-copy from azure portal;
DEVICE_PRIMARY_KEY = "---"
IOT_HUB_NAME = "Arduino-demo"
DEVICE_NAME = "Arduino"

ttl = time() + 3600

sign_key = "%s\n%d" % ((parse.quote_plus(IOT_HUB_NAME + ".azure-devices.net/devices/nano")), int(ttl))
print("Signature plain text: ")
print("---")
print(sign_key)
print("---")

signature = b64encode(HMAC(b64decode(DEVICE_PRIMARY_KEY), sign_key.encode('utf-8'), sha256).digest())

rawtoken = {
    'sr' :  IOT_HUB_NAME + ".azure-devices.net/devices/" + DEVICE_NAME,
    'sig': signature,
    'se' : str(int(ttl))
}

print("SAS token:")
print('SharedAccessSignature ' + parse.urlencode(rawtoken))

```

## References

Hardware:
- [Arduino Nano RP2040 datasheet][9]
- https://www.ametherm.com/thermistor/ntc-thermistors-steinhart-and-hart-equation  
- https://www.ametherm.com/thermistor/ntc-thermistor-beta  
- https://eepower.com/resistor-guide/resistor-types/ntc-thermistor/  
- https://files.seeedstudio.com/wiki/Grove-Temperature_Sensor_V1.2/res/NCP18WF104F03RC.pdf  
- https://wiki.seeedstudio.com/Grove-Temperature_Sensor_V1.2/  
- https://ohmslawcalculator.com/voltage-divider-calculator  
- https://www.Arduino.cc/en/Tutorial/BuiltInExamples/ReadAnalogVoltage  

Azure:
- [Azure SDK documentation][1]
- [Azure SDK samples][2]
- [Azure IoT Hub SAS][6]
- [Azure MQTT protocol][7]
- [Stream analytics and power bi][10]

Code related:
- [mbed hmac api][3]
- [mbed base64 api][4]
- [mbed HMAC tutorial][5]
- [WIFINina firmware updater][8]
 

---
[1]: https://azuresdkdocs.blob.core.windows.net/$web/c/az_iot/1.0.0/index.html
[2]: https://github.com/Azure/azure-sdk-for-c-Arduino/tree/main/examples
[3]: https://siliconlabs.github.io/Gecko_SDK_Doc/mbedtls/html/md_8h.html
[4]: https://siliconlabs.github.io/Gecko_SDK_Doc/mbedtls/html/base64_8h.html
[5]: https://techtutorialsx.com/2018/01/25/esp32-Arduino-applying-the-hmac-sha-256-mechanism/
[6]: https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-dev-guide-sas
[7]: https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-mqtt-support
[8]: https://docs.Arduino.cc/tutorials/generic/WiFiNINAFirmwareUpdater
[9]: https://content.Arduino.cc/assets/ABX00053-datasheet.pdf
[10]: https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-live-data-visualization-in-power-bi