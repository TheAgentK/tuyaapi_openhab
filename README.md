# tuya-mqtt
This project provides an MQTT gateway for locally controlling IOT devices manufactured by Tuya Inc and sold under many different brands.

Using this script requires obtaining the device ID and local keys for each of your devices after they are configured via the Tuya/Smart Life or other Tuya compatible app (there are many).  With this information it is possible to communicate locally with Tuya devices using Tuya protocol version 3.1 and 3.3 without using the Tuya Cloud service, however, getting the keys requires signing up for a Tuya IOT developer account or using one of several other alternative methods (such as dumping the memory of a Tuya based app running on Andriod).

Acquiring device keys is outside the scope of this project.  Please see the instructions at the TuyAPI project (on which this script is based) available at the TuyAPI project site:

https://github.com/codetheweb/tuyapi/blob/master/docs/SETUP.md.

Issues opened regarding acquiring keys will likely be closed without comment.  Please verify that your device can be queried and controlled via tuya-cli before opening an issue.  If your device can't be controlled by tuya-cli then it cannot be used with this project.

**!!!!!!!!!! Important information regarding the 3.0 release !!!!!!!!!!**\
The 3.0.0 release (Oct 17th, 2020) is a major refactor of the tuya-mqtt project and, as such, is a breaking release for all users of previous versions.  Almost everything about the project is different, including configuration method, topic names, etc.  Upgrading users should carefully read the instructions below and assume they are starting over from scratch.

## Installation
Download this project to your system into any directory (example below uses /opt/tuya-mqtt) and install tuyapi from the same folder that the tuya-mqtt.js is in
```
// switch to opt directory
cd /opt

// clone this project
git clone https://github.com/TheAgentK/tuya-mqtt

// change directory to the project directory
cd tuya-mqtt

//installs this project along with codetheweb/tuyapi project
npm install
```

## Configuration
Tuya-mqtt has two different configuration files.  The first is config.json, a simple file which contains settings for connection to the MQTT broker.  The second is devices.conf, a JSON5 formatted file which defines the Tuya devices that the script should connect to and expose via MQTT.  This file uses the same basic format as the "tuya-cli wizard" outputs when used to acquire the device keys, so it can be used as the basis for your tuya-mqtt device configuration.

### Seting up config.json:
```
cp config.json.sample config.json

// Edit config.json with your MQTT broker settings and save
nano config.json 
```

## Setting up devices.conf:
If you use the "tuya-cli wizard" method to acquire your device keys you can leverage the output of this tool as the start of your devices.conf file.  Otherwise, you want to create a file using a formate like this:
```
[
  {
    name: 'Tuya Device 1',
    id: '86435357d8b123456789',
    key: '8b2a69c9876543210'
  },
  {
    name: 'Tuya Device 2',
    id: 'eb532eea7d12345678abc',
    key: '899810012345678'
  }
]
```
Note that, because the format is JSON5, which is a superset of JSON, you can use standard, strict JSON syntax, or the more forgiving JSON5 format, or even mix and match in the same file.

While the above syntax is enough to create a working tuya-mqtt install with generic devices, the full power and simplicity of tuya-mqtt 3.0 is only unlocked by configuring device types to get .  Please see the full [DEVICES](docs/DEVICES.md) documenation for details.

### Start command
```
node tuya-mqtt.js

// To enable debugging output (required when opening an issue)
DEBUG=tuya-mqtt:* tuya-mqtt.js
```

### Usage Overview
Tuya devices are monitored and controlled using a simple API where a devices various functions are mapped to DPS (data point state) values stored in various key indexes.  For example, a simple on/off switch may have a single key, DPS 1, with a setting of true/false representing the on/off state of the device.  The device state can be read via this DPS 1 key, and, for values that can be changed, sending true/false to DPS 1 will turn the device on/off.  A simple dimmer might have the same DPS 1 key using true/false for on/off, and an additional DPS 2 key as a value from 0-255 representing the state of the dimmer.  More complex devices use more DPS keys with various values representing the states and control functions of the device.

tuya-mqtt exposes these DPS keys and their values via MQTT to allow for monitoring and control of such devices via anything that can connect to an MQTT broker.

### MQTT Topic Overview
The top level topics are created using the device name or ID as the primary identifier.  If the device name is available, it will be converted to lowercase and spaced replace with underscores('_') characters so, for example, if using the sample devices.conf file from above, the top level topic would be:
```
tuya/tuya_device_1/
```
If the device name was not available, it would instead use the device ID:
```
tuya/86435357d8b123456789/
```
All state/command topics are then built below this level.

Tuya-mqtt supports two distinct topic types for interfacing with and controlling devices. For all devices, the DPS topics are always published and commands are accepted, however, friendly topics allow mapping DPS values into device specific topic names allowing more control over allowed values, and allowing simple functions like math transforms, min/max limits, etc.  While it's always possible to use the DPS topics directly, friendly topics are the generally recommended approach but require you to create a template for your device if a pre-defined template for your device does not currently exist.

