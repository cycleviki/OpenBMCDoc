## Dbus-Sensor: Service for specific Hwmon device
Dbus-sensors is a collection of sensor applications that provide the xyz.openbmc_project.Sensor collection of interfaces.
They read sensor values from hwmon, d-bus, or direct driver access to provide readings. 
Some advance non-sensor features such as fan presence, pwm control, and automatic cpu detection (x86) are also supported.

 This part is service for monitor specific hwmon device.

## Service List
| Parameter  | Service                                    |   Related hwmon device                                           |
| -----------| -------------------------------------------|  ----------------------------------------------------------------|
| HwmonTemp  | xyz.openbmc_project.hwmontempsensor.service| /sys/class/hwmon/hwmon?/tempx_input                              |
| ADCSensor  | xyz.openbmc_project.adcsensor.service      | /sys/class/hwmon/hwmon?/inx_input,  name=iio_hwmon               |
| CPUSensor  | xyz.openbmc_project.cpusensor.service      | /sys/bus/peci-$bus/0-$address/peci-cpu*/hwmonx                   |
| PSUSensor  | xyz.openbmc_project.psusensor.service      | /sys/class/hwmon/hwmon?/    name belong to psuNames Support list |
| NVMESensor | xyz.openbmc_project.nvmesensor.service     | /sys/bus/i2c/devices/i2c-$bus/mux_device                         |
| FanSensor  | xyz.openbmc_project.nvmesensor.service     | /sys/class/hwmon/hwmon?/fanx_input, /sys/class/hwmon/hwmon?/pwmx |


## Parameter List

| Parameter        | Usage                                    | Temp | ADC | CPU | PSU | NVME | Fan |
| -----------------| -----------------------------------------| -----|-----| ----|-----|------|-----|
| Name(x)          | SensorName for API. **#1**               |  V   |  V  |  V  |  V  |  V   |  V  |
| Index            | For some multi-node device, e.g ADC, FAN |      |  V  |     |     |      |  V  |
| Bus              | Device Bus **#2**                        |  V   |     |  V  |  V  |  V   |  O  |
| Address          | Device slave address   **#2**            |  V   |     |  V  |  V  |      |  O  |
| PowerState       | DC/AC sensor                             |  O   |  O  |     |     |      |     |
| ScaleFactor      | Scale for sensor                         |      |  O  |     |     |      |     |
| BrdgeGpio        | Gpio pin for control ADC sensor monitor  |      |  O  |     |     |      |     |
| PresenceGpio     | Gpio pin for CPU presence                |      |     |  O  |     |      |     |
| Presence         | Gpio pin for Fan presence                |      |     |     |     |      |  O  |
| CpuID            | CPU index for part of CPU sensor  name   |      |     |  V  |     |      |     |
| CPURequired      | Use for ADC sensor dependency with CPU   |      |  O  |     |     |      |      |
| Threshold        | Setup threshold setting                  |  O   |  O  |  O  |  O  |      |     |
| DtsCritOffset    | Use for CPU DTS Sensor threshold offset  |      |     |  O  |     |      |     |
| Connector        | Setup pwm and tach mapping               |      |     |     |     |      |  O  |
| label_parameter  | Custom set for "Name","Scale","Max","Min"|      |     |     |  O  |      |     |
 
V: Required Parameters O: optional Parameters

**#1** 
Some service use "Name" as inventory API name , or use part of sensor name string in combination with hwmon label or pre-defined label name. (e.g. CPUSensor, PSUSensor...)

**#2** 
* For CPU service, "Bus" represents peci bus index, and "Address" represents the peci address (CPU0-CPU4 => 0x30-0x33). For others service, "Bus" represents i2c bus number and "Address" represents device slave address.
* For Fan service, Only need to setup Bus and Address if type is I2cFan.

## Basic structure
1. Define system service

	```
	systemBus = std::make_shared<sdbusplus::asio::connection>(io);
	systemBus->request_name("xyz.openbmc_project.ADCSensor");
	```
