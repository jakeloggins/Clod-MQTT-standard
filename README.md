
Clod MQTT Standard
==================

MQTT is a messaging protocol that is perfect for the Internet of Things. Messages are sent to topics, which is a string separated by forward slashes ` /just/like/this ` and contain payloads that can be a string ` "like this" ` or an object `{ "like": "this" }" `. An MQTT standard is just an agreed way to format the messages so that users and devices can understand each other. Development around IoT, and espressif chips in particular, is constantly changing. Since Clod is a disorganized mess of other great open source software projects, the Clod MQTT Standard is designed to display information intuitively for users and allow multiple languages to understand it.

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


#### Location based path

The intended use for Clod is multiple espressif-based IoT devices throughout the home. If a user has only one device, any topic format will do. But with multiple devices, location based topic paths offer the following advantages:

* A more intuitive experience when viewing a raw stream of messages on the broker. The path tells you the location and the command describes what is happening. The output gets more detailed as you read from left to right.

* The ability to easily view a subset of your network. For example, a subscription to /house/upstairs/# will show you everything that is going on within upstairs including devices at /upstairs/guestroom and upstairs/bathroom.

* A better foundation for the addition of global commands. In the python client examples, the placement of command in the middle of the path string allows the device to parse whether a global command applies to it's location. Eventually, Clod scripts will allow you to send a message like ` /global/house/upstairs/control/lights "off" ` to turn off all the upstairs lights. 

Any number of user-defined combinations are allowed. For example, `/house/upstairs/ ` or ` /house/upstairs/guestroom/closet/storagebox/shoebox/russian-nesting-dolls/large/medium/small ` are both perfectly fine location paths. First, they are descriptive and helpful to the user. Second, they get more specific and limiting as read from left to right. 

**Note**: The user is free to not use location based paths by simply assigning the same path to each each device, like `/default` or `/house`. 


#### Command

There are four commands: control, confirm, errors, and log. Clod first uses the command to parse the topic. Once Clod recognizes the command, it reads everything to the left of it as part of the `[path]`, and everything to the right as the `[name]` and `[endpoint]`.

* A control command tells the device to do something. `[path]/control/[name]/[endpoint]`

* A confirm command informs Clod that the control message has been executed or updates Clod with new data. `[path]/confirm/[name]/[endpoint]` 

* When a device loses power or is disconnected from the MQTT broker, it sends a message to `[path]/errors/[name]/` to notify Clod. 

* The log command currently does nothing, but is reserved for future use. `[path]/log/[name]` 


#### Name

Device names are provided by the user during the upload process and/or hardcoded into the sketch. Spaces in the name are allowed, but will be converted to [camelCase](https://en.wikipedia.org/wiki/CamelCase) and assigned to device_name_key in the device object. More information about device objects are provided below.


#### Endpoint

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


#### Startup and Scripts

Most routine communication between the user and devices are covered by the proceeding sections. However, a device's initial connection and use of the Clod Scripts require special topic formatting. This section simply notes the topic syntax for these processes. For an in-depth look at how these work and what they do, read the [walkthrough](https://github.com/jakeloggins/Clod-scripts). 

 * deviceInfo is where information about the device is exchanged between the user, devices, and Clod. ` /deviceInfo/[command]/[name] `

 * init is used by esp chips when they have first connected to Clod but have not yet gone through the upload process. ` /init/[command]/[chipID] `

 * the persistence script maintains information about all devices within Clod and makes it available to other devices and scripts ` /persistence/[command]/name `

 * the uploader script allows a user to customize and upload a sketch from the [Clod Sketch Library](https://github.com/jakeloggins/Clod) to an esp chip. ` /uploader/[command]/name `

 * the scheduler sends normal control commands to endpoints at specified times. ` /scheduler/[path]/[action type]/[name]/[endpoint]/[value] ` (Again, see the [walkthrough](https://github.com/jakeloggins/Clod-scripts#action-types-and-values) for more details)



Intro to Device Objects
-----------------------

[walkthrough](https://github.com/jakeloggins/Clod-scripts)



Payload Format
--------------

Because Clod evolved from the Crouton dashboard, the payload format must match one of the Crouton cards. 








First, have Crouton and the device connected to the same MQTT Broker. The connection between the device and Crouton will be initiated from Crouton, therefore the device needs to subscribe to its own inbox.

```
Device should subscribe to the following:
/deviceInfo/control/[the device name]
```

Every time the device successfully connects to the MQTT Broker, it should publish its *deviceInfo*. This is needed for auto-reconnection. If Crouton is waiting for the device to connect, it will listen for the *deviceInfo* when the device comes back online.

```
Device should publish deviceInfo JSON once connected
/deviceInfo/confirm/[the device name]/
```

### DeviceInfo

The deviceInfo is the primary method for Crouton to understand the device. It is the first message the device will send to Crouton to establish connection and also the message that describes the device and values to Crouton. The primary object is *deviceInfo*. Within *deviceInfo* there are several keys as follows:

```json
{
  "deviceInfo": {
    "name": "Kroobar",
    "path": "/bar/front/entrance",
    "endPoints": {
      "barDoor": {
        "title": "Bar Main Door",
        "card-type": "crouton-simple-text",
        "units": "people entered",
        "function": "counter",
        "values": {
            "value": 34
        }
      }
    },
    "description": "Kroobar's IOT devices",
    "status": "good"
  }
}
```

* *name*: A string that is the name for the device. This is same name you would use to add the device
* *path*: Specifies the location to publish and subscribe on the mqtt broker
* *endPoints*: An object that configures each dashboard element Crouton will show. There can be more than one endPoint which would be key/object pairs within *endPoints*
* *function*: A string within an endpoint that is used to group together endpoints for global commands
* *description*: A string that describes the device (for display to user only)
* *status*: A string that describes the status of the device (for display to user only)

**Note**: Both *name* and *endPoints* are required and must be unique to other *names* or *endPoints* respectively

**Note**: There is now an additional method for adding and altering single card devices, discussed below. If you have multiple cards and store the deviceInfo JSON in your script, simply select "Auto Import" and type in the device name to add to the dashboard.

### Addresses

Addresses are what Crouton and the device will publish and subscribe to. They are also critical in making the communication between Crouton and the devices accurate therefore there is a structure they should follow.

```
/[path]/[command type]/[device name]/[endPoint name]
```

*command type*: Helps the device and crouton understand the purpose of a message. Generally, *control* is for messages going *to* the device and *confirm* is for messages *from* the device. Last will and testament messages are sent to *errors*. A final command type, *log*, is reserved for future use.

```
control, confirm, errors, log
```

*path*: a custom prefix where all messages will be published. Using location names is recommended. Command type words may not be used within the path.

```
ex: /house/downstairs/kitchen
```

*device name*: The name of the device the are targeting; from *name* key/value pair of *deviceInfo*

*endPoint name*: The name of the endPoint the are targeting; from the key used in the key/object pair in *endPoints*

**Note**: All addresses must be unique to one MQTT Broker. Therefore issues could be encounter when using public brokers where there are naming conflicts.

### Updating device values

Updating device values can come from Crouton or the device. The message payload will be a JSON that updates the value. This JSON will be equivalent to the object of the key *values* within each endPoint. However, only values that are being updated needs to be updated. All other values must be updated by the deviceInfo JSON.

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

