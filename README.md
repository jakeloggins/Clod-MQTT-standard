
Clod MQTT Standard
==================

MQTT is a messaging protocol that is perfect for the Internet of Things. Messages are sent to topics, which is a string separated by forward slashes ` /just/like/this ` and contain payloads that can be a string ` "like this" ` or an object `{ "like": "this" }" `. 

A typical message might look something like this: ` /this/is/the/topic "and this is the payload" `. A device will receive the payload (`"and this is the payload"`) only if it is *subscribed* to `/this/is/the/topic `. 

An MQTT standard is just an agreed way to format the topic and payload so that users and devices can easily understand each other. Development around IoT, and espressif chips in particular, is constantly changing. Since Clod is a disorganized mess of other great open source software projects, the Clod MQTT Standard is designed to display information intuitively for users and allow multiple languages to understand it.

* Intutive when viewed in a continuous output stream

* Allow user customization

* Able to be parsed by multiple programming languages

* Easy for developers to grasp when developing new GUI or device sketches

* Can integrate seamlessly with third party services (slack chat, text message, twitter, etc)


Topic Format
------------

The Clod MQTT topic format is:

` /[location based path]/[command]/[device name]/[endpoint] `

Examples:

```
[location based path] > /house/upstairs/guestroom 
[command] > /control
[device name] > /myEspDevice
[endpoint] > /temperature
```

All together, that would look like this:

` /house/upstairs/guestroom/control/myEspDevice/temperature `


### Location based path

The intended use for Clod is multiple espressif-based IoT devices throughout the home. If a user has only one device, any topic format will do. But with multiple devices, location based topic paths offer the following advantages:

* A more intuitive experience when viewing a raw stream of messages on the broker. The path tells you the location and the command describes what is happening. The output gets more detailed as you read from left to right.

* The ability to easily view a subset of your network. For example, a subscription to /house/upstairs/# will show you everything that is going on within upstairs including devices at /upstairs/guestroom and upstairs/bathroom.

* A better foundation for the addition of global commands. In the python client examples, the placement of command in the middle of the path string allows the device to parse whether a global command applies to it's location. Eventually, Clod scripts will allow you to send a message like ` /global/house/upstairs/control/lights "off" ` to turn off all the upstairs lights. 

Any number of user-defined combinations are allowed. For example, `/house/upstairs/ ` or ` /house/upstairs/guestroom/closet/storagebox/shoebox/russian-nesting-dolls/large/medium/small ` are both perfectly fine location paths. First, they are descriptive and helpful to the user. Second, they get more specific and limiting as read from left to right. 

**Note**: The user is free to not use location based paths by simply assigning the same path to each each device, like `/default` or `/house`. 


### Command

There are four commands: control, confirm, errors, and log. Clod first uses the command to parse the topic. Once Clod recognizes the command, it reads everything to the left of it as part of the `[path]`, and everything to the right as the `[name]` and `[endpoint]`.

* A control command tells the device to do something. `[path]/control/[name]/[endpoint]`

* A confirm command informs Clod that the control message has been executed or updates Clod with new data. `[path]/confirm/[name]/[endpoint]` 

* When a device loses power or is disconnected from the MQTT broker, it sends a message to `[path]/errors/[name]/` to notify Clod. 

* The log command currently does nothing, but is reserved for future use. `[path]/log/[name]` 


### Name