2. Check or establish hwmon device
3. Create Sensor (According to sensor configuration array and hwmon device), establish sensor api (create structure is different from each sensor) and inventory api.
	```
	sensorInterface = objectServer.add_interface(
		"/xyz/openbmc_project/sensors/temperature/" + name,
		"xyz.openbmc_project.Sensor.Value");

	if (thresholds::hasWarningInterface(thresholds))
	{
		thresholdInterfaceWarning = objectServer.add_interface(
            "/xyz/openbmc_project/sensors/temperature/" + name,
		"xyz.openbmc_project.Sensor.Threshold.Warning");
	}
	if (thresholds::hasCriticalInterface(thresholds))
	{
		thresholdInterfaceCritical = objectServer.add_interface(
		"/xyz/openbmc_project/sensors/temperature/" + name,
		"xyz.openbmc_project.Sensor.Threshold.Critical");
	}
	association = objectServer.add_interface(
		"/xyz/openbmc_project/sensors/temperature/" + name,
		association::interface);
	setInitialProperties(conn); 
	```

## Threshold config
```
"Thresholds": [
                {
                    "Direction": "greater than",
                    "Name": "upper critical",
                    "Label" (optional) 
                    "Severity": 1,
                    "Value": 0.901
                },
                {
                    "Direction": "greater than",
                    "Name": "upper non critical",
                    "Severity": 0,
                    "Value": 0.875
                }
            ],
```
* Direction : "less than"(LOW)  & "greather then"(HIGH)
* Name : Name can define with identifiable word
* Label : optional , some type have multiple sensor and  need label to seperate define threshold. (For example, cpu-sensor)
* Serverity : 1 = critical , 0 = warning
* Value : threshold setting value.

## HWMonTemp sensor
### Support Type
```
static constexpr std::array<const char*, 11> sensorTypes = {
    "xyz.openbmc_project.Configuration.EMC1413",
    "xyz.openbmc_project.Configuration.MAX31725",
    "xyz.openbmc_project.Configuration.MAX31730",
    "xyz.openbmc_project.Configuration.MAX6581",
    "xyz.openbmc_project.Configuration.MAX6654",
    "xyz.openbmc_project.Configuration.SBTSI",
    "xyz.openbmc_project.Configuration.TMP112",
    "xyz.openbmc_project.Configuration.TMP175",
    "xyz.openbmc_project.Configuration.TMP421",
    "xyz.openbmc_project.Configuration.TMP441",
    "xyz.openbmc_project.Configuration.TMP75"};
```
SensorTypes definition is for entity-manager configuration json file use. Above array is support list for type setting. We can use this config to make custom function in HWmonsensor service.
### Sensor configuration parameter
1. **Name** : Define the sensor name displayed in sensor API.
2. **Bus** : Device bus setting, used to compare with the detected device and identify the device matching the sensor.
3. **Address** : Device slave address setting, used to compare with the detected device and identify the device matching the sensor.
4. **PowerState** (Optional) : Set sensor as a standby sensor(PowerState="always") or a DC sensor(PowerState="On"). The default value is "always".
5. **Threshold setting** (Optional)
6. **Namex**(optional) : Only set when  the IC support multiple temperature sensors.

### Sensor Create step
1. Check hwmon device exist
	* Scan **/sys/class/hwmon**, find out hwmon device with tempx_input 
	* Check device's bus & slaveaddress. Below example is device with bus=6 & slave=0x49
	`
	lrwxrwxrwx    1 root     root             0 Aug 25 05:14 device -> ../../../6-0049
	`
	* For each hwmon device, scan the sensor configuration array  defined by entity-manager and check **Sensor type**, **Bus** , **Address** whether it match the hwmon device.	
2. Check Sensor information from configuration array 
	* For each match device, find **"Name"** from sensor configuration array to define sensor api name
	* Parsing **Threshold setting** from cofiguration array. 
	* Check the **"PowerState"** to define sensor type (standby sensor or DC sensor)
	* If hwmon IC support grather than one temperature sensor, keep check "Namex" configuration from array and parsing threshold with "label" use.
3. Create sensor with following name rule and establish monitor task for futher use.
`
"/xyz/openbmc_project/sensors/temperature/" + name
`

## ADCsensor
### Support Type
```
static constexpr std::array<const char*, 1> sensorTypes = {
    "xyz.openbmc_project.Configuration.ADC"}
```
SensorTypes definition is for entity-manager configuration json file use. Above array is support list for type setting. We can use this config to make custom function in ADCsensor service.

