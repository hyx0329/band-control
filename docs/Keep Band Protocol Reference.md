# Keep Band Protocol Reference

## Intro

This document is written to explain how to communicate with the
device (Keep B1 and B2) to read statistics and apply settings.

All information is obtained through reverse engineering. For example,
BLE traffic recording and Android App decompilation.

**Note: I have only Keep B2 and I'm not sure whether the following content applies to Keep B1.**

## Content Overview

1. [Basic Information about the Device](#Basic-Information-about-the-Device)
2. [Packet Specification for Standard Protocols](#Packet-Specification-for-Standard-Protocols)
3. [Protocol Types and Data Format](#Protocol-Types-and-Data-Format)
   1. [Standard Protocols](#Standard-Protocols)
   2. [Special Protocols](#Special-Protocols)
4. [Notes & Comments](#Notes--Comments)
5. [Change Log](#Change-Log)

## Basic Information about the Device

About the product (Keep B2):

**Property** | **Content**
:- | :-
**Product Name** | Keep B2
**Manufacture** | Keep (keep.com)
**Launch Date** | 2020.11
**Features** |  heart rate monitoring, 5 ATM water resist, oxygen saturation test, motion detect, colorful touch screen, blablabla
**Variants** | Black, Green, Blue (only straps are different)
**Communication protocol** | BLE

About the BLE service on the device:

+ Primary service UUID: `0000190a-0000-1000-8000-00805f9b34fb`
  - Characteristic to write data: `00000001-0000-1000-8000-00805f9b34fb`
  - Characteristic notifications from: `00000002-0000-1000-8000-00805f9b34fb`

## Packet Specification for Standard Protocols

A `protocol` defines how data is transferred between the host and the device.
You can call each `protocol` a `command` if that helps understand.

For standard protocol, there are 2 variants of packet: the *short* one and the *long* one.
Each of them consists of packet head, packet length, protocol ID and the transferred parameters.
The bit order is little endian unless otherwise specified.

### The *short* packet

A short packet is defined in the following pattern:  
`[Head:0xC6][Length][Protocol ID][return code if response][Parameters(optional)][Checksum]`.

Segment | Length(byte) | Comment
:- |:- |:-
**Head**          | 1         | Packet head marker (**0xC6** here)
**Length**        | 1         | The length of payload (protocol ID and params, and return code if exists)
**Protocol ID**   | 1         | The protocol ID used
**Return code**   | 0 or 1    | ONLY APPEARS IN RESPONSES
**Parameters**    | dynamic   | Data to transfer for specific protocol ID
**Checksum**      | 1         | Checksum (see [below](#About-the-Checksum) for detail)


### The *long* packet

In order to meet the requirement of MTU limit for BLE communication, Keep designed a way to transfer
data stream by splitting it into small segments. So, the *long packet* introduced.

There are 3 types of *long* packets: *start*, *middle* and *end*. Each type has its own head value. The overall structure is similar to the *short* packet.

Note that it doesn't necessary to have a *middle* packet in one transaction, but the *start* and *end* are necessary.

For *start* packet:
Segment | Length(byte) | Comment
:- |:- |:-
**Head**          | 1         | Packet head marker (**0xC7** here)
**Length**        | 2         | The length of payload (protocol ID and params, and return code if exists)
**Protocol ID**   | 1         | The protocol ID used
**Return code**   | 0 or 1    | ONLY APPEARS IN RESPONSES
**Parameters**    | dynamic   | Data to transfer for specific protocol ID (segment)

For *middle* packet:
Segment | Length(byte) | Comment
:- |:- |:-
**Head**          | 1         | Packet head marker (**0xC8** here)
**Parameters**    | dynamic   | Data to transfer for specific protocol ID (segment)

For *end* packet:
Segment | Length(byte) | Comment
:- |:- |:-
**Head**          | 1         | Packet head marker (**0xC9** here)
**Parameters**    | dynamic   | Data to transfer for specific protocol ID (segment)
**Checksum**      | 1         | Checksum (see [below](#About-the-Checksum) for detail)

When transferring data using *long* packets asynchronizedly, pay attention to the packet order, as there no sequence info.

### About the Checksum

The checksum is calculated in the following procedure:
1. Calculate the sum of data from 3 segments byte by byte.
    The 3 segments are *Length*, *Protocol ID* and *Parameters*(add *return code* if available)
2. The checksum is the lowest 8 bit from the result above. In another word, calculate 'RESULT AND 0xFF'.

That's it.

## Protocol Types and Data Format

By decompiling the official app, I've found a lot of protocols, not all of them are currently used by the app though.

### Standard Protocols

Standard protocols follow the packet specification written [above](#Packet Specification for Standard Protocols).

I mark the parameter structure in a sequence like this: `[parameterName:length(byte)]`

Unless otherwise specified, all values are *unsigned intergers*.  
Unless otherwise specified, each command expects a response with at least a return code.

Here is a complete list of discovered standard protocols/commands:
|Protocol ID |Tag | Verified on Keep B2
|:-:|:- |:-:
|0x01|[BIND_WITHOUT_USER](#0x01-BIND_WITHOUT_USER) | ✅
|0x04|[SET_USER](#0x04-SET_USER) | ✅
|0x05|[UNKNOWN_05](#0x05-UNKNOWN_05)
|0x06|[UNKNOWN_06](#0x06-UNKNOWN_06)
|0x07|[UNKNOWN_07](#0x07-UNKNOWN_07)
|0x08|[GET_USER](#0x08-GET_USER)
|0x09|[GET_WEARING_STATUS](#0x09-GET_WEARING_STATUS)
|0x10|[GET_DEVICE_INFO](#0x10-GET_DEVICE_INFO) | ✅
|0x11|[GET_SYSTEM_STATUS](#0x11-GET_SYSTEM_STATUS) | ✅
|0x12|[SET_TIME](#0x12-SET_TIME)
|0x13|[SET_ALARM_CLOCK](#0x13-SET_ALARM_CLOCK) | ✅
|0x14|[GET_ALARM_CLOCK](#0x14-GET_ALARM_CLOCK) | ✅
|0x15|[GET_FEATURES_STATUS](#0x15-GET_FEATURES_STATUS) | ✅
|0x16|[SET_FEATURES_STATUS](#0x16-SET_FEATURES_STATUS) | ✅
|0x17|[SET_VIBRATION](#0x17-SET_VIBRATION)
|0x18|[NOTIFICATION](#0x18-NOTIFICATION) | ✅
|0x19|[SET_DIAL](#0x19-SET_DIAL) | ✅
|0x1A|[OTA](#0x1A-OTA)
|0x1B|[RESET](#0x1B-RESET) | ✅
|0x1C|[CLEAR_DATA](#0x1C-CLEAR_DATA) | ✅
|0x1D|[GET_KEEP_KEY_SEED](#0x1D-GET_KEEP_KEY_SEED)
|0x1E|[SET_KEEP_KEY](#0x1E-SET_KEEP_KEY)
|0x1F|[GET_FIRMWARE_EVENT](#0x1F-GET_FIRMWARE_EVENT)
|0x20|[SET_TIME_WITH_TIMEZONE](#0x20-SET_TIME_WITH_TIMEZONE) | ✅
|0x21|[SET_SPORT_COEFFICIENT](#0x21-SET_SPORT_COEFFICIENT)
|0x23|[SET_GENERAL_STATUS](#0x23-SET_GENERAL_STATUS)
|0x24|[GET_GENERAL_STATUS](#0x24-GET_GENERAL_STATUS)
|0x30|[SET_WORKOUT_NOTICE](#0x30-SET_WORKOUT_NOTICE)
|0x40|[GET_STEP_DATA](#0x40-GET_STEP_DATA) | ✅
|0x41|[GET_LAST_WORKOUT_LOG](#0x41-GET_LAST_WORKOUT_LOG) | ✅
|0x42|[DELETE_LAST_WORKOUT_LOG](#0x42-DELETE_LAST_WORKOUT_LOG) | ✅
|0x45|[GET_STEP_DATA_COMPRESSED](#0x45-GET_STEP_DATA_COMPRESSED)
|0x51|[GET_DAILY_CALORIES_DEBUG_INFO](#0x51-GET_DAILY_CALORIES_DEBUG_INFO)
|0x52|[GET_DAILY_CALORIES](#0x52-GET_DAILY_CALORIES)
|0x53|[GET_DAILY_CALORIES_B2](#0x53-GET_DAILY_CALORIES_B2)
|0x60|[GET_SLEEP_DATA](#0x60-GET_SLEEP_DATA) | ✅
|0x62|[GET_WHOLE_DAY_SLEEP_DATA](#0x62-GET_WHOLE_DAY_SLEEP_DATA)
|0x70|[START_WORKOUT](#0x70-START_WORKOUT) | ✅
|0x71|[STOP_WORKOUT](#0x71-STOP_WORKOUT) | ✅
|0x72|[RECEIVE_NOTIFY_STEP](#0x72-RECEIVE_NOTIFY_STEP)
|0x73|[NOTIFY_STEP_SWITCH](#0x73-NOTIFY_STEP_SWITCH)
|0x82|[RECEIVE_HEART_RATE](#0x82-RECEIVE_HEART_RATE) | ✅
|0x83|[GET_HEART_RATE_DATA](#0x83-GET_HEART_RATE_DATA) | ✅
|0x85|[GET_LAST_HEART_RATE](#0x85-GET_LAST_HEART_RATE) | ✅
|0x86|[GET_HEART_RATE_DATA_COMPRESSED](#0x86-GET_HEART_RATE_DATA_COMPRESSED)
|0x93|[START_TRACK_RECORDER](#0x93-START_TRACK_RECORDER)
|0x94|[STOP_TRACK_RECORDER](#0x94-STOP_TRACK_RECORDER)
|0x95|[RECEIVE_TRACK](#0x95-RECEIVE_TRACK)
|0xA3|[CHECK_RESOURCE](#0xA3-CHECK_RESOURCE)
|0xA4|[PREPARE_RESOURCE](#0xA4-PREPARE_RESOURCE)
|0xA5|[TRANSFER_RESOURCE](#0xA5-TRANSFER_RESOURCE)
|0xB0|[SET_NO_DISTURB](#0xB0-SET_NO_DISTURB) | ✅
|0xF0|[SHUT_DOWN](#0xF0-SHUT_DOWN)
|0xF1|[GET_LOG_LONG](#0xF1-GET_LOG_LONG)
|0xF2|[GET_LOG_DATA](#0xF2-GET_LOG_DATA)
|0xF3|[DELETE_LOG_DATA](#0xF3-DELETE_LOG_DATA)
|0xF8|[SET_ALGORITHM_ACQUISITION_TEMPLATE](#0xF8-SET_ALGORITHM_ACQUISITION_TEMPLATE)
|0xF9|[FETCH_RAW_DATA_SUMMARY_LIST](#0xF9-FETCH_RAW_DATA_SUMMARY_LIST)
|0xFE|[GET_MTU](#0xFE-GET_MTU) | ✅
|0xFF|[AUTHENTICATION](#0xFF-AUTHENTICATION)

#### 0x01 BIND_WITHOUT_USER

Params: No param.

This command is sent to activate the band when it haven't been paired or it has been reseted. Otherwise the band will stuck with an QR code on its screen and will not be usable.

It's safe to send this command without checking if the device is binded, as it will not delete any existing data.

#### 0x04 SET_USER

Set user specified infomation, generally body statistic.

Params: `[height:1][weight:1][age:1][runningStepLength:1][gender:1][hikingStepLength:1][maximumHeartRate:1][restingHeartRate:1]`

Unit for all length values is centimeter.

The gender is boolean, `False` for male and `True` for female.

#### 0x05 UNKNOWN_05

TBC. Have response, not used or found in the app's code.

#### 0x06 UNKNOWN_06

TBC. Have response, not used or found in the app's code.

#### 0x07 UNKNOWN_07

TBC. The band will respond a packet using this protocol ID each time you connect to it. Just ignore it.

#### 0x08 GET_USER

TBC. I think it should respond data in a format like 0x04, but actually not.

#### 0x09 GET_WEARING_STATUS

TBC, no response

#### 0x10 GET_DEVICE_INFO

Params: No params.

Response params: `[manufactureName:16][serialNumber:16][hardwareRevision:2][firmwareRevision:3][hardwareAddress:6]`

`manufactureName` and `serialNumber` are strings.

Revision number example: receive `00 07 02` for ver. 2.7.0, `00 02` for ver. 2.0

#### 0x11 GET_SYSTEM_STATUS

Response params: `[ifIsCharging:1][batteryPercentage:1][statusByte:1]`

`ifIsCharging` is boolean,  
`batteryPercentage` is a percentage number ready to use,  
`statusByte` may have somthing todo with the OTA, or device runtime state, TBC.

#### 0x12 SET_TIME

TBC

Use [SET_TIME_WITH_TIMEZONE](#0x20-SET_TIME_WITH_TIMEZONE) instead.

#### 0x13 SET_ALARM_CLOCK

Params: an alarm data list, have at most 5 entries. This list will replace the alarm list stored on the device.

For each alarm entry, the format is: `[alarmTime:2][repeat:1]`

`alarmTime` is the number of minute counted from 00:00. 
For `repeat`, from HSB to LSB: N/A(Zero),Sat,Fri,Thu,Wed,Tue,Mon,Sun. All zero means the alarm will only ring once.

Snooze will not affect `alarmTime`.

#### 0x14 GET_ALARM_CLOCK

Response params: a list of alarms, the format is the same as 0x13's

#### 0x15 GET_FEATURES_STATUS

Read a lot of features' status.

Response params:
```
[switchByte0:1]
[switchByte1:1]
[wakeOnWristRaiseStatus:1]
[nightModeStartHour:1]
[nightModeEndHour:1]
[isStepGoalNoticeEnable:1]
[isHeartRateWarningNoticeEnable:1] //boolean
[heartRateWarningValue:1]
[standReminderStatus:1]
[standReminderStartHour:1]
[standReminderEndHour:1]
[lunchBreakStartHour:1]
[lunchBreakEndHour:1]
[sleepGoalValueUnsigned:2]
```

About some switch bytes (N/A default 0):
Param  \  bit | 8 | 7 | 6 | 5 | 4 | 3 | 2 | 1 |
:- |:- |:- |:- |:- |:- |:- |:- |:- |
switchByte0 | SMS notify switch|QQ notify switch|Wechat notify switch|if on right hand|(perhaps) if is binded|(perhaps) low power notify switch| alarm switch| call notify switch |
switchByte1 | motion detect switch | N/A | N/A | N/A | N/A | N/A | (perhaps) call notify related | other message notify switch |
wakeOnWristRaiseStatus | lift waist on night off | lift waist on | N/A | N/A | duration* bit 4 | duration* bit 3 | duration* bit 2 | duration* bit 1 |
standReminderStatus | N/A | N/A | N/A | N/A | N/A | N/A | lunch break switch | stand reminder switch |

*\* "duration" is a number taking 4 bits, the screen will dim after "duration" seconds*

Here is an equivalent C++ style struct:
```c++
struct Cmd15Params {
  uint8_t switchByte0; //byte 1->callAlertSwitch, 2->alarmSwitch,3->lowPowerNotifyON?, 4->ifBinded? 5->ifOnRightHand 6->Wechat 7->QQ 8->sms
  uint8_t switchByte1; //byte 1->otherMessage 2->callAlertRelated? 8->motionDetect 
  uint8_t wakeOnWristRaiseStatus; //byte 1,2,3,4->duration(second), 7->liftWaistON 8->liftWaistOnNightOff
  uint8_t nightModeStartHour; //uint
  uint8_t nightModeEndHour; //uint
  uint8_t isStepGoalNoticeEnable; //boolean
  uint16_t stepGoalValueUnsigned; //uint
  uint8_t isHeartRateWarningNoticeEnable; //boolean
  uint8_t heartRateWarningValue; //uint
  uint8_t standReminderStatus; // 1->standReminder 2->LunchBreak(default1) otherBit->(default0)
  uint8_t standReminderStartHour; //uint
  uint8_t standReminderEndHour; //uint
  uint8_t lunchBreakStartHour; //uint
  uint8_t lunchBreakEndHour; //uint
  uint16_t sleepGoalValueUnsigned; //short uint minute
};
```

#### 0x16 SET_FEATURES_STATUS

Set a lot of features' status.

Params: same structure as 0x15's

Note that unless the corresponding switch is not set to '1', the notifications sent with protocol 0x18 will have no effect (a response from the device will still come though).

#### 0x17 SET_VIBRATION

TBC

#### 0x18 NOTIFICATION

Send notifications to the band.

Params: `[type:1][format:1][length:1][message:dynamic]`

explain:
+ type: message type
+ format: mark the character set
+ length: the length of the message, can be 0
+ message: the actual text encoded according to the `format`

type value | corresponding message type
:- |:-
0x00 | Wechat
0x01 | Call
0x02 | Clear/Remove call notification
0x03 | QQ
0x04 | SMS
0x05 | other
after 0x05 | fallback on 'other'(0x05)

format value | corresponding character set
:- |:-
0x00 | US-ASCII
0x01 | UTF-8


#### 0x19 SET_DIAL

Set which watch face to use. All watch faces are intergrated in the firmware.

Params: `[watch face number:1]`

The available watch faces is determined by the firmware version, and the information is only available through a web api which requires authentication. For example, firmware version above 2.0.0 has 4 different watch faces, and the serial numbers are 0,1,2, and 3.

However, it does no harm to try different numbers, as the device will cycle through the list.

#### 0x1A OTA

TBC

#### 0x1B RESET

Params: No param.

Note: The band will not send further response.

By sending this command, the band will perform a factory reset, and all sensitive data will be wiped.

#### 0x1C CLEAR_DATA

Params: No param.

Will wipe all records(step, sleep, heart rate, workout, etc.). **This processs is not reversable.**

#### 0x1D GET_KEEP_KEY_SEED

TBC

#### 0x1E SET_KEEP_KEY

TBC

#### 0x1F GET_FIRMWARE_EVENT

TBC

#### 0x20 SET_TIME_WITH_TIMEZONE

Params: `[timestamp:4][timezoneOffset:2]`

`timezoneOffset` is a signed short int stored in big-endian order. It is the minute offset to add to UTC to get local time. The official app's implementation is a 2-byte array.

#### 0x21 SET_SPORT_COEFFICIENT

TBC

#### 0x23 SET_GENERAL_STATUS

TBC

#### 0x24 GET_GENERAL_STATUS

TBC

#### 0x30 SET_WORKOUT_NOTICE

TBC

#### 0x40 GET_STEP_DATA

Get daily step record.

Params: `[timestamp:4]`

Response params: a list of bytes

The list's length is 1440, which means the band make record of the step count every minute.

#### 0x41 GET_LAST_WORKOUT_LOG

Fetch the last workout log. This is a little complicate.

Params: No param.

Response: `[type:1][data]`

`type` marks the workout type of this record. Each type 

There are 3 formats for `[data]`: common, swim, and motion

+ common: `[startTime:4][endTime:4][duration:4][distance:4][stepCount:4][calorie:4][remains:dynamic]`
  - format for each entry in `remains`: `[heartRate:1]`(basically, heart rate every 12secs)
+ swim: `[startTime:4][endTime:4][duration:4][turns:1][poolLength:1][calorie:4][turnsList:dynamic]`
  - format for each entry in `turnsList`: `[finishTime:4][strokes:1][type:1]`
+ motion: `[startTime:4][endTime:4][duration:4][calorie:4][count:2][remains:dynamic]`
  - - format for each entry in `remains`: TBC, perhaps same as "common"

known workout types:

ID  | Name  | Data format
:-: | :-    | :-
0   |AEROBIC      |Common
1   |STRENGTH     |Common
2   |RUN          |Common
3   |WALK         |Common
4   |SWIM         |Swim
5   |PUNCHEUR     |Common
6   |YOGA         |Common
7   |ROWING       |Common
8   |ELLIPTICAL   |Common
9   |AUTO_RUN     |Common
10  |AUTO_WORK    |Motion
11  |ROPE_SKIPPING|Common
12  |SQUAT        |Motion
13  |PUSH_UPS     |Motion
14  |CRUNCH       |Motion
15  |BURPEE       |Motion
16  |UNKNOWN      |N/A

known swim types:

ID | Name
:-:| :-
0  |UNKNOWN,
1  |BREASTSTROKE,
2  |FREESTYLE,
3  |BACKSTROKE,
4  |BUTTERFLY

#### 0x42 DELETE_LAST_WORKOUT_LOG

Send this command will delete the last workout log and shift to next one, so the next log can be read. The operation is not reversable.

Params: No param.

#### 0x45 GET_STEP_DATA_COMPRESSED

TBC

#### 0x51 GET_DAILY_CALORIES_DEBUG_INFO

TBC, the app use this protocol, but the device won't react. Perhaps this is for B2 only.

#### 0x52 GET_DAILY_CALORIES

TBC

#### 0x53 GET_DAILY_CALORIES_B2

TBC

#### 0x60 GET_SLEEP_DATA

Use this to fetch the sleep data.

Params: `[timestamp:4]`

Here, the timestamp is set to "today, 00:00:00, local time".

Response params: `[sleepStartTime:4][sleepEndTime:4][countOfSleepSegments:1][aListOfSleepSegments:dynamic]`

For each record in `[aListOfSleepSegments:dynamic]`, its format follows `[sleepType:1][duration:2]`

See sleep types in the table below. The unit of duration is minute.

Sleep types ID | Sleep type
:- |:- 
0x00 | ACTIVITY
0x01 | LIGHT_SLEEP
0x02 | DEEP_SLEEP
0x03 | WAKE
0x04 | FIX

#### 0x62 GET_WHOLE_DAY_SLEEP_DATA

TBC

#### 0x70 START_WORKOUT

It means "start realtime heart rate reporting".

See also [RECEIVE_HEART_RATE](#0x82-RECEIVE_HEART_RATE)

Params: No param.

#### 0x71 STOP_WORKOUT

It means "stop realtime heart rate reporting".

#### 0x72 RECEIVE_NOTIFY_STEP

TBC, possibly receive realtime step count report

#### 0x73 NOTIFY_STEP_SWITCH

TBC

#### 0x82 RECEIVE_HEART_RATE

All realtime heart rate report is sent with this protocol ID.

Response params: `[heartrate:1]`

#### 0x83 GET_HEART_RATE_DATA

Get one day's heart rate record.

Params: `[timestamp:4]`

Response params: a byte list of *today*'s heart rate record

The list length is 288, which means the device record the heart rate every 5 minutes.

#### 0x85 GET_LAST_HEART_RATE

Get the last heart rate value.

Params: No param.

Response params: `[heartrate:1]`

#### 0x86 GET_HEART_RATE_DATA_COMPRESSED

TBC

#### 0x93 START_TRACK_RECORDER

TBC

#### 0x94 STOP_TRACK_RECORDER

TBC

#### 0x95 RECEIVE_TRACK

TBC

#### 0xA3 CHECK_RESOURCE

TBC

#### 0xA4 PREPARE_RESOURCE

TBC

#### 0xA5 TRANSFER_RESOURCE

TBC

#### 0xB0 SET_NO_DISTURB

Set options for "do not disturb" feature.

Params: `[enable:1][startHour:1][startMinute:1][endHour:1][endMinute:1][noonEnable:1][noonStartHour:1][noonStartMinute:1][noonEndHour:1][noonEndMinute:1]`

#### 0xF0 SHUT_DOWN

TBC

#### 0xF1 GET_LOG_LONG

TBC

#### 0xF2 GET_LOG_DATA

TBC

#### 0xF3 DELETE_LOG_DATA

TBC

#### 0xF8 SET_ALGORITHM_ACQUISITION_TEMPLATE

TBC

#### 0xF9 FETCH_RAW_DATA_SUMMARY_LIST

TBC

#### 0xFE GET_MTU

Read remote MTU.

Params: No params.

Response params: `[MTU:1]`

#### 0xFF AUTHENTICATION

TBC

### Special Protocols

Special protocols are sent directly without standard packaging. I pick each packet's head as the protocol ID.

ID | Notes | Verified on B2
:-|:-|:-
0x03 | Received after apply feature settings(protocol 0x16)
0x04 | Received after binding(protocol 0x01)
0x22 | Get debug log  | ✅

#### 0x03

example: `03 30 88 01 00`

#### 0x04

example: `04 01 00 00`

#### 0x21

send: `21 11`

response (multiple packets): `21 10 XX [Data]`  
response end with: `21 10 00`

XX: cycle from 0x01 to 0xFF, mark the order of packets.  
`[Data]`: log string

## Notes & Comments

TBW

## Change Log

### 2021.3.10
+ initial version