If you create a template for your device, please feel free to share it with the community as adding additional pre-defined devices is desired for future versions of tuya-mqtt.  There is a templates section of the project that you can submit a PR for your templates.

If you would like to use the raw DPS topics, please jump to the [DPS topics](#dps-topics) section of this document.

## Friendly Topics
As noted above, friendly topics are only available when using a pre-defined device template or, for the generic device, when you have defined a custom template for your device.  Friendly topics use the tuyq-mqtt templating engine to map raw Tuya DPS key values to easy to consume topics and transform the data where needed.

Another advantage of friendly topics is that not all devices respond to schema requets (i.e. a request to report all DPS topics the device uses).  Because of this, it's not always possible for tuya-mqtt to know which DPS topics to acquire state information from during initial startup.  With a defined template the required DPS keys for each friendly topic are configured and tuya-mqtt will always query these DPS key values during initial connection to the device and report their state appropriately.

Also, when using friendly topics, it is always possible to get the current state of all topics by sending a message to the device "command" topic with the mssage "get-states".  For more details on using friendly topics, please read the [DEVICES](docs/DEVICES.md) documentation.

## DPS Topics
Controlling devices directly via DPS topics requires enough knowledge of the device to know which topics accept what values.  There are two differnt methods interfacing with DPS values, the JSON DPS topic, and the individual DPS key topics.

### DPS JSON topic
The JSON DPS topic allows controlling Tuya devices by sending raw, Tuya style JSON messages to the command topic, and by monitoring for Tuya style JSON replies on the state topic.  You can get more details on this format by reading the [TuyAPI documentaiton](https://codetheweb.github.io/tuyapi/index.html), but, for example, to turn off a dimmer switch you could issue a MQTT message containing the JSON value {dps: 1, set: false} to the DPS/command topic for the device.  If you wanted to turn the dimmer on, and set brightness to 50%, you could issue separate messages {dps: 1, set: true} and then {dps: 2, set: 128}, or, the Tuya JSON protocol also allows setting multiple values in a single set command using the format {'multiple': true, 'data': {'1': true, '2': 128}}.  JSON state and commands should use the DPS/state and DPS/command topics respectively.  Below is an example of the topics:
```
tuya/dimmer_device/DPS/state
tuya/dimmer_device/DPS/command
```
### DPS Key topics
In addition to the JSON DPS topic, it's also possible to use the DPS key topics.  DPS key topics allow you to monitor and send simple bool/number/string values directly to DPS keys without having to use the Tuya JSON format, the conversion to Tuya JSON is handled by tuya-mqtt.  Using the example from above, turning on the dimmer and setting brightness to 50% you would simply issue the message "true" to DPS/1/command and the message "128" to DPS/2/command.
```
tuya/dimmer_device/DPS/1/state    <-- true/false for on/off state
tuya/dimmer_device/DPS/2/command  <-- 1-255 for brightness state
tuya/dimmer_device/DPS/1/state    <-- accept true/false for turning device on/off
tuya/dimmer_device/DPS/2/command  <-- accepts 1-255 for controlling brightness level
```
**!!! Important Note !!!**\
When sending commands directly to DPS values there are no controls on what values are sent as tuya-mqtt has no way to know what are valid vs invalid values.  Sending values that are out-of-range or of different types can cause unpredicatable behavior of your device, from causing timeouts, to reboots, to hanging the device.  While I've never seen a device not recover after a restart, please keep this in mind when sending commands to your device.

## Issues
Not all Tuya protocols are supported.  For example, some devices use protocol 3.2 which currently remains unsupported by the TuyAPI project due to lack of enough information to reverse engineer the protcol.  If you are unable to control your devices with tuya-mqtt please verify that you can query and control them with tuya-cli first.  If tuya-cli works, then this script should also work, if it doesn't then this script will not work either.

## Integration with other Home Automation tools
openHAB examples are [here](docs/openHAB.md).

## Contributors
- [TheAgentK](https://github.com/TheAgentK)
- [tsightler](https://github.com/tsightler)
- [Tycale](https://github.com/Tycale)
- [crashdummymch](https://github.com/crashdummymch)
- [GadgetAngel](https://github.com/GadgetAngel)


## Related Projects:
- https://github.com/codetheweb/tuyapi
- https://github.com/unparagoned/njsTuya
- https://github.com/clach04/python-tuya
- https://github.com/Marcus-L/m4rcus.TuyaCore
- Specs: https://docs.tuya.com/en/cloudapi/cloud_access.html

[![forthebadge](https://forthebadge.com/images/badges/made-with-javascript.svg)](https://forthebadge.com)
[![forthebadge](https://forthebadge.com/images/badges/built-with-love.svg)](https://forthebadge.com)