Device names are provided by the user during the upload process and/or hardcoded into the sketch. Spaces in the name are allowed, but will be converted to [camelCase](https://en.wikipedia.org/wiki/CamelCase) and assigned to device_name_key in the device object. More information about device objects are provided below.


### Endpoint

A device can have multiple endpoints. Each endpoint represents a dashboard card that will be displayed for the user to control or view data from the device.

The device must subscribe to the `/[path]/control/[name][endpoint]` of each endpoint which may receive values from the user. For example, a toggle switch may change a value on the device therefore a subscription is necessary; however, an alert button which sends only messages from device to to the dashboard may not need a subscription because the device does not expect any values from the user.

```
Subscription address for endPoints on device:
/[path]/control/[name]/[endpoint name]
```

Upon receiving a new value from the user, the device **must** send back the new value or the appropriate value back to the dashboard on [path]/confirm/[name]/[endpoint]. This is because the dashboard will not reflect the new value change *unless* it is coming from the device.


```
Address: /[path]/confirm/[name]/[endpoint name]
Payload: {"value": "some new value here"}
```


### Startup and Scripts

Most routine communication between the user and devices are covered by the proceeding sections. However, a device's initial connection and use of the Clod Scripts require special topic formatting. This section simply notes the topic syntax for these processes. For an in-depth look at how these work and what they do, read the [walkthrough](https://github.com/jakeloggins/Clod-scripts). 

 * deviceInfo is where information about the device is exchanged between the user, devices, and Clod. 
   * ` /deviceInfo/[command]/[name] `

 * init is used by esp chips when they have first connected to Clod but have not yet gone through the upload process. 
   * ` /init/[command]/[chipID] `

 * the persistence script maintains information about all devices within Clod and makes it available to other devices and scripts 
   * ` /persistence/[command]/name `

 * the uploader script allows a user to customize and upload a sketch from the [Clod Sketch Library](https://github.com/jakeloggins/Clod) to an esp chip. 
   * ` /uploader/[command]/name `

 * the scheduler sends normal control commands to endpoints at specified times. 
   * ` /scheduler/[path]/[action type]/[name]/[endpoint]/[value] ` (Again, see the [walkthrough](https://github.com/jakeloggins/Clod-scripts#action-types-and-values) for more details)



Payload Format
--------------

### Device Objects

All of a device's information that is required by Clod can be found in its device object. For more information on how the device object changes during a device's life cycle, see the [walkthrough](https://github.com/jakeloggins/Clod-scripts). The device object is sent to `/deviceInfo/` whenever a user adds a device to the Crouton dashboard, a new sketch is uploaded, or after loss of connection to the MQTT broker. The device object is the primary method for Crouton to understand the device. It is the first message the device will send to Crouton to establish connection and also the message that describes the device and values to Crouton. 

Device object example:

```
{
  "deviceInfo": {
    "current_ip": "192.168.1.141",
    "type": "esp",
    "espInfo": {
        "chipID": "16019999",
        "board_type": "esp01_1m",
        "platform": "espressif",
        "flash_size": 1048576,
        "real_size": 1048576,
        "boot_version": 4,
        "upload_sketch": "basic_esp8266",
    },
    "device_name": "Floodlight Monitor",
    "device_name_key": "floodlightMonitor",
    "device_status": "good",
    "path": "/backyard/floodlight/",
    "card_display_choice": "custom",
    "endPoints": {
      "lastTimeTriggered": {
        "title": "Last Time Triggered",
        "card-type": "crouton-simple-input",
        "static_endpoint_id": "time_input",
        "values": {
          "value": "n/a"
        }
      },
      "alertEmail": {
        "title": "Alert Email",
        "card-type": "crouton-simple-input",
        "static_endpoint_id": "alert_input",
        "values": {
          "value": "your_email_address@gmail.com"
        }
      }
    }
  } 
}
```

* *type*: Specifies whether the device is an esp chip. Required by the uploader.
* *espInfo*: Information about the esp chip. Required by the uploader.
* *device_name*: A string that is the name for the device.
* *device_name_key*: A camelized version of the device_name so that the MQTT topic will not include a space.
* *device_status*: A string that describes the status of the device.
* *path*: Specifies the location based topic path.
* *card_display_choice*: Notes whether the user chose to specify custom endpoints during the upload process.
* *endPoints*: An object that configures each dashboard element Crouton will show. There can be more than one endPoint which would be key/object pairs within *endPoints*
* *card-type*: How data should be displayed from the endpoint on the dashboard. See next section.
* *description*: A string that describes the device (for display to user only)

**Note**: Both *device_name* and *endPoints* are required and must be unique to other *device_names* or *endPoints* respectively


### Card Types

Since Clod evolved from the Crouton dashboard, each endpoint must follow a Crouton card's payload format. For a full description of the available dashboard cards, go [here](https://github.com/jakeloggins/crouton-new#dashboard-cards). Most cards require sending a `values` object containing a single `value` key to the endpoint's topic. Some cards, such as the [Line Chart](https://github.com/jakeloggins/crouton-new#line-chart), have a more complicated `values` object.


### Updating Endpoints








### Updating device values

Updates to endpoints can come from user or the device. The message payload will be a JSON that updates the value. This JSON will be equivalent to the object of the key `values` within each endpoint. 

```json
Payload: {"value": 35}

An entry in endPoints:
"barDoor": {
  "title": "Bar Main Door",
  "card-type": "crouton-simple-text",
  "units": "people entered",
  "values": {
      "value": 34
  }
}
```

##### From Crouton

Crouton has the ability to update the value of the device's endPoints via certain dashboard cards. Therefore the device needs to be subscribe to certain addresses detailed in the Endpoints section below. The payload from Crouton is in the same format as the one coming from the device.

```
Address: /[path]/control/Kroobar/barDoor
Payload: {"value": 35}
```

##### From Device

To update values on Crouton from the device, simply publish messages to the outbox of the endPoint which Crouton is already subscribed to. The payload is just the same as the one coming from Crouton.

```
Address: /[path]/confirm/Kroobar/barDoor
Payload: {"value": 35}
```

### Last will and testament (LWT)

In order for Crouton to know when the device has unexpectedly disconnected, the device must create a LWT with the MQTT Broker. This a predefined broadcast that the broker will publish on the device's behalf when the device disconnects. The payload in this case can be anything as long as the address is correct.

```
Address: /[path]/errors/Kroobar
Payload: anything
```

