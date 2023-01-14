# Event forwarding

This document describes how Netlia system forwards events to the partner's application.

**The communication details are agreed between Netlia and the partner and set on the Netlia side.**

# Data transfer methods

Data is transmitted by a communication protocol in JSON format and describes events that occur in the NETLIA system.
Datetime entries are in UTC format specified in ISO 8601. The order of the parameters is not guaranteed and may change.

## Event forwarding by HTTP callback

Partner can specify the endpoint URLs to which events are sent as HTTP(S) POST requests. Requests have UTF-8 encoding and "application/json" Content-Type. The system tries to pass events in at-least-once mode. Therefore, an event may be delivered more than once. This situation can be handled by using the `EventId` item, which contains event identifier.

### HTTP request URL

It is possible to insert placeholders in the URL that will be replaced by the corresponding value.

| Parameter       | Description                    |
|:----------------|:-------------------------------|
| ProtocolVersion | communication protocol version |
| DeviceId        | device identifier              |
| DeviceType      | type of device                 |
| EventType       | type of event                  |

URL example (recommended settings):

`https://someaddress.com/event/v{ProtocolVersion}/{DeviceId}/{EventType}/`

For example, the following requests will be called for the specified URL:

https://someaddress.com/event/v1/abc123/event-start/

https://someaddress.com/event/v1/abc123/measured-humidity/

### HTTP request headers
Partner can specify additional configuration by adding an HTTP headers (key and value). This can, for example, solve authorization.

### Response for HTTP request

System expects 200-299 HTTP status in the response, which is used by the partner to acknowledge receiving the event. Any other response is interpreted as a nondelivery.

### Event non-delivery handling
 If the event transfer fails, the system attempts to transfer the event 10 more times with 5s delay after each attempt. The event is then dropped.

# Communication protocol
Data are always sent as a separate event. Events have a common parameter section.

Common parameters:

| Parameter       | Type    | Description                    |
|:----------------|:--------|:-------------------------------|
| ProtocolVersion | integer | communication protocol version |
| DeviceId        | string  | device identifier              |
| EventId         | string  | event identifier               |
| EventTime       | string  | time of event                  |
| DeviceType      | string  | type of device                 |
| EventType       | string  | type of event                  |

Other possible parameters not listed in the table depend on event and message type.

# Devices and supported event types

## Water detector
Detects the presence of water in a defined area.

![WaterDetector](../images/devices/water-detector.png)

If flooding occurs in the idle state, the `event-start` event is triggered. The device then checks every minute whether the flooding continues. If the flooding is detected even after 10 minutes, `event-continue` event is triggered. The device proceeds with periodical checks and triggers second `event-continue` event after another 10 minutes if the flooding is still detected. Another message is not sent unless the flooding ends, in which case `event-end` message is trasnmitted.

> DeviceType: water-detector

