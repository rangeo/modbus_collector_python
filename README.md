# Modbus Application 
## Overview
Modbus application demonstrates how to acquire data from modbus slave and push the data to 
a cloud visualizer like freeboard.io. This app is built in dockerized development 
environment and it follows various development concepts recommended for IOx apps. 
Complete guide to IOx app development concepts can be found [here] (https://developer.cisco.com/media/iox-dev-guide-11-28-16/concepts/app-concepts/)

Broadly we will cover the following:

* Implementing modbus application in python
* Creating a docker image with python application
* Writing package descriptor file
* Creating an IOx application package from the docker image
* Deploying and testing on the target platform

## Developing the Application
### Workflow
Modbus application (app/main.py) polls the below mentioned data from holding registers
of modbus slave every few seconds. This data is then sent in JSON format to dweet.io
and backend web server. Freeboard.io can use this dweet or web server URL
as the data source to display real-time data on its dashboard

* Temperature (in Celcius)
* Humidity (in % Relative humidity)
* Pressure (in kPa)
* Key operation detected (UP/DOWN/LEFT/RIGHT/SELECT)
* Location of the device (Latitude and Longitude)

Modbus slave simulator code can be found at location  modbus_simulator/sync_modbus_server.py. 
Backedn web server code can be found at location cloud/cloudendpoint.py.

### Bootstrap configuration file
We can externalize certain variables whose values will need to be configurable at the time of 
deployment or can be updated while installed on the device. IOx enables this via bootstrap 
configuration file.  This file should be named ```package_config.ini``` and should be present in 
the root of the application package. Administration tools (Fog Director, Local Manager, ioxclient) 
provide ability to modify this file so that the values can be customized to a deployment environment.

For the modbus application, we have externalized the configuration parameters for modbus slave, dweet,
cloud backend web server and logging level using bootstrap configuration file. Then we can modify these parameters 
as applicable during runtime of the application.

```
File: app/project/package_config_ini

[sensors]
server: 127.0.0.1
port: 5020
poll_frequency: 10
temperature_reg: 0x01
humidity_reg:0x02
pressure_reg:0x03
geo_latitude_reg:0x04
geo_longitude_reg:0x06
key_operation_reg:0x08

[dweet]
# Set to no to disable it
enabled: yes
server: dweet.io
name: awake-transport

[server]
port: 9000

[cloud]
enabled: yes
server: 127.0.0.1
url: /
port: 10001
method: POST
scheme: http

[logging]
# DEBUG:10, INFO: 20, WARNING: 30, ERROR: 40, CRITICAL: 50, NOTSET: 0
log_level: 10
# Enable/disable logging to stdout
console: yes
```
### Environment variables
Cisco App hosting Framework (CAF) in IOx provides a set of environment variables for
applications. We have utilized ```CAF_APP_PATH``` and ```CAF_APP_CONFIG_FILE``` to obtain 
absolute path of the app and absolute path of the bootstrap configuration file.
 
```
# Get hold of the configuration file (package_config.ini)
moduledir = os.path.abspath(os.path.dirname(__file__))
BASEDIR = os.getenv("CAF_APP_PATH", moduledir)

tcfg = os.path.join(BASEDIR, "project",  "package_config.ini")
CONFIG_FILE = os.getenv("CAF_APP_CONFIG_FILE", tcfg)
```
We can find the entire list of environment variables provided by CAF [here.]
(https://developer.cisco.com/media/iox-dev-guide-11-28-16/concepts/app-concepts/#environment-variables)

### Application logging and persistent storage
Modbus app uses the persistent logging feature provided by IOx. In order to do that the app
writes the logs to the directory indicated by the environment variable ```CAF_APP_LOG_DIR```.

```
    log_file_dir = os.getenv("CAF_APP_LOG_DIR", "/tmp")
    log_file_path = os.path.join(log_file_dir, "thingtalk.log")
```

For further details on application logging refer the section [here] (https://developer.cisco.com/media/iox-dev-guide-11-28-16/concepts/app-concepts/#application-logging-and-persistent-storage)

### Safeguarding against flash wear
Due to constraints of the flash storage like limited PE cycles and susceptible to wear, we have
limited the size of log file to 1MB and rotate the logs with upto 3 backup log files.

```
 # Lets cap the file at 1MB and keep 3 backups
    rfh = RotatingFileHandler(log_file_path, maxBytes=1024*1024, backupCount=3)
    rfh.setLevel(loglevel)
    rfh.setFormatter(formatter)
    logger.addHandler(rfh)
```
Other recommendations for safeguarding against flash wear can be found [here]
(https://developer.cisco.com/media/iox-dev-guide-11-28-16/concepts/app-concepts/#safeguarding-against-flash-wear)

### Package Descriptor

IOx package descriptor is a file that describes the requirements, metadata about an IOx application or a service.

Every IOx package MUST contain a descriptor file.

More details here: https://developer.cisco.com/media/iox-dev-guide-11-28-16/concepts/package_descriptor/#iox-package-descriptor

```
descriptor-schema-version: "2.2"

info:
  name: Gateway_Agent
  description: "Gateway-as-a-Service agent application. Monitors southbound controllers and publishes data to the cloud"
  version: "2.0"
  author-link: "http://www.cisco.com"
  author-name: "Cisco Systems"

app:
  cpuarch: "x86_64"  
  type: docker

  resources:
    profile: c1.small

    network:
      -
        interface-name: eth0
        ports:
            tcp: [9000]

    devices:
      -
        type: serial
        label: HOST_DEV1
        usage: App monitors Weather and Location

# Specify runtime and startup
  startup:
    rootfs: rootfs.tar
    target: ["python", "/usr/bin/main.py"]
```

* Resource requirments
 * CPU : some units
* Network requirements
 * network->eth0
 

### Externalizing bootstrap configuration variables

Our app will need to interact with sensors. Also talk to a weather api server to get weather information. These details can be changed at the time of application deployment and therefore should be externalized into the bootstrap configuration file. By doing so, we will allow the application administrator configure the settings as applicable in the deployment scenario.

Create a `package_config.ini` file.

```
[sensors]
server: 127.0.0.1
port: 5020
poll_frequency: 10
temperature_reg: 0x01
humidity_reg:0x02
pressure_reg:0x03
geo_latitude_reg:0x04
geo_longitude_reg:0x06
key_operation_reg:0x08

[dweet]
# Set to no to disable it
enabled: yes
server: dweet.io
name: awake-transport

[server]
port: 9000

[cloud]
enabled: yes
server: 127.0.0.1
url: /
port: 10001
method: POST
scheme: http

[logging]
# DEBUG:10, INFO: 20, WARNING: 30, ERROR: 40, CRITICAL: 50, NOTSET: 0
log_level: 10
# Enable/disable logging to stdout
console: yes

```



##
###