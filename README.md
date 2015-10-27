# Micrium/Renesas AWS IoT Getting Started

## Kit Contents

The Renesas RX63N AWS IoT Starter Kit has the following contents:
* Renesas YRDKRX63N board
* USB to mini-USB Cable
* Preloaded with Micrium's Smart Home Gateway Demo

![YRDKRX63N-AWS](./img/yrdkrx63n_aws.jpg)


## Smart Home Gateway Overview
The Smart Home Gateway Demo simulates a whole-home application connected to AWS IoT, utilizing an RX63N as the gateway device that handles the interaction between sensors in a home. The idea is an appliance or temperature sensor may have Personal Area Network connection (Bluetooth, ZigBee, Wireless USB, etc.) instead of a Local Area Network connection that provides a connection to the internet. This gateway would be the connection point all of the devices on the PAN to connect to the internet. In the Smart Home Gateway the PAN is simulated by the buttons and potentiometer on the YRDKRX63N. 

The connection between the YRDKRX63N and AWS IoT is done via a protocol call MQTT. More information on MQTT can be found on the AWS IoT website, as well as [here](http://www.mqtt.org).

The Smart Home Gateway has a few different components to it. Currently three features are implemented:
* Appliances
* Temperature sensors
* Alarms

### Hardware Interaction
The image below shows you the options you have for interacting with the Smart Home Gateway simulation:
![YRDKRX63N Interactions](./img/yrdkrx63n_interactions.png)

Anytime you change an appliance state or temperature value on the RX63N that change is immediately sent to AWS IoT. 

### Appliances
The Smart Home Gateway simulation has three appliances: Dishwasher, Lamp and a Dryer. As shown in the image in the Hardware Interactions section below, you can scroll through the appliances and change their state using Switch 1 and Switch 2 on the YRDKRX63N. Anytime an appliance’s state is changed a MQTT message is immediately published to AWS IoT. 

### Temperature Sensors
The Smart Home Gateway simulation also has three temperature sensors: Kitchen, Family Room and Garage. You can change the temperature sensor using Switch 3, and you can use the potentiometer to change the actual temperature value. Similar to the appliances, any time a temperature value is changed it is immediately published to AWS IoT.

### Alarms
The alarms in the Smart Home Gateway are directly related to the temperature sensors. You can set a low and high trigger value for each of the temperature sensors, so if a temperature drops below the low trigger or goes above the high trigger the alarm is set to active. The user has the ability to set a silence flag on an alarm so the alarm no longer shows as active. Once the temperature value returns to a non-alarm value, the active and silence flags are automatically cleared. 


## Prerequisites

In order to run the Smart Home Gateway Demo on your own AWS account you need the following:
* AWS account. Click [here](https://aws.amazon.com) to create an account.
* IAR Embedded Workbench for RX<sup>1</sup>. A free 30-day trial can be obtained [here](https://www.iar.com/iar-embedded-workbench/renesas/rx).
* OpenSSL installed on your machine. Downloads available for [Win](https://www.openssl.org/community/binaries.html), [OS X](http://apple.stackexchange.com/questions/126830/how-to-upgrade-openssl-in-os-x) and Linux.
* Bin2Header python script. Download [here](http://sourceforge.net/projects/bin2header/).
* Mosquitto client found [here](http://mosquitto.org/download/). 
* Contact Micrium [here](http://www.micrium.com/aws-iot-starter-kit) to obtain the Smart Home Gateway software.

<sup>1</sup>IAR is only available on Windows. 


## Importing and compiling the Smart Home Gateway project

1. Import the Smart Home Gateway project into IAR
    * Navigate to File -> Open -> Workspace

    * From the top of the RX63N project folder, open the .eww file located at:
    ![Micrium -> Example -> Renesas -> YRDKRX63N -> OS3-MQTT-SSL -> IAR - NoSource -> OS3-MQTT-SSL.eww](./img/iar_path.png)

2. Right click on the project name `OS3-MQTT-SSL - FLASH` then click `Rebuild All`:
![Found on the left side of IAR](./img/iar_rebuild.png)

3. After the project finishes building, plug in your ethernet cable to the YRDKRX63N, and then plug in the USB to the J-Link USB as shown below.
![Ethernet and USB connections](./img/yrdkrx63n_connections.png)

4. Click the Download and Debug button ![Download and Debug button](./img/iar_download_debug_btn.png). If prompted to upgrade the firmware or setup the hardware, click ok.

5. Once the screen has changed to the debugging view and it pauses with `main()` highlighted, click to Go button in the top left corner to run the program:
![IAR Go Button](./img/iar_go.png)

## Downloading and converting X.509 certificates from AWS IoT

AWS IoT requires every device that connects to provide a signed X.509 certificate in order to connect. For the Smart Home Gateway to connect we use [Micrium's TCP/IP stack](http://micrium.com/rtos/uctcpip/overview/) and [Mocana's NanoSSL stack](https://www.mocana.com/iot-security/nanossl) to connect directly to AWS IoT.


1. Generate a certificate in AWS IoT
    * Navigate to [AWS IoT](https://aws.amazon.com/iot). Log in using your AWS account.
        
    * Click on the `+ Create a resource` and then on `Create a certificate`:
    ![Create a certificate](./img/aws_create_cert.png)
        
    * If you have a CSR you'd like signed, now is the time you can upload one otherwise click `1-Click Certificate Create`. This will provide you with a certificate, public key and private key. You should download all three of them.
        
2. Download the root CA certificate file

    The secure connection between AWS IoT and the RX63N requires Amazon to send us a certificate in addition to us sending Amazon a certificate. In order for us to validate the certificate Amazon sends us we need to have a root certificate to validate the certificate against. The root certificate in this case is signed by Symantec and can be downloaded from [here](https://www.symantec.com/content/en/us/enterprise/verisign/roots/VeriSign-Class%203-Public-Primary-Certification-Authority-G5.pem). You should save this file to the same directory as the keys and certificate from Amazon as it will also have to be converted to the correct format. 

3. Activate the certificate and add a policy

    More detailed information on this can be found in AWS IoT's documentation [here](http://docs.aws.amazon.com/iot/latest/developerguide/what-is-aws-iot.html). 
    * Click on the checkbox under your certificate, then under the actions dropdown click activate.

    * Go back up to the `+ Create a resource` at the top and click on `Create a policy`. For simplification of this getting started guide we're creating a policy that allows the certificate full access to all of AWS IoT's features. Fill out the policy form with the following parameters:
        * Name: PubSubToAnyTopic
        * Action: iot.\*
        * Resource: \*
![Create a policy](./img/aws_create_policy.png)

    * Go back and click the checkbox on the certificate you created. On the Actions dropdown click `Attach a policy`. Enter the name of your policy (PubSubToAnyTopic). 

4. Convert the certificate to a header file

    In order for Mocana's NanoSSL library to be able to use the certificates from Amazon and Symantec we need to convert them from ASCII (PEM) to binary (DER). Once we have our binary certificates we will create three array's that will be loaded into our Smart Home Gateway project. 
    * In a terminal window, navigate to the directory that has the certificates and keys. 

    * First convert the two PEM certificates (AWS IoT cert and root CA) to DER using OpenSSL:
    ```
    openssl x509 -outform der -in rootCA.pem -out rootCA.der
    openssl x509 -outform der -in AWSIoT.pem -out AWSIoT.der
    ```

    * Next convert the private key to DER format as well using OpenSSL:
    ```
    openssl rsa -outform der -in privkey.pem -out privkey.der
    ```

    * Still on the terminal window, use the bin2header python script to create a header file from each DER file.
    ```
    bin2header rootCA.der
    bin2header cert.der
    bin2header privkey.der
    ```

    * Copy the array values from each of the header files it generates into the appropriate array in `cert.h`. You can find `cert.h` in IAR under the APP folder:
    ![IAR cert.h](./img/iar_cert.png)

5. Recompile and flash the RX63N
    * Just as before, right click on `OS3-MQTT-SSL` and click `Rebuild All`.
    * Once the build is finished click the Download and Debug button ![Download and Debug button](./img/iar_download_debug_btn.png). If prompted to upgrade the firmware or setup the hardware, click ok.
    * When the debug window appears click the go button.

6. Checking your connection to AWS IoT
    
    At the bottom of the LCD screen on the RX63N it may read `MQTT Not Connected` initially. Once connected to AWS IoT that message will be replaced with `Pub: 0 Sub: 0` to show you how many MQTT messages have been published and received. If that message persists longer than 15-30 seconds then there is an issue with your certificates. You'll want to go back and make sure they were converted to an array correctly and added to the IAR project correctly. 
    
    One trouble shooting step you can take is using the mosquitto_sub and mosquitto_pub tools to attempt to publish a message to AWS IoT from your computer to make sure the certificate is valid and able to publish. Those commands would be:
    ```
    mosquitto_sub --cafile rootCA.pem --cert cert.pem --key privkey.pem -h data.iot.us-east-1.amazonaws.com -p 8883 -d -q 1 -t test/topic 
    mosquitto_pub --cafile rootCA.pem --cert cert.pem --key privkey.pem -h data.iot.us-east-1.amazonaws.com -p 8883 -d -q 1 -t test/topic -m "Hello World!"
    ```
    After executing these commands you should see `Hello World` published to the mosquitto_sub client.
    
    If your issues persist feel free to reach out to us at Micrium. Our contact info can be found [here](http://micrium.com/about/contact/).


## Smart Home Gateway topic organization

The Smart Home Gateway uses `com.ucos` as the top level for all MQTT messages. The last level of all MQTT topics in the Smart Home Gateway is the MAC address of the board. Using the MAC address gives us a unique way to identify each board. All data sent from the RX63N to AWS IoT uses a three level topic: `com.ucos/specific-topic/001122334455` where `001122334455` would be the board's MAC address. 

On a few of the topics, you can command the Smart Home Gateway to do things like turn off or on an appliance, or set an alarm value. To command the RX63N you add a command topic level such as: `com.ucos/specific-topic/cmd/001122334455`. More details on which topics can be commanded are shown below.

**SHG Topics:**

| Topic Name | Topic Value | Accepts Commands | Example |
| --- | --- | --- | --- |
| Appliance | appliance | Yes | com.ucos/appliance/001122334455 |
| Temperature | temperature | No | com.ucos/temperature/001122334455 |
| Alarm | alarm | Yes | com.ucos/alarm/001122334455 |


**Appliances:**

| Appliance Name | Topic Value |
| --- | --- |
| Dishwasher | dishwasher |
| Lamp | lamp |
| Dryer | dryer |


**Appliance Parameters:**

| Parameter | Values | Cmd | Description |
| --- | --- | --- |
| state | 0-1 | Yes | 0 - Off, 1 - On |
| milliamps | 0 - 1000 | No |Number of milliamps used since last transmit |


**Appliance Examples**

```
Topic: com.ucos/appliance/001122334455
Payload: {"dishwasher" : {"state" : "0", "milliamps" : "5"}}

Topic: com.ucos/appliance/cmd/001122334455
Payload: {"dishwasher" : {"state" : "1"}}
```


**Temperature Sensors:**

| Temperature Sensors | Topic Value |
| --- | --- |
| Family Room | family\_rm |
| Kitchen | kitchen |
| Garage | garage |


**Temperature Parameters:**

| Parameter | Values | Cmd | Description |
| --- | --- | --- | --- |
| F | 60-90 | No | Temp ranges from 60 - 90 degrees F |
| humidity | 0 - 100 | No | Humidity in the room |


**Alarm Parameters:**

| Parameter | Values | Cmd | Description |
| --- | --- | --- | --- |
| low | 60-75 | Yes | Low alarm trigger |
| high | 75-90 | Yes | High alarm trigger |
| active | 0-1 | No | Active alarm state. 0 - Off, 1 - On |
| silent | 0-1 | Yes | Silenced alarm state 0 - Off, 1 - On |


**Temperature and Alarm Examples:**
```
Topic: com.ucos/temperature/001122334455
Payload: {"kitchen" : {"F" : "75", "humidity" : "35"}}

Topic: com.ucos/alarm/001122334455
Payload: {"kitchen" : {"low" : "63", "high" : "88", "active" : "0", "silent" : "0"}}

Topic: com.ucos/alarm/cmd/001122334455
Payload: {"kitchen" : {"low" : "65", "high" : "85"}}
```


## Interacting with the RX63N via Mosquitto

### Subscribing to SHG data via Mosquitto

We can use the Mosquitto clients to view the data the RX63N is sending to AWS IoT. In a terminal window you'll want to execute the following command:
```
mosquitto_sub --cafile rootCA.pem --cert cert.pem --key privkey.pem -h data.iot.us-east-1.amazonaws.com -p 8883 -d -q 1 -t com.ucos/#
```

This command will receive all messages sent to the `com.ucos` topic. Messages are sent every 15 seconds unless you interact with the hardware via buttons or potentiometer.

### Publishing data to the SHG via Mosquitto

We can also use the Mosquitto client to send data. In a terminal window you'll want to execute the following command (be sure to replace   `001122334455` with your board's MAC address):
```
mosquitto_pub --cafile rootCA.pem --cert cert.pem --key privkey.pem -h data.iot.us-east-1.amazonaws.com -p 8883 -d -q 1 -t MQTT-TOPIC -m JSON-PAYLOAD

Turn on the dishwasher:
MQTT-TOPIC: com.ucos/appliance/cmd/001122334455
JSON-PAYLOAD: {"dishwasher" : {"state" : "1"}} 

Set the garage low alarm trigger:
MQTT-TOPIC: com.ucos/alarm/cmd/001122334455
JSON-PAYLOAD: {"garage" : {"low" : "65"}} 
```