### Sensor configuration parameter
1. **Index** : Define ADC channel setting (0-based).
2. **Name** : Define the sensor name displayed in sensor API.
3. **ScaleFactor** (Optional) : value to to transfer to object value. Default value = 1.0.
<br>`y = ( x / 1000 ) / ScaleFactor`<br>
4. **PowerState** (Optional) : Set sensor as a standby sensor(PowerState="always") or a DC sensor(PowerState="On"). The default value is "always".
5. **Threshold setting** (Optional)
6. **CPURequired** (Optional): Set if ADC sensor is related to the cpu.
7. **BridgeGpio** (Optional) : If ADC sensor control by gpio pin, set as below example. 
	* GPIO Name must pre define in dts file
	* Polarity is the direction setting.
 
 	```
	{
            "BridgeGpio": [
                {
                    "Name": "FM_BAT_MON",
                    "Polarity": "High"
                }
            ],
		"Index": 7,
            "Name": "VOL_BATTERY",
            "ScaleFactor": 0.3333,
            "Thresholds": [
                {
                    "Direction": "greater than",
                    "Name": "upper critical",
                    "Severity": 1,
                    "Value": 3.74
                },
                {
                    "Direction": "less than",
                    "Name": "lower critical",
                    "Severity": 1,
                    "Value": 2.73
                }
            ],
            "Type": "ADC"
}
	```


### Sensor Create step
1. Check hwmon device exist
	* Scan /sys/class/hwmon, find out hwmon device with inx_input 
 	* For each hwmon device with inx_input object, check device's name is "iio_hwmon" to make sure hwmon device is for adc sensor.
	* Find **"Index"** from sensor configuration array and check if it match inx_input (index number -1 ) 
2. Check configuration array for sensor create
	* Find **"Name"** from sensor configuration array to define sensor api name
	* Parsing **Threshold setting** from cofiguration array. 
	* Find **"ScaleFactor"**. 
	* Check the **"PowerState"** to define sensor type (standby sensor or DC sensor)
	* If **"CPURequired"** set, check cpu present state. 
	* Check **BridgeGpio** set .
3. Create sensor with following name rule and establish monitor task for futher use.
`
"/xyz/openbmc_project/sensors/voltage/" + name
`

## CPUsensor
This service only support if platform support PECI channel. Because of the dependency with cpu, this service have extra process to detect cpu before create sensor with basic strucutre.
### Support Type
```
static constexpr std::array<const char*, 1> sensorTypes = {"XeonCPU"};
```
SensorTypes definition is for entity-manager configuration json file use. Above array is support list for type setting. We can use this config to make custom function in CPUsensor service.
### Sensor configuration parameter
1. **Name** : Define inventory name for CPU. . It will created cpu to inventory interface with following example 
		`/xyz/openbmc_project/inventory/system/chassis/motherboard/$Name`
2. **Bus** : Device bus setting. this bus index represents the peci bus index. e.g if we use peci-0, set bus=0.
3. **Address** : Device slave address setting this address represents CPU PECI address. 
4. **CpuID** : Defined CPU index idenfication for sensor name. Sensor name combined with `${labelname}_CPU${CpuID}` 	
	For example , Die sensor with CPUID=1
	<br>`/xyz/openbmc_project/sensors/temperature/Die_CPU1`            
5. **DtsCritOffset**(Optional) : Adjust offset for threshold value to CPU DTS sensor. Default value is 0.
6. **Threshold setting** (Optional)
7. **PresenceGpio** (Optional) : Only need it if there is a present gpio pin.

	* GPIO Name must pre define in dts file
	* Polarity is the direction setting.
 
 	```
 	 "PresenceGpio": [
                {
                    "Name": "CPU1_PRESENCE",
                    "Polarity": "Low"
                }
            ],
	```


### Sensor Create step
1. Detect CPU present state
	* Get CPU config from sensor configuration (Name, PresenceGpio)
	* Check CPU present state with Gpio pin 
	* Establish CPU inventory if cpu is present
2. Test PECI connection and export peci-cputemp and peci-dimmtemp device by user space command
	* Check Bus and Address from configuration.
 	* Test peci-x device and send raw peci command to test PECI is normal.
	* Export peci-cputemp and peci-dimmtemp device with "peci-client" driver. 
	  <br> e.g peci bus=0, address= 0x30 (CPU0, 0-base) 
        <br> `echo peci-client 0x30 /sys/bus/peci/device/peci-0/new_device`
