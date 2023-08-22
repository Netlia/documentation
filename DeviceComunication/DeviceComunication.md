# Device communication

This document describes the behaviour of each device and a common communication protocol.

## Basic device features

 - The devices communicate with the server using a [LoRa](https://en.wikipedia.org/wiki/LoRa) or [NB-IoT](https://en.wikipedia.org/wiki/Narrowband_IoT) network. The choice of network depends on the type of equipment you purchase.
 - The devices communicate using a hexadecimal payload. Example: 020902050098434F0000020000.
 - Devices allow to [send](#messages-from-device-to-server) and [receive-messages](#receiving-messages-from-the-server) from the server.
 - The device contains 1 to N-sensors that monitor the environment. Based on the information from these sensors, the device decides what message to send to the server. 

## General device behaviour

Devices send three main types of messages - event, restart and measure. Event messages are sent
whenever the device detects an event (e.g., motion detection device detects motion, water detection device comes into contact with water). Restart messages inform the server that
the device has been restarted. Measure messages are sent with a specific time period and contain the measured data (e.g. Thermometer).
A detailed description of these and other messages can be found [here](#messages-from-device-to-server).

Depending on the settings and device type, acknowledgment of some messages is required from the server. Messages not requiring acknowledgment may be lost because the device does not try to resend them. The loosing of a message can be detected based on discrepancies between the message counter on the server and the [header counter](#0th-byte---counter-of-sent-messages) of subsequent messages.

Some messages may arrive on the server more than once. Duplicates can be detect using [header counter(#0th-byte---counter-of-sent-messages).

In addition to acknowledgment, the device can receive settings and commands from the server.
For more information, see [here](#receiving-messages-from-the-server).

## Restart and start of the device

Newly manufactured devices are always switched to transport mode. This mode is designed to put minimal load on the battery
and is primarily used for transportation. The device does not send any messages in transport mode and all its sensors are disabled.

In order to wake up the device a restart must be initialized by pressing the restart button.

1. Flashes LED 1x to indicate restart
2. Flashes LED 1x to indicate the end of initialization
3. [Battery status check](#battery-status-check)
4. Sends [restart message](#restart)
5. Sends 0-N [test messages](#test) (depending on device type)
6. Begins normal operation - event detection, measure messages transmittion etc.

Once the device is running, it can be restarted at any time using either standard or hard restart.

### Standard restart

The standard restart is triggered by pressing the button,
[server command](#structure-of-messages-received-from-the-server) or a device error.

After the device is restarted, it performs the following steps (if no error occurs):

1. Flashes LED 1x to indicate restart
2. Flashes LED 1x to indicate the end of initialization
3. [Battery status check](#battery-status-check)
4. Sends [restart message](#restart)
5. Sends [error message](#error) (if the restart was triggered by an error)
6. Sends 0-N [test messages](#test) (depending on device type)
7. Begins normal operation - event detection, measure messages transmittion etc.

All information, configuration and counter variables retained in the device memory are restored to their default values.There are only two exceptions to this - the device mode and the restart counter ([restart message](#restart)).

A restart will cause all processes to abort.

### Hard restart

A hard restart occurs by removing the battery or by completely discharging the battery and then draining the device's capacitors.
Capacitors in devices vary according to the device network used. For a hard restart, you need to proceed as follows:

* For LoRa devices, simply remove the battery and insert it back in
* For NB-IoT devices remove the battery, press the restart button and then reinsert the battery.

After the device is restarted, it performs the following steps (if no error occurs):

1. Flashes LED 1x to indicate restart
2. Flashes LED 1x to indicate the end of initialization
3. [Battery status check](#battery-status-check)
4. Sends [restart message](#restart)
5. Sends 0-N [test messages](#test) (depending on device type)
6. Begins normal operation - event detection, measure messages transmittion etc.

A hard restart restores all information, configuration and counter variables keept the device memory to their default values.

If a battery that is not fully charged (2.1 V - 2.9 V) is inserted into the device, an error is detected and dealt with as described in section [Fatal error](#fatal-error) - for error number 2. If the battery voltage is lower than 2.1 V which is considered as a critical condition, the device will switch to [transport mode](#transport-mode).

If any other error is detected, it is dealt with as described in the [error messages](#error) section.

### Battery status check

The device checks the battery status during restart, initialization, before sending a message or measuring data. If the battery voltage is below 2.1 V, the device will switch to [transport mode](#transport-mode).

### Transport mode

Transport mode is a special state of the device in which the device does not communicate and saves the battery. It is used for transporting or storing the devices. The device sends a [Transport](#transport) message when enters transport mode.

## Device state diagram

The following diagram describes the states that the device can get into.  The state automaton running on most devices isn't comprised of all states, but of a subset described states. Details regarding the individual devices can be found [here](#device).

The state machine does not contain a restart transition. The device can restart in any state.

The state machine also does not take into account the receipt of messages from the server.

![State machine](diagram.png)

## LED notifications

All devices have a notification LED that informs the user of various events.

Notification means a momentary activation of the notification component into an active state - LED lighting and subsequent pauses. The default LED lighting time is 200ms (further referred as LED blinking). The number of blinks differentiates individual notifications. ERROR and TRANS notifications are initiated by a long blink at the beginning (marked as L in the description below) with a duration of 1000ms to differentiate them.

Notification groups:
* RESET/INIT notifications - executed immediately after reset.
* EVENT notifications - when an event is detected (usually from a sensor)
* INFO notifications - notification while device is running (e.g. command execution)
* ERROR notifications - when an error is detected
* TRANS notification - when the device enters the transport mode

The following table describes the relation between the number of LED flashes and corresponding events:

| Notification group  | Number of flashes | Event                                                    |
|---------------------|-------------------|------------------------------------------------------------|
| RESET/INIT          | 1x                | [Restart](#standartní-restart)                             |
| RESET/INIT          | 1x                | Completion of initialization when the battery is inserted  |
| RESET/INIT          | 1x                | Completion of initialization after restart                 |
| EVENT               | 1x                | [Event](#event) detection                                  |
| ERROR               | L+2x              | Firmware error, unexpected state                           |
| ERROR               | L+3x              | Hardware error                                             |
| ERROR               | L+4x              | Discharged battery inserted                                |
| ERROR               | L+5x              | Low battery voltage detected *not currently implemented    |
| ERROR               | L+6x              | Network connection error                                   |
| TRANS               | L+10x             | Transition into [transport mode](#transport-mode)          |

Details regarding the error condition (ERROR notification group) can be delivered in the [error](#error) message.

## Messages from device to server

Messages are sent in hexadecimal format and consist of two parts - the header and the data.
The header contains general information and the message type. The message type then determines the format of the data section of the message.

>For NB-IoT devices where the payload is sent in a UDP datagram, there may be 8 chars containing the SIM card identifier before the payload itself. See [Device identification](#device-identification)  below.

## Device identification
Netlia, as the manufacturer, identifies the device with a serial number. This serial number cannot be obtained in any form from the message payload. Further identification data are supplied according to the type of network and the required configuration. It is up to the client to implement a process to identify the device according to their needs.

### LoRa devices identification
Netlia supplies the data needed for registration to the network server with the device. One of these items is the unique DevEUI identifier, which is passed along with the message from the network server.

### NB-IoT devices identification
The method of identification depends on the client's requirements and the device is configured accordingly during production.

In case of networks with private APN, the IP address uniquely identifies the device and can be used by the client to distinguish devices.

Alternatively, the UDP datagram can be extended with the SIM card identifier - IMSI. In this case, the datagram contains 16 characters at the beginning, representing 15 digits of the IMSI prefixed by 0. This solution is used in networks with a shared APN, where devices are hidden behind NAT.

Example of datagram with IMSI: `0AAAAAAAAAAAAAAAXXXXXXXXXX`  
`AAA...`: IMSI (contains numbers only)  
`XXX...`: Hexadecimal payload

## Header

The header has 7 bytes and contains general information that is common to all message types.
Example: 0209020. The following table contains an abbreviated description of the header. General description for individual bytes follows below.

| Position  | Abbreviated description                                                                                   |
|-----------|-----------------------------------------------------------------------------------------------------------|
| 0th byte    | Counter of sent messages                                                                                  |
| 1st byte    | Counter of received messages                                                                              |
| 2nd byte    | Battery voltage in the device                                                                             |
| 3rd byte    | Processor temperature                                                                                     |
| 4th byte    | RSSI                                                                                                      |
| 5th byte    | 0th bit - Request for message acknowledgement,<br/>1st bit - disconect<br/>Remaining bits - number of attempts to send a message |
| 6th byte    | Not used                                                                                                  |
| 7th byte    | Message type                               

### 0th byte - counter of sent messages

The device will increment the counter by 1 for each new message.
If the counter reaches 255, it is set to 0 the next time a message is sent.

The counter does not increment if the device tries to resend a message that has not been confirmed.

The primary purpose of the counter is to provide [idempotence](https://en.wikipedia.org/wiki/Idempotence) on the server.

Example:

| Message number| Value of 0th byte                                      |
|---------------|--------------------------------------------------------|
| 1             | 0x00  (remainder of header is ommitted for simplicity) |
| 2             | 0x01                                                   |
| 3             | 0x02                                                   |
| 255           | 0xFE                                                   |
| 256           | 0xFF                                                   |
| 257           | 0x00  (overflow occured and the counter is reset)      |


### 1st byte - counter of received messages

Set to 0 as default. After receiving a message from the server, it is set to the value received in the message from the server. More about this value [here](#0th-byte---identifier).

### 2nd byte - battery voltage in the device

Specifies the battery voltage value. If the device is powered from a socket it contains the voltage value of the power supply.

The voltage can range from 1.8 V to 4.35 V. A value of 0 corresponds to a voltage of 1.8 V and a value of 255 corresponds to a voltage of 4.35 V.

The current voltage can be calculated by the following formula: *received value in the message* / 100 + 1.8

### 3rd byte - processor temperature

Determines the processor temperature. This byte can take the value 0-160 and 255. The current temperature is calculated using the following formula: *current temperature* = *received value in the message* - 40.
If the temperature is higher than 120°C the sensor sends the value 0xFF (255). The total temperature range measured is -40 °C to 120 °C.

The device will never send a value of 161 to 254 that corresponds to temperatures higher than 120 °C.

### 4th byte - RSSI

Contains the [RSSI](https://en.wikipedia.org/wiki/Received_Signal_Strength_Indication) value measured when the previous message was sent from NbIOT device.
>LoRa devices do not support RSSI, the value contains 0x00. The reason for this is that RSSI itself does not indicate the quality of the signal and it is necessary to evaluate quality from the additional information transmitted from the network server with the message.

### 5th byte - acknowledgement and number of attempts to send a message

This byte contains more information about the message being sent. The contents of the byte vary depending on whether the device is using an NB-IoT or LoRa network.

#### 5th byte - LoRa

The byte contains information about the number of failed sends. The following table explains byte composition:

| Positions   | Short description |
|-------------|-----------------------------------------|
| 0th bit     | Not used                                |
| 1st bit     | Not used                                |
| 2nd-7th bit | Number of attempts to send a message    |

In the LoRa network, the device communicates with the network server, which then forwards messages to the server.
If the device fails to send a message to the network server, it will increase the "Number of attempts to send a message" counter and try to send the message again after a certain time interval (due to the size of the 6-bit number the counter range is 0 - 63, after overflow it increments again from 0).

After a certain number of unsuccessful sends, the device waits an increasingly long time before sending another message.

The following table describes the value of the 5th byte in relation to the number of attempts to send a message.

| Transmission attempt    | Value of 5th byte | Description                                                                                                                      |
|----------------------|----------------|----------------------------------------------------------------------------------------------------------------------------|
| 1st transmission attempt  | 000000--       | Device transmited 1st message. A dash represents two bits with no informational value. |
| 2nd  transmission attempt  | **000001**--   | Device waited for 3 s for acknowlegement from the network server. Repetition counter (bold) was incremented by 1.|
| 3rd  transmission attempt  | 000010--       | Device waited for 3 s for acknowlegement from the network server and the waiting period is going to increase for the following attempts.                            |
| 4th  transmission attempt  | 000011--       | Device waited for 15 min and the waiting period is going to increase for the following attempts.                                                        |
| 5th  transmission attempt  | 000100--       | Device waited for 15 min for acknowlegement from the network server.                                                                     |
| 6th  transmission attempt  | 000101--       | Device waited for 15 min for acknowlegement from the network server.                                                                     |
| 7th  transmission attempt  | 000110--       | Device waited for 60 min for acknowlegement from the network server.                                                                     |
| 8th  transmission attempt  | 000111--       | Device waited for 60 min for acknowlegement from the network server.                                                                     |
| 29th  transmission attempt | 011101--       | Device waited for 60 min for acknowlegement from the network server.                                                                     |
| 30th  transmission attempt | 011110--       | Device had waited waited until the next [alive message](#alive) was to be transmitted. <br/>It then transmitted this message instead of the alive message. |
| 31st  transmission attempt | 011111--       | Device waits until the next alive message is to be sent.                                                                  |

The device waits until it is time to send the [alive message](#alive) before any futher attempts to resend a message. The device tries to send the same message over and over again until it is restarted (battery drain or manual reset).

**The device ignores the request to send any other message until
the current message is delivered.**

#### 5th byte - NB-IoT

This byte contains two pieces of information. The following table explains the byte composition:

| Position  | Abbreviated description                       |
|-----------|-----------------------------------------------|
| 0th bit   | Request for message acknowlegement by server  |
| 1st bit   | Not used                                      |
| 2-7th bit | Number of message transmittion attempts       |

If the 0th bit is set to 1 the device requires an acknowledgement message.
The device waits 3s after sending and in this window
an acknowledgment, configuration message, or command must come from the server. For more on sending messages
to the device, see [here](#receiving-messages-from-the-server).

If the confirmation message does not arrive within 3 seconds, the device will increase the "Number of attempts to send a message" counter and try to send the message again after a certain time interval. (due to the size of the 6-bit number the counter range is 0 - 63, after overflowing it increments again from 0).

The following table describes the value of the 5th byte in relation to the number of attempts to send a message.

| Transmission attempt     | Value of 5th byte | Description                                                                                                                                      |
|----------------------|----------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| 1st  transmission attempt  | 000000-**1**   | Device transmitted a message and requires an acknowledgment (bold bit). A dash represents two bits with no informational value. |
| 2nd  transmission attempt  | **000001**-1   | Device waited for 3 s. Repetition counter (bold) was incremented by 1.  |
| 3rd  transmission attempt  | 000010-1       | Device waited for 3 s and the waiting period is going to increase for the following attempt. |
| 4th  transmission attempt  | 000011-1       | Device waited for 15 min.                                               |
| 5th  transmission attempt  | 000100-1       | Device waited for 15 min.                                               |
| 6th  transmission attempt  | 000101-1       | Device waited for 15 min.                                               |
| 7th  transmission attempt  | 000110-1       | Device waited for 60 min.                                               |
| 8th  transmission attempt  | 000111-1       | Device waited for 60 min.                                               |
| 29th  transmission attempt | 011101-1       | Device waited for 60 min.                                               |
| 30th  transmission attempt | 011110-1       | Device had waited waited until the next [alive message](#alive)<br/> was
to be transmitted. It then transmitted this message instead of the alive message.               |
| 31st  transmission attempt | 011111-1       | Device waits until the next alive message is to be sent.                |

The device waits until it is time to send the [alive message](#alive) before any other attempt to perform sending. The device tries to send the same message over and over again until it is restarted (battery drain or manual reset).

**The device ignores the request to send any other message until
the current message is delivered.**

### 6th byte - unused

### 7th byte - message type

The message type determines the content of the other bytes in the message. The following table lists the values this byte can take:

| Value   | Message type        |
|---------|---------------------|
| 0x01    | DownlinkAcknowlege  |
| 0x02    | Restart             |
| 0x03    | Test                |
| 0x04    | Error               |
| 0x05    | Event               |
| 0x07    | Alive               |
| 0x08    | Transport           |
| 0x09    | Measure             |


Downlink, Acknowledge, Restart, Test, Error, Alive, Transport and Event messages are identical for all devices. These messages are described in the following paragraphs on data.

The Measure message informs about the measured value (e.g. temperature for thermometer).
Measure message varies by device type and is described later in this document for each [device](#device) separately.

## Data

The following paragraphs describe the part of the message containing data. All of these messages begin with a common header which is [described above](#header).

### Acknowledging a message from the server

"Reception from server" acknowledgement message is sent by the device whenever configuration or command message is received from the server.
This message serves as an acknowledgement. For more information about sending a message from the server, see [here](#receiving-messages-from-the-server).

In the header it is marked with the value 0x01 in the 7th byte (Message type).

Content:

| Byte   | Description                       |
|--------|------------------------------|
| 0th byte | Not used, always 0xFF |
| 1st byte | Not used, always 0x00 |

### Restart

Always sent after [device restart](#restart-and-start-of-the-device).

It can be indetified by the value 0x02 in the 7th byte (Message type) of the payload.

Content:

| Byte    | Description                   |
|---------|-------------------------------|
| 0th byte  | Not used, always 0xFF       |
| 1st byte  | Not used, always 0x0C       |
| 2nd byte  | Device type                 |
| 3rd byte  | Device mode                 |
| 4th byte  | Service information         |
| 5th byte  | Service information         |
| 6th byte  | Service information         |
| 7th byte  | Service information         |
| 8th byte  | Service information         |
| 9th byte  | Service information         |
| 10th byte | Service information         |
| 11st byte | Service information         |
| 12th byte | number of restarts          |
| 13th byte | restart code                |

The device type and mode identify the device and its internal settings. Details about these values can be found in description of corresponding [device](#device).

The number of restarts indicates how many times the device has been restarted since [hard-restart](#hard-restart).

The restart code indicates the reason why the restart occurred. The byte can take the following values:

| Value | Description                                                                                          |
|---------|----------------------------------------------------------------------------------------------------|
| 0x00    | Hardware restart (usually triggered by a button on the device)                                     |
| 0x01    | Restart triggered by a button on the device                                                        |
| 0x02    | Restart triggered by [receiving of a message from the server](#receiving-messages-from-the-server) |
| 0x03    | Restart triggered by a detected error, followed by sending an [error message](#error)              |
| 0x04    | Restart triggered by a firmware change, occurs after a firmware update                             |
| 0xFF    | Restart triggered by an unknown error                                                              |

The 0th byte in the header, indicating the number of sent messages, is reset during restart and therefore it cannot be used to recognize a duplicate message. For deduplication it is possible to partly use the 12th byte containing the number of restarts, except for the case of hard restart (0x00 in the 13th byte) when this counter is reset.

### Test

The test message is sent after the device sends a restart message. The primary goal of the test messages is to receive the settings from the server at the LoRa network before the device begins normal operation. You can find more about this issue on the LoRa network [here](#lora). The message can also be used for signal quality control.

The default setting for most devices is to send one test message.

The device always waits 1 minute after sending a restart message and then sends a test message.
There is a delay of 1 minute between every two test messages.

Content:

| Byte   | Description             |
|--------|-------------------------|
| 0th byte | Not used, always 0xFF |
| 1st byte | Not used, always 0x00 |

### Error

An error message is sent from the device if a hardware error occurs.

Content:

| Byte                  | Description                           |
|-----------------------|---------------------------------------|
| 0th byte              | Error type                            |
| 1st byte              | Not used, always 0x04                 |
| 2nd byte - 5th byte   | Contains error register value         | 


Errors are divided into two types identifiable by 0th data byte - fatal(0th byte == 1) and standard (0th byte == 0).

Source of the error can be determined using a value of the error register.

#### Standard errors

If a standard error occurs, the device sends an error message and does not restart. If a normal error message is detected and sent 5 times, a fatal error is triggered.

When a standard error is detected, the LED always flashes 3 times.

| Error register                      | Description      |
|-------------------------------------|------------------|
| 00000000 00000000 00000000 00000000 | I2C Error        |
| 00000000 00000000 00000000 00000001 | UART Error       |
| 00000000 00000000 00000000 00000010 | EEPROM Error     |
| 00000000 00000000 00000000 00000100 | sensor Error     |
| 00000000 00000000 00000000 00001000 | actuator Error   |


#### Fatal error

If a fatal error occurs on the device, the device will reset and send an error message.
It then proceeds as normal - it sends a test message and then sends other messages depending on the type of device.

The error register contains information about what error occurred. The following table describes
the values and their meanings:

| Error register                      | Description                                                                       |
|-------------------------------------|-----------------------------------------------------------------------------------|
| 00000000 00000000 00000001 00000000 | 1 - Radio isn't working correctly                                                 |
| 00000000 00000000 00000010 00000000 | 2 - Inserted battery isn't fully charged. Checked only after hard restart.        |
| 00000000 00000000 00000100 00000000 | 3 - Main state machine is in an undefined state.                                  |
| 00000000 00000000 00001000 00000000 | 4 - Device cannot connect to the network after standart restart.                  |
| 00000000 00000000 00010000 00000000 | 5 - Standard error occured 5*                                                     |


If error numbers 1,3 and 5 occur 4 hours after restart, the device proceeds as follows:

1. The LED flashes X times in 10 cycles (according to the [notifications table](#led-notifications)) to indicate an error.
2. The device restarts and continues to operate as if it were [restarted](#standard-restart)
3. If the error still persists, the error processing is repeated

If the error number 1,3,4 and 5 occurs within 4 hours of restart, the device proceeds as follows:

1. The LED flashes X times in 10 cycles (according to the [notifications table](#led-notifications)) to indicate an error.
2. The device restarts and continues to operate as [usual](#standard-restart).
3. If the time has not passed, the device goes to sleep for 2 minutes and then repeats points 1 and 2.
4. If the time has passed, the device restarts and continues to operate as [usual](#standard-restart).
5. If the error still persists, the error processing is repeated again.


For error number 2, the device behaves according to the following list:

1. The user invokes [hard restart](#hard-restart).
2. The LED will flash 1x to indicate a restart.
3. The LED will flash 1x to indicate initialization.
4. The device detects in initialization that the battery is not fully charged.
5. The LED flashes 4 times in 10 cycles (according to the [notifications table](#led-notifications)) to indicate an error.
6. The device performs point 5 at two-minute intervals for the following 4 hours.
7. The device restarts.
8. The LED will flash 1x to indicate a restart.
9. The LED will flash 1x to indicate initialization.
10. The initialization is completed. Error 2 detection (Inserting a battery that is not fully charged) no longer occurs.
11. The device sends a restart message.
12. The device sends an error message.
13. The device sends a test message.
14. The device operates in the normal way.

### Alive

The Alive message serves as a notification that the device is OK and still transmitting. The Alive message
is sent only if the device has not communicated for a long time. By default, an alive message is sent if the device has not sent any message for approximately 12 hours. The interval can be set using [server command](#alive-messages-period).

> There is a bug in the current version of the firmware that may cause the alive message to come more often than the set interval.

Content:

| Byte     | Description                  |
|----------|------------------------------|
| 0th byte | Not used, always 0xFF        |
| 1st byte | Not used, always 0x00        |


### Transport

The transport message is sent when the device is switched to [transport mode](#transport-mode). The Transport mode is
a special state of the device in which the device does not communicate and saves the battery. It is usually used for
device transport or storage.

| Byte   | Description             |
|--------|-------------------------|
| 0th byte | Not used, always 0xFF |
| 1st byte | Not used, always 0x00 |


### Event

The Event message indicates that the device has experienced some form of event. What events the device responds to varies by device type. For example, a device that detects motion sends an event message when a motion is detected.

Some devices can also detect opening of device cover or free fall.
  This section describes the event message in general terms.
In [the following section of the documentation](#device) the individual
devices and their behaviour are described.

The null byte in the message specifies the type of event that occurred.

| Byte   | Description                                 |
|--------|---------------------------------------------|
| 0th byte | Event type - start, continue, end, tamper |


The values of the null byte can be as follows:

| Value   | Event    |
|---------|----------|
| 0x01    | start    |
| 0x02    | continue |
| 0x03    | end      |
| 0x04    | tamper   |


#### Event start, continue and end

If the main sensor on the device detects an event, it sends an event start message.
It then waits 10 minutes to record data about the next possible event. If no further events occur during this time period, the device sends an event end message. Otherwise, it sends an event continue message, waits another 10 minutes and repeats the procedure.

Exactly what caused the alarm varies by device. It can be triggered by detecting movement of the device (Move), water in the area (Water) or motion in the room (PIR). A more detailed description of the operation is provided in the [Devices](#device) section.

Some devices do not send all types of events or behave
differently. More about these differences in the [Devices](#device) section.

When an event start occurs, the device in the default state indicates the transition by flashing the LED 1x and a single beep (the same behavior as for tamper event - 0x04). The indication can be modified [by a configuration command from the server](#turning-led-flashing-and-buzzer-indication-during-event-start-onoff).

Event start, end and continue have the following format:

| Byte                | Description                       |
|---------------------|-----------------------------------|
| 1st byte            | Not used, always 0x03             |
| 2nd byte            | Number of events                  |
| 3rd byte - 4th byte | Seconds since last event (2B LSB) |

The number of events indicates how many events have occurred since the previous message was sent.
For event start and event end this value is always 0. For event continue this value is in the range 1-255 (0x01 - 0xFF).

Time since last event is a timestamp of most recently recorded event. For event start this value is always 0, as the message is sent immediately after the event is recorded. For the continue event, the value is always 0-10 minutes, due to the 10 minute message period. For event end, this value is always greater than 10 minutes. The value is a two-byte number (LSB).

#### Event tamper

This type of alarm indicates that the device cover has been opened or closed.

If the device is in it's normal state, the tamper event is inticated by a single LED flash and beep (the same behavior as event 0x01 - start). The indication can be modified [by a configuration command from the server](#turning-led-flashing-and-buzzer-indication-during-event-start-onoff).

Event format

| Byte   | Description                  |
|--------|------------------------------|
| 1st byte | Not used, always 0x03      |
| 2nd byte | 0x00 - Not used            |
| 3rd byte | 0x00 - Not used            |


## Device

The following paragraphs describe individual devices and their behaviour.

### Water detector

![WaterDetection](../images/devices/water-detector.png)

The device is used to detect water that the device has come into contact with. Event start is sent when the device is flooded with water. If the device detects water in the next 10 minutes,
the event continue message is sent. Event end message is sent if
no water is detected for 10 minutes.

* Supported events: Event start/continue/end, Restart, Alive, Transport, Error, DownlinkAcknowlege
* Device type (2nd byte in restart message): 0x01
* Default mode (3rd byte in restart message): 0x00 (currently no more modes)

The water device, unlike other devices, has a maximum number of event continue messages set. The device sends only 2 event continue messages and then waits for the alarm to end, i.e. it does not send any more event continue messages.

### Motion detector

![MovementDetection](../images/devices/motion-detector.png)

The device is used to detect the movement of the device itself or of the object on which the device is placed. Event start message is sent when the device detects motion. Event continue message is sent if another movement is detected in the next 10 minutes. An event end message is then sent if the device does not detect any movement for 10 minutes.

* Supported events: Event start/continue/end, Restart, Alive, Transport, Error, DownlinkAcknowlege
* Device type (2nd byte in restart message): 0x02
* Default mode (3rd byte in restart message): 0x00 (currently no more modes)

### Magnetic detector

![MagneticDetector](../images/devices/magnetic-detector.png)

The device can be used to monitor the frequency of opening/closing of doors, covers, passing of moving parts by detecting the presence of a magnetic field.
The device supports two modes. Continuous and simple mode. The user can switch between these modes using the [device mode setting server message](#setting-device-mode). Simple mode is chosen as a default.

If a magnetic field is detected shortly after restart, the first event message sent to the server is going to be the event end message. This behaviour is correct and follows the device's intended functionality.

* Supported events: Event start/continue/end, Restart, Alive, Transport, Error, DownlinkAcknowlege
* Device type (2nd byte in restart message): 0x06
* Default mode (3rd byte in restart message): 0x00 (continuous)

#### Continuous mode

If a dissapearance of a magnetic field is detected with the device in idle state, an Event start message is sent. Reappearance of the magnetic field is ignored, but all detected dissapearances of magnetic field are counted and Event continue message is sent after 10 minutes. If nothing happens within 10 minutes (the magnetic field doesn't disappear), the device sends an Event end message.

* Mode value (3rd byte in restart message): 0x00

#### Simple mode

Each magnet delay sends an Event start message. Each magnet approach sends an Event end message. In this mode, no alarms are counted or Event continue messages are sent.

* Mode value (3rd byte in restart message): 0x01

### PIR detector

![PirDetector](../images/devices/pir-detector.png)

Detects human movement or presence in a defined area up to 10m away using a passive infrared detector. When motion is detected by the sensor, the device sends an Event start message.
If it continues to detect motion, it sends Event continue messages at 10-minute intervals.
   The sensor sends an Event end message if no movement occurs for 10 minutes.

* Supported events: Event start/continue/end, Restart, Alive, Transport, Error, DownlinkAcknowlege
* Device type (2nd byte in restart message): 0x07
* Default mode (3rd byte in restart message): 0x00 (currently no more modes)

### SOS button

![SosButton](../images/devices/sos-button.png)
![AlarmButton](../images/devices/alarm-button.png)

A device with a button to call for help or raise an alarm.
The device sends an Event start message if someone presses the button.
The event end message is never sent.

* Supported events: Event start, Restart, Alive, Transport, Error, DownlinkAcknowlege
* Device type (2nd byte in restart message): 0x05
* Default mode (3rd byte in restart message): 0x00 (currently no more modes)

### Thermometer

![Thermometer](../images/devices/hygrometer-thermometer.png)
![Thermometer](../images/devices/motion-detector.png)

The device measures ambient temperature with an adjustable sampling period (default period 1 min). After X samples are acquired (default 10), an average value is calculated and Measure message is sent, containing the result.

* Supported events: Measure, Restart, Alive, Transport, Error, DownlinkAcknowlege
* Device type (2nd byte in restart message): 0x03
* Default mode (3rd byte in restart message): 0x00 (currently no more modes)

Measure message of the Thermometer device has the following format:

| Byte                  | Description            |
|-----------------------|------------------------|
| 0th byte              | Not used - always 0xFF |
| 1st byte              | Not used - always 0x14 |
| 2nd byte - 21st byte  | Measured temperatures  |


Bytes 2 - 21 contain the newest measurement and the previous 9 values. Older values are included in the message eliminate or at least minimize thhe loss of data due to loss of signal.

The messages are sorted from the most recent measurement to the oldest.

Each measured value in the message occupies 2 bytes and is coded using a two's complement - 1 in the highest bit indicates a negative number and 0 indicates a positive number. To obtain the temperature it is necessary to calculate the two's complement and divide the result by 100. For example, if byte 2 contains - 0x01 and byte 3 contains 0x00, the measured temperature = (256/100) or 2.56 °C. If the second byte contains 10000001 (0x81) and the third 0x00 then the measured temperature = (256/100) * - 1 or -2.56 °C.

The device allows you to set how often the measure message should be sent and also how many samples should be measured in between the messages. The default setting is 10 minutes and 10 samples, hence the temperature is measured every minute and after 10 minutes the device sends the average of the measured temperatures together with the 9 previous sent temperatures. How often the messages should be sent and how many times the temperature should be measured in this interval can be set by a command from the server. Read more [here](#sampling-period-for-temperature-and-humidity-devices-and-how-often-to-send-a-measure-message).

### Hygrometer/Thermometer

![Thermometer](../images/devices/hygrometer-thermometer.png)
![Thermometer](../images/devices/motion-detector.png)

The device measures ambient temperature and humidity with an adjustable sampling period (default period 1 min). After X samples are acquired (default 10), an average value is calculated and Measure message is sent, containing the result.

* Supported events: Measure, Restart, Alive, Transport, Error, DownlinkAcknowlege
* Device type (2nd byte in restart message): 0x04
* Default mode (3rd byte in restart message): 0x00 (currently no more modes)

The Measure message has the following format:

| Byte             | Description            |
|------------------|------------------------|
| 0th byte         | Not used - always 0xFF |
| 1st byte         | Not used - always 0x1E |
| 2nd byte - 31st byte | Measured values of temperature and humidity    |

The device's functionality is identical to that of the Thermometer described [here](#thermometer) with the difference that humidity is measured as well as temperature. Each measured value has a length of 3 bytes, where the first two bytes contain the temperature and the third byte contains the humidity.

The humidity is determined in % and can range from 0-100.

## Receiving messages from the server

The device can receive messages from the server. These messages can be divided into confirmation, configuration settings and commands. Confirmation is used to confirm messages from the device. Configuration settings allow you to set the device configuration and commands allow you to control the device in certain ways, such as forcing a restart.

All messages are [idempotent](https://en.wikipedia.org/wiki/Idempotence). The server can send the same message several times and the result will be the same. Idempotency is ensured by the nature of the messages, i.e. the device processes the message several times,
but all the commands and settings are created to avoid errors.

If the device successfully receives a configuration or command message, it sends back an acknowledgement to the server. The format of the confirmation message sent by the device is described [here](#acknowledging-a-message-from-the-server).

The device always sends the acknowledgement of receipt first and then processes the command, e.g. if the device receives a restart command, it sends the acknowledgement first and then restarts.

## LoRa and NB-IoT

The devices communicate over a LoRa or NB-IoT network. Sending messages to the device varies depending on which network the device uses. The network is selected when the device is manufactured and cannot be changed.

### NB-IoT

Whenever a device sends a [message requiring confirmation](#5th-byte---nb-iot), the server has the option to send
a message to the device. The device waits 5 seconds for a message from the server. The server must respond
with a command, configuration setting, or [confirmation](#message-acknowledgment) if it does not want to change anything on the device.
If the device does not receive any message within 5 seconds, it resends the original
message, more [here](#5byte---confirm-disconnect-and-number-of-attempts-to-send-message).

If the server sends multiple messages at once, the device will only process the first message
and ignores others.

For simplicity of implementation on the server, it is possible to acknowledge all received
messages. If the device does not require an acknowledgement, but the server sends one, the
acknowledgment is ignored.

The following diagrams show sending a command or configuration
messages to the device. An uplink is any message sent from a device to a server. A downlink is any message sent from a server to a device.

![NbIotCommunication](NbIot_communication.png)

A situation where an acknowledgment message is lost:

![NbIotCommunicationRetry](NBIot_retry.png)

### LoRa

In a LoRa network, the device communicates with a network server that takes care of sending messages
to the server and to the device. It acts as an intermediary between the server and the device.

If the server wants to send a message to the device, it must send it to the network server. The network server saves the message and then waits until the device sends a message. Then, the network server forwards the upling message from the device to the server and at the same time the message from the server is sent to the device.

The following diagrams show sending a command or configuration
messages to the device. An uplink is any message sent from a device to a server. A downlink is any message sent from a server to a device.

![LoraCommunication](Lora_communication.png)

Network server must always wait for the device to initiate the communication. Therefore, it is possible that the delivery of a message from the server might take a long time.

If the device successfully receives the message, it sends an acknowledgement to the server.
  The format of the confirmation message is described [here](#message-acknowledgment).

There is no point for the LoRa network to send an acknowledgement messages from the server to the device, even if the device [requests it](#header),
since the LoRa network takes care of the acknowledgement automatically (for messages requiring confirmation).

In some situations, the device must re-establish communication with the network server. This causes the messages that are stored on the network server to be deleted. Re-establishing communication happens whenever the device is restarted, when the radio modem is [restarted](#radio-restart) and in some cases when a message repeatedly fails to be delivered.

Multiple messages can be sent to the LoRa network server at the same time.
This functionality is usually not needed, but it is possible to implement it
and use [message-received-counter](#1st-byte---counter-of-received-messages) to identify which message
the device is acknowledging.

#### Implementation of sending messages to devices in LoRa network

We can implement sending messages to devices in the LoRa network in several ways. The simplest way is "marking messages". In this implementation, the server sends the downlink and marks it as "sent". After it receives the first uplink since the downlink was sent, it changes the status of the sent downlink to "will be acknowledged in the next message". The following uplink message should contain acknowledgment. Arrival of acknowledgment means that the downlink message has been successfully delivered.

![LoraMessageMarker](Lora_MessageMarker.png)

The following diagram shows the behaviour when a device
performs a join (re-establishing communication with the network server) and deletes a message that is waiting on the network server, or possible behaviour in case of a dropped message:

![LoraMessageMarker](Lora_MessageMarker_join.png)

The disadvantage of this solution is that it may take a long time to deliver the message when a new connection to the network server is established. Speed of the delivery can be increased by immediately resending the current message waiting for acknowledgment when a reset message arrives on the server, without waiting for another uplink message. This method is possible because a reset will always cause messages on the network server to be deleted.

## Structure of messages received from the server

Messages are divided into header and data. The header is common for all messages and the data part differs
according to the value of the 4th and 5th byte.

### Header of the message received from the server

#### 0th byte - identifier

Message identifier. The value received from the server is copied and returned as the [1st byte - counter of received messages](#1st-byte---counter-of-received-messages). This value can serve to identify messages that have been received by the device.

If the server never sends more than one message at a time, this identifier
is not needed.

#### 1st byte

Unused.

#### 2nd byte

Unused.

#### 3rd byte

Unused.

#### 4th byte - message category

This byte specifies the category of the downlink message the server needs to respond with. The category can take on the following values:

| Value of header's 4th Byte | Description             |
|----------------------------|-------------------------|
| 0x01                       | Message acknowledgement |
| 0x02                       | Commands                |
| 0x03                       | Unused                  |
| 0x04                       | Settings                |


The chapters below describe the different categories and types of messages they can contain.

#### 5th byte - message type

Together with the message category, it identifies the structure of the data part of the message.

### Data

The data part is divided into three categories - message acknowledgment, commands and settings. The following chapters describe these categories and messages they may contain.

#### Categories - message acknowledgment

The category is identified by the value 0x01 in the 4th Byte of the header and contains the following message types:

| Value of header's 5th byte | Description                                                 |
|----------------------------|-------------------------------------------------------------|
| 0x01                       | Message acknowledgement                                     |

#### Message acknowledgment

The message is identified by the value 0x01 in the 5th byte of the header.

Acknowledgement message is used to confirm the receipt of message that came from the device. Message acknowledgment only needs to be sent if the device uses the NB-IoT network and requests it in the [header of the uplink message](#header).

By default, the following messages must be acknowledged: Event start, Alive, Reset. In addition, the thermometer and hygrometer devices require that every sixth message is confirmed by the server. For other devices, every seventh message must be acknowledged.

Structure:

| Byte    | Description                    |
|---------|--------------------------------|
| 0th byte| Not used - always 0x00         |

#### Categories - commands

The category is identified by the value 0x02 in the 4th byte of the header and contains the following message types:

| Value of header's 5th byte    | Description                             |
|-------------------------------|-----------------------------------------|
| 0x02                          | Device restart                          |
| 0x03                          | Device switched into the transport mode |
| 0x04                          | Modem restart                           |

##### Device restart

The device restarts.

Structure:

| Byte    | Description                    |
|---------|--------------------------------|
| 0th byte| Not used - always 0x00         |


##### Switching the device into transport mode

The device switches to [transport mode](#transport-mode).

Structure:

| Byte    | Description                    |
|---------|--------------------------------|
| 0th byte| Not used - always 0x00         |

##### Radio restart

Restarts the device's radio modem.

Structure:

| Byte    | Description                    |
|---------|--------------------------------|
| 0th byte| Not used - always 0x00         |

#### Categories - settings

The category is identified by the value 0x04 in the 4th byte of the header and contains the following message types:

| Value of header's 5th byte | Description                                              |
|-------------------------|-------------------------------------------------------------|
| 0x01                    | Number of consecutive messages not requiring acknowledgement|
| 0x02                    | Turning message acknowledgement on/off                      |
| 0x03                    | Turning Event start acknowledgement on/off                  |
| 0x04                    | Alive message period                                        |
| 0x05                    | Measure message period                                      |
| 0x06                    | Turning LED flashing and buzzer indication during Event start on/off |
| 0x08                    | Turning LoRa ADR on/off                                     |
| 0x09                    | Setting LoRa dataRate                                       |
| 0x0A                    | Setting device mode                                         |
| 0x0B                    | Limiting the maximum number of Event continue messages      |
| 0x0C                    | Specifies the sampling period for temperature and humidity devices |
| 0x0E                    | Motion detection device sensitivity setting                 |


##### Number of consecutive messages not requiring acknowledgement

Determines message numbers which must always be acknowledged. The default value is 6 for thermometer and hygrometer,
7 for other devices. The default value can be set by sending the value 0xFF in the 1st byte.

Structure:

| Byte   | Description                                                          |
|--------|----------------------------------------------------------------------|
| 0th byte | Not used - always 0x01                                             |
| 1st byte | Number of consecutive messages not requiring acknowledgement, 0x00 - all messages require acknowledgement |


##### Turning message acknowledgement on/off

Sending this message to the device turns off or on the device's request for message acknowledgement.

Structure:

| Byte   | Description                                          |
|--------|------------------------------------------------------|
| 0th byte | Not used - always 0x01                             |
| 1st byte | 0x00 turns off message acknowledgement, 0x01 - turns on message acknowledgement |


##### Turning Event start acknowledgement on/off 

Turns the Event start acknowledgement off or on.

Structure:

| Byte   | Description                                          |
|--------|------------------------------------------------------|
| 0th byte | Not used - always 0x01                             |
| 1st byte | 0x00 turns off event acknowledgement, 0x01 - turns on event acknowledgement |


##### Alive messages period

Sets how long it takes to send an alive message since the last communication.
The default value is 12 hours since the last message was sent.

| Byte     | Description                  |
|----------|------------------------------|
| 0th byte | Not used - always 0x03       |
| 1st byte | Hours                        |
| 2nd byte | Minutes                      |
| 3rd byte | Seconds                      |


##### Measure message period    

Described [here](#sampling-period-for-temperature-and-humidity-devices-and-how-often-to-send-a-measure-message).

##### Turning LED flashing and buzzer indication during Event start on/off

| Byte   | Description                    |
|--------|--------------------------------|
| 0th byte | Not used - always 0x01       |
| 1st byte | Contains settings            |


Composition of 1st byte (LSB):

| Bit               | Description |
|-------------------|-------------|
| 0th bit           | LED         |
| 1st bit           | Not used    |
| 2nd bit           | Buzzer      |
| 3rd bit           | Not used    |
| 4th bit - 7th bit | Not used    |

By default, the byte is set to 0x05, that is 00000101 - both flashing and beeping are enabled during Event start.

##### Turning LoRa ADR on/off

Turns off or on [Lora ADR](https://lora-developers.semtech.com/documentation/tech-papers-and-guides/understanding-adr/).

It can only be used for LoRa network devices.

| Byte     | Description                  |
|----------|------------------------------|
| 0th byte | Not used - always 0x01       |
| 1st byte | 0x00 - off, 0x01 - on              |


##### Setting LoRa Data Rate

Sets [Data Rate](https://lora-developers.semtech.com/uploads/documents/files/Understanding_LoRa_Adaptive_Data_Rate_Downloadable.pdf).

It can only be used for LoRa network devices.

| Byte     | Description                  |
|----------|------------------------------|
| 0th byte | Not used - always 0x01       |
| 1st byte | Data Rate value (1 - 5)      |

##### Setting device mode

Sets the mode on the device.

If the message is sent to a device that does not support other modes, it will only restart the device.

| Byte     | Description                  |
|----------|------------------------------|
| 0th byte | Not used - always 0x01       |
| 1st byte | Device operation mode        |

Currently only Magnet devices support mode switching (0x00 - simple, 0x01 - continuous). When another mode number is sent, the device is set to continuous mode.

##### Limiting the maximum number of Event continue messages

Sets the maximum number of messages of type [Event continue](#event). The default value for Magnet, PIR and Move device is unlimited, but there is two message limit for Water device.

When the maximum number of event continues messages is reached, nothing is sent, until the Event end message. The message counter is reset with new Event start message and the device sends Event continue messages until Event end is detected or the maximum number is reached.

| Byte     | Description                                                            |
|----------|------------------------------------------------------------------------|
| 0th byte | Not used - always 0x01                                                 |
| 1st byte | Maximal number of Event continue messages. 0x00 sets unlimited number of messages |


##### Specifies the sampling period for temperature and humidity devices

Described [here](#temperature-and-humidity-devicess-sampling-period-and-setting-the-frequency-of-measure-messages).

##### Motion detection device sensitivity setting

Sets the sensitivity of the motion detection device.

| Byte     | Description                    |
|----------|--------------------------------|
| 0th byte | Not used - always 0x04         |
| 1st byte | ACC ZERO Setting               |
| 2nd byte | MAG ZERO Setting               |
| 3rd byte | ACC Count Setting              |
| 4th byte | MAG Count Setting              |


The following settings should be sufficient for normal use:

| Sesitivity| 0th byte | 1st byte | 2nd byte | 3rd byte | 4th byte |                      
|-----------|----------|----------|----------|----------|----------|
| Low       | 4        | 100      | 200      | 10       | 5        |
| Medium    | 4        | 75       | 100      | 10       | 5        |
| High      | 4        | 50       | 50       | 10       | 5        |

##### Temperature and humidity devices's sampling period and setting the frequency of Measure messages

The values 0x05 and 0x0C both refer to the thermometer and hygrometer and are closely related.
Both of these values must be set at once in the correct order and without restarting between processing of these messages.

First, it must be set how often the measure message should be sent and then what the number of samples per message should be.

The default setting is 10 minutes between messages and 1 minute between consecutive samplings. The device samples a signle value from the sensor every minute and averages the current batch of values every 10 minutes and sends a measure message including the result.

It is important that the measure message period is always an integer divisible by the sampling period in minutes, otherwise the device will send the message at the wrong interval.

Both of these values have the following format:

| Byte     | Description                    |
|----------|--------------------------------|
| 0th byte | Not used - always 0x03         |
| 1st byte | Hours                          |
| 2nd byte | Minutes                        |
| 3rd byte | Seconds                        |

If we set the period of sending measure messages to a value that is not divisible by the sampling period,
the device will behave according to the following example:

Let's imagine that we set the measure message period to 10 minutes and the measurement period to 4 minutes.
The device calculates the number of measurements that need to be taken before sending the message as follows:
10 / 4 = 2 (integer division is performed).

The device will then wait 4 minutes between measurements and send a message after every two measurements. Measure
messages will therefore be sent every 8 minutes.

## Simplified acknowledgement implementation
Since the device requires acknowledgement of some messages (specified 
by the [5th byte in the header](#5th-byte---nb-iot)), it is necessary to implement at least basic communication from the server to the device. These messages can be simply responded with a static payload `0000000001FF00` which contains an acknowledgement of the received message.