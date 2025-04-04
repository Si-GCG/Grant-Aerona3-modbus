# Grant-Aerona3-modbus
Control a Grant Aerona3 ASHP via modbus in Home Assistant

This project is my own fork of this project: https://github.com/aerona-chofu-ashp/modbus

TL/DR UK heat pump installers bad Home Assistant and DIY Controls good!

I have gone down the route of using an Waveshare POE modbus to ethernet adaptor to my Home Assistant Green, I have used AI to assist as I am no coder!!
I have also very heavily relied on others on both github and the Renewable Heating Hub and open energy monitor forums
I genuinely thank you all for getting me this far!

Why go down this route?

Well Grant as a company make great boilers and have done for many years, however they are not great at control systems. I was fortunate enough to qualify for the ECO4 grant in the UK and had a Grant Aerona3 heat pump installed at my house. The install was a nightmare as so many others have experienced in the UK.
However I was determined I would get the best out of my new heat pump. The installers did bare minimum with the controllers and left us with basically the controls of a regular gas/oil boiler! On/Off or reliant on the temperature of one room to control the whole house, absolutely the opposite of what would be the most efficient way of heating our house. My system had been designed to have a flow rate of 45c at -3c. 

![D86E6574-A449-4CC5-AF35-C7D348A192D9_1_102_o](https://github.com/user-attachments/assets/2c0f1ad5-14b7-4d18-9d2e-220d29509342)

Grant use rebranded Japanese Chofu heat pumps so that is a great place to start. Grants documentation is not great so I would recommend looking for the Chofu manuals to get the best information about the heat pumps.  The installers initially set the system at a fixed flow rate of 45c and told us to just move the thermostat into the room we were in and set the temp to what we found confortable. This resulted in usage of around 50kWh of energy a day which is obvioulsy ridiculous! I then went down the rabbit hole of trying to find out how to control the system better. 

The contols fitted in my example were from EPH a great company who were very good responding to my requests for information. I had the EPH Combo Pack 04 installed, even though Grant do a smart controller the installer admitted that their installers didn't understand how to fit it as they were more used to installing gas boilers!!!! The CP04 was designed for gas boilers not ASHPS, so I bought and installed the EPH GW04 Gateway (again thanks to EPH for their help getting this set up!) and installed the Ember app which gave me some smarter control of the system but it was still on/off controls based on the internal thermostat. 

![C76D4DFB-57E2-41D5-B256-B2DAE9EF967B_1_102_o](https://github.com/user-attachments/assets/f11877be-f894-447c-899e-549ab4b01eda)


Thanks to the great folks at the Renewable Heating Hub and youtube channels like Heat Geeks and Urban Plumbers I learned about weather compensation so implemented that on the manual controls I had installed. However I thought there must be a better way and here I am down the deeper rabbit hole of Home Assistant, modbus and how to control the ASHP as best I possibly can.

I emailed Grant for a technical document on the modbus registers and their response was that as they hadn't tested the modbus functionality they could not give me any information!

This led me to this project https://github.com/aerona-chofu-ashp/modbus and my own research on the subject. 

I chose the Waveshare POE variant as I had a cable run into where the pipework for the ASHP was going to come into the house so I got the (insert rude comment!) installers to run some sheilded external CAT7 cable from the "plant room" to the ASHP. You can just as easily do this project with USB or wireless controllers. 

Once installed I installed the Wavesure into a Din Rail Enclosure (you do need to modify it for the waveshare to fit (file down the top and bottom edges). Once connected then the fun began (because I am an idiot!) Layer 1 people! check and double check your cables!!!!! If it doesn't work straight away it is the cables. Both my connection to the heat pump and my ethernet cable were badly terminated,which led to this!

![4271CC76-96B0-4770-8885-34C9EBA1A256_1_102_o](https://github.com/user-attachments/assets/a50f47af-0c95-491c-a554-a90b82fcd40d)

Again for your own sanity check your cables!

Once connected you navigate to the IP address of the Waveshare (the default is: 192.168.1.200) and all you need to do is set it up with the following settings. 

I would recommend you don't go down the very long rabbit hole of following the Waveshare instructions.
change the following:
```
Device Port: 502
device web port: 80
Work Mode: TCP Server
Destination Port: 502
IP mode: static
Baud Rate: 19200
Databits: 8
Parity: none
Stopbits: 2Flow Control: none
No Data Restart: Disable
No Data Restart Time: 300
Reconnect time: 12
Protocol: Modbus TCP to RTU
Instruction Time Out 480
Enable Multi host: Yes
RS485 conflict time gap:200
```
I have attached the modbus integration settings but here is a description of the main points:
```
modbus:
  - name: aerona3 #give your heat pump an appropriate name like Pumpy McPumface!!
    type: tcp
    host: 192.168.1.200 #enter the IP address of your modbus connection
    port: 502 #keep this the same
    delay: 5

# Coil Registers I have given each a unique ID of the register address in the installer menu
switches:
      - name: "Weather Compensation"
        unique_id: "1200"
        slave: 1                         # Slave ID of the ASHP
        address: 2                       # Coil register address for Enable Flow Set Point
        write_type: coil                 # Use coil for binary values
        command_on: 1                    # 1 = Climatic Curve (Weather Compensation)
        command_off: 0                   # 0 = Fixed Set Point

sensors:
    # Input Registers
      - name: "Return water temperature"
        unique_id: aeyc-0643xu-ch-input-1-0
        slave: 1
        address: 0
        input_type: input
        unit_of_measurement: °C
        scale: 1.000000
        state_class: measurement
        scan_interval: 9
        
# Holding Registers
        # Fixed Flow Temperature
      - name: "Fixed Flow Temperature"
        unique_id: fixed_flow_temperature     
        slave: 1
        address: 2
        input_type: holding
        scale: 0.1 # Adjust scaling if needed (e.g., divide by 10 to convert to °C)
        precision: 1
        unit_of_measurement: "°C"
        device_class: temperature
        scan_interval: 11

input_number:
  fixed_out_temp:
    name: Fixed Flow Temperature
    initial: 42 # Example default value.
    min: 20
    max: 60
    step: 0.5
    unit_of_measurement: "°C""
```
I am still trying to find the best way of keeping this neat in Home Assistant and will update this as I move forward. My next biggest challenge is how I display the data in Home Assistant. 

I want to monitor power consumption and costs but am finding that the Energy cards in HA are slightly awkward at converting the data from modbus:

      - name: "Current consumption value"
        unique_id: aeyc-0643xu-ch-input-1-3
        slave: 1
        address: 3
        input_type: input
        unit_of_measurement: W
        scale: 100.000000
        state_class: measurement
        scan_interval: 10
        device_class: power

 I have also found out that the latest unofficial Octopus integration (Home Assistant Octopus Energy in HACS) allows you to track individual devices at the correct rates throughout the day. As the Grant input register for power outputs in watts this needs to be converted to kWh for the octopus integration to work.

      - platform: integration
        source: sensor.Current_consumption_value
        name: Daily_Energy_Usage
        unique_id: daily_energy_usage
        unit_prefix: k
        round: 2
 
I have added a few utility meters:

```
utility_meter:
  daily_energy:
    unique_id: daily_energy
    source: sensor.daily_energy_usage
    cycle: daily
  weekly_energy:
    unique_id: weekly_energy
    source: sensor.daily_energy_usage
    cycle: weekly
  monthly_energy:
    unique_id: monthly_energy
    source: sensor.daily_energy_usage
    cycle: monthly
``` 
I have installed the following addons through HACS https://www.hacs.xyz/docs/use/download/download/ 

```
lovelace-card-mod
apexcharts-card.js?v=2.1.2
mushroom.js
card-mod.js
energy-flow-card-plus.js
button-card.js
```
So far I have this dashboard that I am pretty happy with I have only added elements that I think are usefull.

![3210825F-9E4F-4625-B0C2-334785F26587](https://github.com/user-attachments/assets/549abc6c-1c3a-42b5-8bec-9a3f06f3db60)


Links:


https://renewableheatinghub.co.uk/forums

https://openenergymonitor.org

https://www.home-assistant.io/green/

https://thepihut.com/products/rs232-to-rj45-ethernet-module

New version
https://www.waveshare.com/rs232-485-422-to-poe-eth-b.htm

https://www.amazon.co.uk/Enclosure-Consumer-Waterproof-Terminals-Connectors/dp/B0BGSC2FF2