3. Create PECI sensor hwmon device
	* Check /sys/bus/peci/devices/peci-x/ folder, find out device created with peci-client driver. 

	Following example is platform with 2 cpu and plug in one dimm for cpu 0.
	```
	root@hs9216:/sys/bus/peci/devices# ls ./*/peci*/hwmon/hwmon?/name
	./0-30/peci-cputemp.0/hwmon/hwmon5/name   
	./0-30/peci-dimmtemp.0/hwmon/hwmon6/name  
	./0-31/peci-cputemp.1/hwmon/hwmon7/name
	```
4. Check configuration array for sensor create
	* For each sensor hwmon device, use its sensor label with temp*_label and combined with **CpuID** value to create sensor. 
	* Default peci-cputemp sensor object include **Die, Corex, TJMax,TControl,DTS**, we can use hiddenprops        	  array to hidden unused sensor.
	* peci-dimmtemp sensor only create present dimm and naming with dimm channel index.
	* Check **DtsCritOffset** value.
	* Parsing **Threshold setting** from cofiguration array. If configuration array not include Threshold setting, it will search hwmon folder and check following file exist and set threshold value from those file
		* tempx_min : Warning Low value
		* tempx_max : Warning High value
		* tempx_lcrit : Critical Low value
		* tempx_crit : Critical High value 
              
> memo : CPUSensor use **labeHead** to parsing threshold , e.g Core 1 with labelHead "Core", label includes below object
> * Core
> * DTS
> * Die
> * Tjmax (Default set is hide)
> * Tcontrol (Default set is hide)
> * Tthrottle (Default set is hide)
> * DIMM

5. Create sensor with following name rule and establish monitor task for futher use.
`
"/xyz/openbmc_project/sensors/temperature/" + cpusensorname
`

## PSUsensor
### Support Type
```
static constexpr std::array<const char*, 13> sensorTypes = {
    "xyz.openbmc_project.Configuration.ADM1272",
    "xyz.openbmc_project.Configuration.ADM1278",
    "xyz.openbmc_project.Configuration.INA219",
    "xyz.openbmc_project.Configuration.INA230",
    "xyz.openbmc_project.Configuration.ISL68137",
    "xyz.openbmc_project.Configuration.MAX16601",
    "xyz.openbmc_project.Configuration.MAX20730",
    "xyz.openbmc_project.Configuration.MAX20734",
    "xyz.openbmc_project.Configuration.MAX20796",
    "xyz.openbmc_project.Configuration.MAX34451",
    "xyz.openbmc_project.Configuration.pmbus",
    "xyz.openbmc_project.Configuration.PXE1610",
    "xyz.openbmc_project.Configuration.RAA228228"};
```
SensorTypes definition is for entity-manager configuration json file use. Above array is support list for type setting. We can use this config to make custom function in PSUsensor service.

PSU still need to define pmbusname list as below array. 
```
static std::vector<std::string> pmbusNames = {
    "adm1272",  "adm1278",  "ina219",   "ina230",   "isl68137",
    "max16601", "max20730", "max20734", "max20796", "max34451",
    "pmbus",    "pxe1610",  "raa228228"};
```

### Sensor configuration parameter
1. **Name** : Defined PSU idenfication for sensor name. Sensor name combined with `${Name}_${labelname}` 
	 <br> For example , Name=PSU, psu fan1 sensor is show as
	 <br>`/xyz/openbmc_project/sensors/fan_tach/PSU_fan1`    
2. **Bus** : Device bus setting, used to compare with the detected device and identify the device matching the    sensor.
3. **Address** : Device slave address setting, used to compare with the detected device and identify the device matching the sensor.
4. **Labels**: Setup which PSU hwmon device we want to monitor.
5. **label_parameter**(Optional) : include **label_Name**, **label_Scale** , **label_Min**, **label_Max** <br>
 	e.g  pin_Name , pout_Max, iout_Scale.
6 **Threshold setting** (Optional) : PSUSensor use define **labeHead** to parsing threshold.  Need "Label" configure in threshold setting.


### Sensor Create step
1. Check hwmon device exist
	* Scan /sys/class/hwmon to find the hwmon device whose name matches the pmbusNames array element.
	* Check device's bus & slaveaddress. Below example is device with bus=12 & slave=0x59
	`
	lrwxrwxrwx    1 root     root             0 Aug 25 05:14 device -> ../../../12-0059
	`
 	* For each hwmon device, scan the sensor configuration array  defined by entity-manager and check **Sensor 	  type**, **Bus** , **Address** whether it match the hwmon device.	
