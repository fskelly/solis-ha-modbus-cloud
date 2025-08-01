## Purpose
A combination of Solis Cloud, Home Assistant via RS485 (Modbus) communication. This repo is a documented workaround for Solis inverters to connect Solis Cloud and the local Home Assistant based on my own experience. It includes references, examples of [the code in Home Assistant](https://github.com/alienatedsec/solis-ha-modbus-cloud/wiki/Sample-Automations), more about [configuration](https://github.com/alienatedsec/solis-ha-modbus-cloud#configuration), as well as [wiring](https://github.com/alienatedsec/solis-ha-modbus-cloud#wiring) and all [required components](https://github.com/alienatedsec/solis-ha-modbus-cloud#devices-and-cost).

## Limitations and some history
My PV installation commenced back in August 2022. Since then, I started my journey with Home Assistant, and several Solis integrations made by some clever folks out there. My main goal was to keep Solis Cloud functionality and allow for advanced monitoring, remote control, and general automation of my home equipment. As I was going through my journey, I found many obstacles and limitations with Solis Cloud, dataloggers, and its design. Therefore, to get over those obstacles, I have designed the below, which meets my requirements and it works quite well.

Most importantly, my design doesn't disconnect the Solis cloud, and works (coexists) with your local Home Assistant integration in combination with [any Solis datalogger](https://github.com/alienatedsec/solis-ha-modbus-cloud#supported-solis-dataloggers). 

You can watch a great video from `Gordon Markus` summarising his 12-months experience and Solis inverters limitations - https://youtu.be/F7r12UjzZyU

## Diagram
![Diagram](/images/solis-ha-modbus-cloud-diagram.png)

## Description
The main goal is to ensure that a datalogger can work as normal and without any distruption. The datalogger acts as the Master device, and it talks directly to the inverter (Slave). Having an additional Master (e.g., Waveshare Gateway) on the same RS485 network is to create collisions, which could affect all RS485 devices. Some dataloggers don't accept multiple TCP connections (e.g., `S2-WL-ST`), and those will disconnect the Solis Cloud whenever used with an integration; others don't support Modbus TCP (e.g., `DLS-W`, `S3-WIFI-ST`), and some will fully rely on the cloud (`S3-WIFI-ST`). Some obsolete dataloggers couldn't provide `remote control` functionality via Solis Cloud (e.g., `DLS-W`, `DLS-L`).

**However, there were reports that `remote control` functionality started working with obsolete dataloggers after Solis' maintenance on 13 April 2023. Furthermore, as of April 2025, these dataloggers are being phased out, and may no longer work due to dependencies on third-party integrations (SolarMan). Solis recommends purchasing a newer model of their datalogger for improved compatibility and ongoing support.**

To intgrate with the Home Assistant, we introduce the Waveshare gateway to query the inverter. This project uses two Waveshare devices that can serve multiple clients, keep the inverter's cloud reporting functionality,  and is not limited by any supported datalogger. The below is my configuration, but the use case could differ per datalogger.

The main device is the Waveshare RS485 to PoE ETH (B) - the non-PoE version is described as per [this guideline](https://github.com/wills106/homeassistant-solax-modbus/wiki/Installation-Notes#option-1-waveshare-rs485-to-eth-b-din-rail-mounted-model). There are no major differences between PoE and non-PoE devices, except for the convenience of powering those when you have multiple next in line.

I was trying different settings as I had some teething issues, and I noticed that it works better when the integration (I am using [Home Assistant SolaX Modbus integration](https://github.com/wills106/homeassistant-solax-modbus) for my Solis inverter) is polling data more often (e.g., every 5 seconds). As much as no longer relevant, you can find more about those historical issues [at the following address.](https://github.com/wills106/homeassistant-solax-modbus/issues/340)

## Solution Concepts
**DLS-W - Solis Sensor, Home Assistant Solax Modbus, SolisMon3, Grafana**

![Diagram](/images/solis-custom-diagram.png)

**S2-WL-ST - Solis Sensor, Home Assistant Solax Modbus**

![Diagram](/images/solis-custom-diagram-s2.png)

## Wiring
I recommend using one of the pairs from the Ethernet cable for RS485 wiring. e.g., A twisted pair of `blue` for RS485(A) and `white-blue` for RS485(B). There is no need for 120 Ohms resistors at each end of the RS485. Both, Waveshare and the Inverter have resistors built-in.

![Wiring](/images/solis-ha-modbus-cloud-wiring-diagram.png)

## Configuration

**Please use VirCom (not the Web Interface) to configure your Waveshare devices**

- Follow the prerequisite (Optional) - [Waveshare firmware update to `v1.486`](https://github.com/alienatedsec/solis-ha-modbus-cloud/discussions/17) - further discussed in this thread https://github.com/alienatedsec/solis-ha-modbus-cloud/discussions/14
- The `TCP Server` on my [diagram](https://github.com/alienatedsec/solis-ha-modbus-cloud#diagram) and [presented here as 187 GW](https://github.com/alienatedsec/solis-ha-modbus-cloud#final-result) acts as a gateway for other network devices (e.g., the datalogger wired to `TCP Client` and for any compatible HA integration).
- The Serial baud rate on the `TCP Server` is 9600 - default on the inverter; however, you can amend it independently to datalogger's settings - up to 38400 on the inverter, and match the same within `TCP Server` settings - increasing the baud rate is highly recommended.
- The `TCP Server` - `Modbus gateway type` is set to `Multi-host non-storage type` - RS485 Multi-Host and Bus Conflict detection **enabled**.
- The `TCP Client` - `Modbus gateway type` is set to `Simple Modbus TCP to RTU` - RS485 Multi-Host and Bus Conflict detection **disabled**.
- You should use the IP address and the port number of your `TCP Server` when configuring any `Modbus TCP` compatible integration - e.g., like in the [Home Assistant Solax Modbus](https://github.com/wills106/homeassistant-solax-modbus#installation) installation notes.
- Therefore, the `TCP Client` [presented here as 171 DLG](https://github.com/alienatedsec/solis-ha-modbus-cloud#final-result) has its own IP address (irrevelevant), but within its `Dest IP/Domain` and `Dest. Port` config fields, it has to use the IP and the port of the `TCP Server`, and it needs the same Serial config as on a datalogger - baud rate 9600 - default for `S2-WL-ST` and `S3-WIFI-ST` dataloggers, where `DLS-W` baud rate can be amended via its hidden config web page `http://IP/config_hide.html`. The same applies to `DLS-L` under `http://IP/hide_set.html`, but it is password protected with username: `admin`, password: `8 characters linked with a datalogger`
- Both `TCP Server` and `TCP Client` - `Transfer Protocol` is set to `Modbus_TCP Protocol` - Waveshare is converting `Modbus RTU` messages to `Modbus TCP` and vice-versa - screenshots below.

**TCP Server Config**

![server](/images/solis-ha-modbus-cloud-server.png)

**More Advanced Settings for TCP Server**

![server-advanced](/images/solis-ha-modbus-cloud-server-adv.png)

**TCP Client Config**

![client](/images/solis-ha-modbus-cloud-client.png)

**More Advanced Settings for TCP Client**

![client-advanced](/images/solis-ha-modbus-cloud-client-adv.png)

- The Waveshare used for the batteries connection is the same device, but the `Transfer Protocol` is set to `None`. I use VirCom to extend its connection to COMX on my laptop, so the software can connect to batteries and monitor them. Looking into using `Modbus_TCP Protocol` and integrating with HA via MQTT or even setting the MQTT on the Waveshare, but that is not my priority at the moment.

## HA Integrations
Datalogger/Waveshare - Modbus TCP - direct connection - e.g., `DLS-L`, `S2-WL-ST`, `Waveshare RS485 to PoE ETH (B)`
- [Home Assistant Solax Modbus](https://github.com/wills106/homeassistant-solax-modbus) - stable since [2023.03.2b18](https://github.com/wills106/homeassistant-solax-modbus/releases/tag/2023.03.2b18)
- [HA Solis Modbus](https://github.com/fboundy/ha_solis_modbus)

Datalogger - Solarman based - direct connection - e.g., `DLS-W`
- [Home Assistant Solarman](https://github.com/StephanJoubert/home_assistant_solarman)

Cloud connection - [any Solis datalogger](https://github.com/alienatedsec/solis-ha-modbus-cloud#supported-solis-dataloggers)
- [Solis Sensor](https://github.com/hultenvp/solis-sensor)
- [Solis Cloud Control](https://github.com/mkuthan/solis-cloud-control)

## MQTT
- [SolisMon3](https://github.com/NosIreland/solismon3) - advertising Solis' Inverter data to MQTT - compatible with `DLS-W`

## Supported Solis Dataloggers
- DLS-W
- DLS-L
- S2-WL-ST
- S3-WIFI-ST

![Table](/images/datalogger-table.png)

## Untested Solis Dataloggers
- S1-W4G-ST (4-pin)

## Incompatible and untested Solis Dataloggers
The below will be incompatible with the 4-pin `Exceedconn EC04681-2023-BF Male/Female` connector, but could still work with the above [HA Integrations](https://github.com/alienatedsec/solis-ha-modbus-cloud#ha-integrations)
- S1-W4G-ST (USB)
- S4-WIFI-ST (USB)

## Devices and cost
- 2 * £35.00 - Waveshare RS485 to PoE ETH (B) - https://www.waveshare.com/wiki/RS485_TO_POE_ETH_(B)

![image](https://user-images.githubusercontent.com/73167064/224035664-2b01545d-6b8d-4646-ae75-2170aca94882.png)

- 1 * £28.00 - PoE Switch - P-Link PoE Switch 5-Port 100 Mbps (TL-SF1005P)

![image](https://user-images.githubusercontent.com/73167064/224323924-eb8add51-3f87-4143-954b-692bfe46346a.png)

- 1 * £12.00 - Exceedconn EC04681-2023-BF Male/Female for Solis/Ginlong Inverter RS-485 port

![image](https://user-images.githubusercontent.com/73167064/224319164-74400817-a5dd-426f-a5b4-1f6fbb563006.png)

- 1 * £2.50 - DIN rail

![image](https://user-images.githubusercontent.com/73167064/224319811-1bc2c361-811b-4d4f-8518-c0ee8b928317.png)

- 1 * £20.00 - Sourcingmap 240mm x 160mm x 90mm Plastic Dustproof IP65 DIY Junction Box Power Protection Case - I recommend a bigger box = more expensive

![image](https://user-images.githubusercontent.com/73167064/224323031-75df733b-3bf0-4851-bb71-75eeabc4e6af.png)

- 1 * £6.00 - Pack M20 20mm IP68 Waterproof Red Cable Glands, Suitable for 6mm - 12mm Cables, Plastic Nylon Compression Glands Connectors with Locknut and Washer - AVARTEK

![image](https://user-images.githubusercontent.com/73167064/224322440-14c7f351-8b1b-48f5-9538-c028f3aafc99.png)

- around £10.00 for some RJ45 cabling and connectors

#### Total Cost
**£148.50** (pricing out of date)
  or 
**£93.00** if you follow [this discussion #24](https://github.com/alienatedsec/solis-ha-modbus-cloud/discussions/24)

Optional to connect batteries:
- 1 * £35.00 - Waveshare RS485 to PoE ETH (B) - https://www.waveshare.com/wiki/RS485_TO_POE_ETH_(B)
![image](https://user-images.githubusercontent.com/73167064/224035664-2b01545d-6b8d-4646-ae75-2170aca94882.png)

## Final Result
![Final](/images/solis-ha-modbus-cloud-final.jpg)

## Future work
- Compatibility with other brands and inverters
- More Solution Concepts
- Battery Monitoring - HA, MQTT

## DISCLAIMER
**I AM NOT RESPONSIBLE FOR ANY USE OR DAMAGE THIS DESIGN MAY CAUSE. THIS IS INTENDED FOR EDUCATIONAL PURPOSES ONLY. USE AT YOUR OWN RISK.**

## DONATIONS
**IF YOU FOUND THIS USEFUL, YOU CAN BUY ME A BEER**

[![paypal](https://www.paypalobjects.com/en_US/GB/i/btn/btn_donateCC_LG.gif)](https://www.paypal.com/cgi-bin/webscr?cmd=_s-xclick&hosted_button_id=K3V4PSH2CV9AA)