| EventType                                           | Description                                                                  |
|:----------------------------------------------------|:-----------------------------------------------------------------------------|
| [restart](#eventtype-restart)                       | Device restart.                                                              |
| [alive](#eventtype-alive)                           | Occurs at periodic intervals, confirming the functionality of the device.    |
| [transport](#eventtype-transport)                   | Switching to transport mode - inactive state with minimum power consumption. |
| [event-start](#eventtype-event-start)               | Flooding detected.                                                           |
| [event-continue](#eventtype-event-continue)         | Flooding still detected.                                                     |
| [event-end](#eventtype-event-end)                   | End of flooding.                                                             |

## Motion detector
Detects movement of an object which the device is attached to or placed on.

![MotionDetector](../images/devices/motion-detector.png)

Useful in situations when the information whether the object has moved is required. For example - door, window, office drawer, bag, car, motorcycle, bicycle, stroller, backpack, suitcase...

The device counts the number of input events, where the input event is a motion of the sensor. If the input event occurs in the idle state, the `event-start` event is triggered and a 10 minute interval is started. During the interval the detector counts the number of input events. At the end of the interval an `event-continue` message with the reached number is transmitted, another 10 minute interval is started, and the counter is reset. If no input event is detected during the interval, `event-end` message is transmitted and the device returns into idle state.

> DeviceType: motion-detector

| EventType                                           | Description                                                                  |
|:----------------------------------------------------|:-----------------------------------------------------------------------------|
| [restart](#eventtype-restart)                       | Device restart.                                                              |
| [alive](#eventtype-alive)                           | Occurs at periodic intervals, confirming the functionality of the device.    |
| [transport](#eventtype-transport)                   | Switching to transport mode - inactive state with minimum power consumption. |
| [tamper](#eventtype-tamper)                         | Opening or closing of device case, tampering switch manipulation.            |
| [event-start](#eventtype-event-start)               | Movement detected.                                                           |
| [event-continue](#eventtype-event-continue)         | Movement continues.                                                          |
| [event-end](#eventtype-event-end)                   | There was no movement for 10 minutes.                                        |

## Magnetic detector
Detects the insertion/removal of a magnet into/from it's proximity.

![MagneticDetector](../images/devices/magnetic-detector.png)

### Simple mode

Can be used to monitor the state (opened/closed) of doors, windows, or cabinets, or to detect the removal of one object from another.

The `event-start` event is triggered whenever the magnet is removed form detectors proximity. When the magnet is placed back, the `event-end` event is triggered.

> DeviceType: magnetic-detector-simple

| EventType                                           | Description                                                                  |
|:----------------------------------------------------|:-----------------------------------------------------------------------------|
| [restart](#eventtype-restart)                       | Device restart.                                                              |
| [alive](#eventtype-alive)                           | Occurs at periodic intervals, confirming the functionality of the device.    |
| [transport](#eventtype-transport)                   | Switching to transport mode - inactive state with minimum power consumption. |
| [tamper](#eventtype-tamper)                         | Opening or closing of device case, tampering switch manipulation.            |
| [event-start](#eventtype-event-start)               | Magnet removed, start of alarm.                                              |
| [event-end](#eventtype-event-end)                   | Magnet placed back, end of alarm.                                            |

### Continuous mode

Can be used to monitor the frequency of opening/closing of doors, covers, passing of moving parts.

The device counts the number of input events, where the input event is a removal of a magnet from detector's proximity (the placing of magnet into detector's proximity triggeres no responce). If the input event occurs with the detector in the idle state, the `event-start` event is triggered and a 10 minute interval is started. During the interval the detector counts the number of input events. At the end of the interval an `event-continue` message with the reached number is transmitted, another 10 minute interval is started, and the counter is reset. If no input event is detected during the interval, `event-end` message is transmitted and the device returns into idle state.

> DeviceType: magnetic-detector-continuous

| EventType                                           | Description                                                                  |
|:----------------------------------------------------|:-----------------------------------------------------------------------------|
| [restart](#eventtype-restart)                       | Device restart.                                                              |
| [alive](#eventtype-alive)                           | Occurs at periodic intervals, confirming the functionality of the device.    |
| [transport](#eventtype-transport)                   | Switching to transport mode - inactive state with minimum power consumption. |
| [tamper](#eventtype-tamper)                         | Opening or closing of device case, tampering switch manipulation.            |
| [event-start](#eventtype-event-start)               | Input event detected, alarm starts.                                          |
| [event-continue](#eventtype-event-continue)         | Further input events detected, alarm continues.                              |
| [event-end](#eventtype-event-end)                   | No further input events detected, alarm ends.                                |

## PIR detector
Detects personal presence in a specified area up to 10 meters distance. 

![PirDetector](../images/devices/pir-detector.png)

Can be used to detect the time and frequency of personal presence in a room or specified area.

The device counts the number of input events, where the input event is a movement in the detection area. If the input event occurs with the detector in the idle state, the `event-start` event is triggered and a 10 minute interval is started. During the interval the detector counts the number of inpupt events. At the end of the interval an `event-continue` message containing the reached number and timestamp of the last detected input event is transmitted, another 10 minute interval is started, and the counter is reset. If no input event is detected during the interval, `event-end` message is transmitted and the device returns into idle state.

> DeviceType: pir-detector

| EventType                                           | Description                                                                  |
|:----------------------------------------------------|:-----------------------------------------------------------------------------|
| [restart](#eventtype-restart)                       | Device restart.                                                              |
| [alive](#eventtype-alive)                           | Occurs at periodic intervals, confirming the functionality of the device.    |
| [transport](#eventtype-transport)                   | Switching to transport mode - inactive state with minimum power consumption. |
| [tamper](#eventtype-tamper)                         | Opening or closing of device case, tampering switch manipulation.            |
| [event-start](#eventtype-event-start)               | Input event detected, alarm starts.                                          |
| [event-continue](#eventtype-event-continue)         | Further input events detected, alarm continues.                              |
| [event-end](#eventtype-event-end)                   | No further input events detected, alarm ends.                                |

## SOS button
Device with a button to call for help or raise an alarm.

![SosButton](../images/devices/sos-button.png)
![AlarmButton](../images/devices/alarm-button.png)

The device triggers `event-start` event whenever the button is pressed.

> DeviceType: sos-button

| EventType                             | Description                                                                  |
|:--------------------------------------|:-----------------------------------------------------------------------------|
| [restart](#eventtype-restart)         | Device restart.                                                              |
| [alive](#eventtype-alive)             | Occurs at periodic intervals, confirming the functionality of the device.    |
| [transport](#eventtype-transport)     | Switching to transport mode - inactive state with minimum power consumption. |
| [event-start](#eventtype-event-start) | Button pressed, alarm started.                                               |

## Thermometer
Measures the ambient temperature.

![Thermometer](../images/devices/hygrometer-thermometer.png)

The device measures the temperature every minute. After X measurements, it calculates the average value and sends a `measured-temperature` message.

> DeviceType: thermometer

| EventType                                               | Description                                                                  |
|:--------------------------------------------------------|:-----------------------------------------------------------------------------|
| [restart](#eventtype-restart)                           | Device restart.                                                              |
| [alive](#eventtype-alive)                               | Occurs at periodic intervals, confirming the functionality of the device.    |
| [transport](#eventtype-transport)                       | Switching to transport mode - inactive state with minimum power consumption. |
| [measured-temperature](#eventtype-measured-temperature) | Measured values.                                                             |

### EventType measured-temperature

Triggered when the measured value is sent.

Additional parameters:

| Parameter   | Type  | Mandatory | Description          |
|:------------|:------|:----------|:---------------------|
| Temperature | float | yes       | measured temperature |

A sample of the `measured-temperature` message:

```yaml
{
    "ProtocolVersion": 1,
    "DeviceId": "abc123",
    "DeviceType": "thermometer",
    "EventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "EventTime": "2021-05-03T14:25:31.8437511Z",
    "EventType": "measured-temperature",
    "Temperature": 25.5
}
```

## Hygrometer/Thermometer
Measures ambient temperature and humidity.

![HumidityMeter](../images/devices/hygrometer-thermometer.png)
![HumidityMeter](../images/devices/motion-detector.png)

The device measures the temperature and humidity every minute. After X measurements, it calculates the average value and sends a `measured-humidity-temperature` event.

> DeviceType: hygrometer-thermometer

| EventType                                                                 | Description                                                                  |
|:--------------------------------------------------------------------------|:-----------------------------------------------------------------------------|
| [restart](#eventtype-restart)                                             | Device restart.                                                              |
| [alive](#eventtype-alive)                                                 | Occurs at periodic intervals, confirming the functionality of the device.    |
| [transport](#eventtype-transport)                                         | Switching to transport mode - inactive state with minimum power consumption. |
| [measured-humidity-temperature](#eventtype-measured-humidity-temperature) | Measured values.                                                             |

### EventType measured-humidity-temperature

Triggered when the measured value is sent.

Additional parameters:

| Parameter   | Type  | Mandatory | Description          |
|:------------|:------|:----------|:---------------------|
| Temperature | float | yes       | measured temperature |
| Humidity    | float | yes       | measured humidity    |

A sample of the `measured-humidity-temperature` message:

```yaml
{
    "ProtocolVersion": 1,
    "DeviceId": "abc123",
    "DeviceType": "hygrometer-thermometer",
    "EventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "EventTime": "2021-05-03T14:25:31.8437511Z",
    "EventType": "measured-humidity-temperature",
    "Temperature": 25.5,
    "Humidity": 27.5
}
```

# Common events

Events referenced from specific message types.

## EventType restart
Occurs when the device restarts. A restart can be caused by pressing the restart button located on the device's PCB, or by firmware during a hardware error or by changing the device configuration using a downlink API.

A sample of the event:
```yaml
{
    "ProtocolVersion": 1,
    "DeviceId": "abc123",
    "DeviceType": "magnetic-detector-simple",
    "EventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "EventTime": "2021-05-03T14:25:31.8437511Z",
    "EventType": "restart"
}
```

## EventType alive
Occurs at periodic intervals, confirming the functionality of the device.

A sample of the event:
```yaml
{
    "ProtocolVersion": 1,
    "DeviceId": "abc123",
    "DeviceType": "magnetic-detector-simple",
    "EventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "EventTime": "2021-05-03T14:25:31.8437511Z",
    "EventType": "alive"
}
```

## EventType transport

Occurs when the device enters transport mode, which happens when a new battery is inserted or when an appropriate command is sent through the downlink API.

A device in the transport mode has very low power consumption and sends no messages. To wake the device up from transport mode
the RESET button located on the PCB must be pressed.

A sample of the event:
```yaml
{
    "ProtocolVersion": 1,
    "DeviceId": "abc123",
    "DeviceType": "magnetic-detector-simple",
    "EventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "EventTime": "2021-05-03T14:25:31.8437511Z",
    "EventType": "transport"
}
```

[//]: # (## EventType downlink-acknowledge)

[//]: # (Informuje o doručení downlinkového commandu na zařízení.)

[//]: # ()

[//]: # (Ukázka zaslané události:)

[//]: # (```yaml)

[//]: # ({)

[//]: # (    "ProtocolVersion": 1,)

[//]: # (    "DeviceId": "abc123",)

[//]: # (    "DeviceType": "magnetic-detector-simple",)

[//]: # (    "EventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",)

[//]: # (    "EventTime": "2021-05-03T14:25:31.8437511Z",)

[//]: # (    "EventType": "downlink-acknowledge")

[//]: # (})

[//]: # (```)

## EventType tamper

Occurs when the device casing is opened or closed, or the safety switch is manipulated.

A sample of the event:

```yaml
{
    "ProtocolVersion": 1,
    "DeviceId": "abc123",
    "DeviceType": "magnetic-detector-simple",
    "EventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "EventTime": "2021-05-03T14:25:31.8437511Z",
    "EventType": "tamper"
}
```

## EventType event-start
It is triggered by the first occurrence of an input event such as flooding of the contacts, a magnet removal from detector's proximity or detection of movement.

A sample of the event:
```yaml
{
    "ProtocolVersion": 1,
    "DeviceId": "abc123",
    "DeviceType": "magnetic-detector-simple",
    "EventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "EventTime": "2021-05-03T14:25:31.8437511Z",
    "EventType": "event-start"
}
```

## EventType event-continue
Triggered when the input event occurs after `event-start` event.

A sample of the event:

```yaml
{
    "ProtocolVersion": 1,
    "DeviceId": "abc123",
    "DeviceType": "motion-detector",
    "EventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "EventTime": "2021-05-03T14:25:31.8437511Z",
    "EventType": "event-continue",
    "EventCount": 0,
    "SecondsSinceLastEvent": 0
}
```

The `EventCount` specifies the number of event repetitions since the last `event-start` or `event-continue` was sent.

`SecondsSinceLastEvent` specifies the number of seconds between the last event and the sending of the message.

## EventType event-end
Occurs at the end of the event. The situation when the `event-end` occurs is described for each device that produces this event.

A sample of the event:
```yaml
{
    "ProtocolVersion": 1,
    "DeviceId": "abc123",
    "DeviceType": "magnetic-detector-simple",
    "EventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "EventTime": "2021-05-03T14:25:31.8437511Z",
    "EventType": "event-end"
}
```