2. Check configuration array for sensor create
	* SensorName is combined with "Name" and labelname **($Name_$labelname)**, which is the mapping name from **labelMatch** array. The activate sensor is defined in "Labels" content from configuration array.
		* For example, "Name" = "PSU0" with "Labels" = ["pin", "vin", "iout1", "temp1", "fan1"] <br>
		 Sensor name will be "PSU0_InputPower", "PSU0_Input_Voltage", "PSU0_Output_Current", "PSU0_Temperature", "PSU0_Fan_Speed1"
	* Check **label_key_parameter** include "_Name", "_Scale", "_Min", "_Max"
	* Parsing **Threshold setting** from cofiguration array. 
3. Check PSU PWM sensor and created sensor(use default pwm table fan1->pwm1 fan2->pwm2), NameRule: Pwm_$Name_Fan_1
4. Create sensor with following name rule and establish monitor task for futher use.
`
"/xyz/openbmc_project/sensors/$labeltype/" + name
`

## NVMETemp sensor
### Support Type
```
"xyz.openbmc_project.Configuration.NVMe"
```
SensorTypes definition is for entity-manager configuration json file use.

### Sensor configuration parameter
1. **Name** : Define the sensor name displayed in sensor API.
2. **Bus** : Device bus setting , used to compare with the detected device and identify the device matching the sensor.
3. **Threshold setting** (Optional)

### Sensor Create step
1. Check hwmon device exist
	* Scan **/sys/bus/i2c/device/i2c-$BUS/mux_device**, find out hwmon device and make NVMEMap
2. Check Sensor information from configuration array 
	* For each match device, find **"Name"** from sensor configuration array to define sensor api name
	* Establish NVME map
	
3. Create sensor with following name rule and establish monitor task for futher use.
`
"/xyz/openbmc_project/sensors/temperature/" + name
`


## Fansensor
### Support Type
```
static constexpr std::array<const char*, 3> sensorTypes = {
    "xyz.openbmc_project.Configuration.AspeedFan",
    "xyz.openbmc_project.Configuration.I2CFan",
    "xyz.openbmc_project.Configuration.NuvotonFan"};
```
SensorTypes definition is for entity-manager configuration json file use.

### Sensor configuration parameter
1. **Index** : Define Fan Tach setting (0-based).
2. **Name** : Define the sensor name displayed in sensor API.
3. **Bus** : Device bus setting  for I2cFan, used to compare with the detected device and identify the device matching the sensor.
4. **Address** : Device slave address setting for I2cFan, used to compare with the detected device and identify the device matching the sensor.
5. **Threshold setting** (Optional)
6. **Presence** (Optional) : Only need it if there is a present gpio pin. Sub-parmater includes **PinNmae** and **Polarity**.
	* GPIO Name must pre define in dts file
	* Polarity is the direction setting.
 
 	```
 	 "Presence": [
                        {
                            "PinName": "CPU1_PRESENCE",
                            "Polarity": "Low"
                        }
                     ],
	```
7. **Connector** (Optional) : Define pmw and tach connector.
	```
	"Connector": {
      	                 "Name": "Sys FAN Connector 9",
            	         "Pwm": 7,
		         "PwmName"(optional) : Pwm_1
	                 "Tachs": [
      	                            9
            	                  ]
	             }
	```

### Sensor Create step
1. Check hwmon device exist
	* Scan **/sys/class/hwmon**, find out hwmon device with fanx_input and check "device" 
	* For each hwmon device, scan the sensor configuration array  defined by entity-manager and check **Sensor type**, If sensor type = I2CFan, check **Bus** , **Address** whether it match the hwmon device. 	
		Fan type check example:

		`
		lrwxrwxrwx    1 root root 0 Aug 28 05:47 device -> ../../../1e786000.pwm-tacho-controller
		`
	* Type match rule
		* AspeedFan : 1e786000.pwm-tacho-controller or 1e610000.pwm-tacho-controller
		* NuvotonFan : f0103000.pwm-fan-controller
		* I2cFan : other string.
2. Check Sensor information from configuration array 
	* For each match device, find **"Index"** from sensor configuration array and check if it match fanx_input(index number-1 ) and use **"Name"** from sensor configuration array to define sensor api name , 
	* Check **Presence** pin setting.
	* Check pwm **Connector** setting.(For pwm sensor created)	
	
3. Create sensor and inventory api with following name rule and establish monitor task for futher use.
```
/xyz/openbmc_project/sensors/fan_tach/" + name
````

