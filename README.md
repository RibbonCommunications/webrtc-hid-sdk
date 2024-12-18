# Ribbon's WebRTC HID SDK

The HID SDK enables application developers to handle HID device call operations by abstracting device functions.

It is currently supported for use in Chrome-based Browsers and Electron-based WebRTC applications running on Windows and Mac desktops (non-VDI), and in a Distant VDI environment. Note that use in a Distant VDI environment requires the [WebRTC VDI Toolkit](https://github.com/RibbonCommunications/webrtc-vdi-toolkit).<br>

The following Jabra headsets and configurations are currently supported, with support for other devices and vendors possible.<br>

| Device Make / Model | Desktop | VDI eLux | VDI Windows | VDI Mac |  Browser |   Firmware           |
| :-----------------: | :-----: | :------: | :---------: | :-----: | :------: | :------------------: |
|   Jabra Engage 65   | &#9745; | &#9745;  |   &#9745;   | &#9745; | &#9745;  | 2.0.5, 3.4.1, 5.11.0 |
|   Jabra Engage 50   | &#9745; | &#9745;  |   &#9745;   | &#9745; | &#9745;  | 1.24.0, 2.3.1        |
| Jabra Engage 50 II  | &#9745; | &#9745;  |   &#9745;   | &#9745; | &#9745;  | 3.7.2, 3.9.0         |
|   Jabra Speak 750   | &#9745; | &#9745;  |   &#9745;   | &#9745; | &#9745;  | 2.24.0               |
|   Jabra Evolve2 40  | &#9745; | &#9745;  |   &#9745;   | &#9745; | &#9745;  | 1.19.0               |

As of version 2.0.0, this SDK is to be used in Electron's Renderer process, rather than the Main process. It's necessary to `require` the SDK in Electron's Main Process to maintain backwards-compatibility for configurations where WebHID is not used. See Installation and `initializeHIDDevices` below. Refer to [the README for version 1.x](./docs/README_v1.MD) for instructions on using 1.x versions of the SDK in Electron's main process.

### Definitions
Legacy eLux/VDI is defined as an eLux VDI environment where the version of WebRTC Distant Driver for VDI is <1.6.0, or where the WebRTC HID Driver for VDI is used. Therefore, **non-legacy VDI** referenced below is any release of Mac or Windows VDI, and eLux VDI where the version of WebRTC Distant Driver for VDI is 1.6.0 or greater **and** the HID eLux Driver is not used.

## Installation

The SDK is shipped as a .tgz package that can be retrieved from NPM:

`npm install @rbbn/webrtc-hid`

### Electron Main Process
`require` the top-level library
```javascript
require('@rbbn/webrtc-hid')
```

### Electron Renderer Process / Browser
Import `initializeHIDDevices` from '@rbbn/webrtc-hid/local'.

```javascript
import { initializeHIDDevices } from '@rbbn/webrtc-hid/local'
```

## Logging

See [`Logging`](./docs/logging.md)

## API
All of the API's below are to be called from the Browser or Electron's Renderer process in the local app (as opposed to the remote) unless otherwise specified.

### `initializeHIDDevices(config)`

Initializes the SDK with configuration parameters.

```javascript
{
  mode: 'desktop'       // Valid values are 'desktop', 'browser', 'VDI' for Distant VDI and 'CitrixVDI'
  driverInfo: {}        // Object returned by 'vchannel.getInfo()' in the Main process. Required in a Distant VDI environment.
  rendererLogs: false   // See Logging
}
```

#### Initializing the local instance for Desktop or Browser
If your app is operating in a non-VDI environment, you can call `initializeHIDDevices` at any time during your app's initialization. In those cases, `mode` should be specified as `'desktop'` or `'browser'` as appropriate.

```javascript
const HID = initializeHIDDevices({ mode: 'desktop', rendererLogs: ... })
```

#### Initializing the local instance for Citrix VDI
If your app is operating in a Citrix VDI environment, you can call `initializeHIDDevices` at any time during your app's initialization, in which case `mode` should be `'CitrixVDI'`.

```javascript
const HID = initializeHIDDevices({ mode: 'CitrixVDI', rendererLogs: ... })
```

#### Initializing the local instance for Distant VDI

If your app is operating in a Distant VDI environment, initialization of the SDK **must be deferred until after the Distant VDI channel has been opened**. This is necessary because information about the remote client must be retrieved by calling vchannel.getInfo() and included in the configuration object passed in to `initializeHIDDevices` as `driverInfo`.

Example:

In your Electron Main process code, where the channel used by the main Distant VDI session is opened:

```javascript
const { ipcMain } = require('electron')
const vchannel = require('@rbbn/distant-vchannel')

let driverInfo

async function openChannel() {
  const channel = await vchannel.openVirtualChannel('RIBBON')
  driverInfo = await channel.getInfo()
}

ipcMain.on('getDriverInfo', event => {
  event.returnValue = driverInfo
})
```

In the Renderer process code, at the point where the local HID SDK will be initialized:

```javascript
const mode = 'VDI'

// retrieve driverInfo from the Main process
const driverInfo = ipcRenderer.sendSync('getDriverInfo')

const HID = initializeHIDDevices({ mode, driverInfo })
```

**Returns** an object that contains:

1. An event emitter that emits `HIDFunctionRequest` events. See [HIDFunctionRequest](#hidfunctionrequestoperation).
2. All APIs that follow.

### `setChannel(commsObject)` (non-legacy Distant VDI only)

All that's required to initialize the remote is to call `setChannel` with a valid communication object. This API **must be called on both the local and remote sides in Distant VDI**. The `commsObj` function parameter is an object that contains:

* a `send` function that will send a message over the Distant VDI channel to the far end
* a key called `receive` that has a value of `undefined`. The SDK will insert its receive function here (this is the same method used to initialize Ribbon's WebRTC.js SDK)

Examples:

#### Local
On the local side, the `setChannel` function is part of the object returned by `initializeHIDDevices`.

```javascript
const HID = initializeHIDDevices({
  mode: 'VDI',
  driverInfo: ipcRenderer.sendSync('getDriverInfo')
})

HID.setChannel({...})
```

#### Remote
On the remote side, `setChannel` is imported directly from the remote library:

```javascript
import { setChannel } from '@rbbn/webrtc-hid/remote'
```

Usage is the same in both local and remote:
```javascript
const HIDChannel = {
  send: message => {
    yourSendFunction('hid', message)
  },
  receive: undefined
}

setChannel(HIDChannel)

// handle incoming messages from the far end
yourReceiveFunction(messageType, ...data) {
  switch (messageType) {
    case 'hid':
      HIDChannel.receive(...data)
      break
    ...
  }
}
```

### `storeMainWindowID(id)` (legacy VDI only)
This SDK emits events in the Electron Renderer on the `HIDFunctionRequest` event. In order to do this, it needs the id of the Electron renderer window that should receive these messages.

Example:

```javascript
const electronRemote = require('electron').remote

HID.storeMainWindowID(
  electronRemote.getCurrentWindow().id
)
```

### `isSupportedDevice(label)`

Accepts a `string`, typically the label component of a deviceInformation object (see selectHIDDevice).

**Returns** a boolean indicating whether a device is supported for use or not.

Example:

```javascript
function selectMicrophone(deviceObject) {
  const result = HID.isSupportedDevice(deviceObject.label);

  if (!result) {
    console.log('unsupported device selected...')
  }

  HID.selectHIDDevice('microphone', deviceObject);
}
```

This API will return true for supported devices listed in the introductory section and for 'Jabra Evolve 80'. It will return false for any other string. The device label passed in is compared against the device names as they appear in the introduction. The label must **contain** one of these names in order for this API to return true - it does not have to match exactly.

Examples:
```
isSupportedDevice('G/N Audio Jabra Speak 750 Mono'); // true
isSupportedDevice('Jabra Speak 750'); // true
isSupportedDevice('Jabra Speak 75'); // false
```

It's **important** that `isSupportedDevice()` **not** be used to filter devices before selecting them via `selectHIDDevice()`. For proper operation the SDK should be informed when device selections change, whether a supported HID device is selected or not. If the app initially selects a HID device for ringing, then later selects a different device for ringing but doesn't inform this SDK, the HID device will continue to ring.

### `selectHIDDevice(deviceType, deviceInformation)`

Registers an association between a HID device (e.g. Jabra Engage 65) and a media device type (e.g. microphone).

Once the same device is selected for all 3 device types, a connection to the device will be established. That means, this API must be called by your app 3 times before a device can be used, once for each `'deviceType'`: 'microphone', 'speakers' and 'alert_speakers'.

The `'deviceInformation'` object will typically be a `'MediaDeviceInfo'` as returned by [`enumerateDevices`](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/enumerateDevices), but the only requirement is that it contain a `label` property. The `label` must match or contain one of the supported device model names before a device will be opened for use.

This API does not return anything, however calling it will cause the [deviceSelectionChange](#deviceselectionchange) event to be emitted. When the same device is selected for all device types, the [deviceStateChange](#devicestatechange) event will be emitted indicating the device has been opened. When the selections no longer match, it will be emitted again indicating the device has been closed.

### `allowHIDDeviceOpens(bool)`
Allows or disallows HID devices from being opened. True by default.

Devices should be closed before exiting an application; attempting to exit an app with an open device may cause issues. Depending on what your app does during shutdown, it may be necessary to prevent devices from being opened (after they've been closed) during app shutdown procedures.

Included for backwards-compatibility only. The SDK closes devices automatically during app exit.

### `invokeHIDFunction(operation)`

Sends an instruction to the SDK to cause a specific state change on a device.

Operation (`string`) may be one of:
* 'call_start': Indicates that an outgoing call has started Causes the device to go offhook
* 'call_accept': Indicates that an incoming ringing call has ben answered. Causes the device to stop ringing and go offhook
* 'call_reject': Indicates that an incoming ringing call has been rejected. Stops device alerting / ringing
* 'call_end': Indicates that an active call has ended. Returns the device to idle state
* 'call_mute': Mutes the HID device. On some devices this causes visual or audible alerts
* 'call_unmute': Unmutes the HID device
* 'start_ringing': Causes the HID device to perform its alerting function (ringing, blinking, etc.)
* 'stop_ringing': Causes the HID device to stop its alerting function. Note that sending any call state change operation will stop device alerting, so it's not necessary to call 'stop_ringing' when a call is answered or rejected
* 'call_hold': Causes the HID device to perform a call hold action. On some devices this causes audible alerts or visible state changes
* 'call_resume': Causes the HID device to exit its hold state
* 'offhook': Causes the HID device to go offhook
* 'onhook': Causes the HID device to go onhook
* 'reset': Resets the device to default state
* 'calls_on_hold': Informs the SDK when calls are put on hold in the app. See [call swap documentation](./docs/swap.md).

### `getVersion()`
**Returns** the current version of the SDK as a string.

## Events
### `HIDFunctionRequest(operation)`

When a user performs an action on a HID device, the SDK will interpret the action based on previous state and send this interpretation to the app. For example, if the user presses the mute button during an active call, 'call_mute' will be emited to the app.

Operation will be one of the following:
* 'call_start': The device has gone offhook indicating an attempt to originate a call (device was previously idle)
* 'call_accept': The device has gone offhook while it was in ringing state, indicating an attempt to answer the call
* 'call_reject': The device's call reject sequence was invoked, indicating an attempt to reject the ringing call
* 'call_end': The device has gone onhook, indicating an attempt to end the active call
* 'call_mute': The device's mute button was pushed, indicating an attempt to mute the active call
* 'call_unmute': The device's mute button was pushed while in the muted state, indicating an attempt to unmute the active call
* 'call_hold': The device's hold function sequence was invoked, indicating an attempt to put the active call on hold
* 'call_resume': The device's resume function sequence was invoked, indicating an attempt to resume/unhold the held call
* 'call_swap': The user has indicated the desire to swap between an active and a held call (1)
* 'device_error': The SDK has detected a previously connected device has been disconnected (power loss or physical disconnection)
* 'channel_error': (legacy VDI eLux only) The SDK has detected a loss of communication with the Thin Client

These messages are emitted in the Electron renderer window identified by `storeMainWindowID()`, or the renderer process that called `initializeHIDDevices()`.

The app must have an handler to receive these events. Device-initiated events have identifier `HIDFunctionRequest`.

```javascript
const HID = initializeHIDDevices()

HID.on('HIDFunctionRequest', operation => {
  switch (operation) {
    case 'call_start':
      // start a call in your app
      break;

    case 'call_mute':
      // mute an active call in your app
      break;
      ...
  }
})

window.addEventListener('beforeunload', () => {
  HID.off('HIDFunctionRequest')
})
```

**IMPORTANT** you'll notice that the list of operations emitted to the app as a result of a user actions on the device are a subset of the operations your app sends via `invokeHIDFunction()`. That is not coincidental. When this SDK notifies your app of a state change, it's important that you take whatever actions are necessary within your app (i.e. invoke your mute function), but also **replay the operation back to the SDK to update the device's state**<sup>1</sup>.

For example, let's say a user presses the mute button on a HID device during an active call. The device does **not** independently enter the mute state, and in fact doesn't change state at all. The only thing that happens when the mute button is pressed is that the device sends a signal to your PC. That signal is received by this SDK, interpreted (because signals have different meanings depending on previous app and device state) and then this SDK sends an interpretation of this signal to your app. So, upon receiving the mute signal from this SDK, your app should attempt to mute the active call and then, if successful, send a 'call_mute' instruction back to this SDK, which will then send a corresponding signal to the device instructing it to enter its mute state. If for whatever reason your app fails to mute the active call, it should not send this SDK the instruction to mute the device.

This may seem complicated, but it's absolutely necessary in order to keep your app, this SDK and the controlled device in sync. **Your app** is in charge of updating device state via this SDK. The device will not change state, and this SDK will not modify device state, unless instructed to do so by your app (other than in error scenarios).

<sup>1</sup> Do not replay operations 'device_error', 'channel_error' or 'call_swap' back to the SDK. In error scenarios this SDK will take appropriate actions  before alerting your app. For a device-initiated call swap, your app should send 'call_hold' followed by 'call_resume'. See [call swap documentation](./docs/swap.md).

### `deviceSelectionChange`
Each time that `selectHIDDevice()` is called, a `deviceSelectionChange` event will be emitted providing details on selected devices, for example:

```javascript
{
  selectedDevices: {
    microphone: { label: 'Default - Jabra Speak 710 (0b0e:2475)', supported: true },
    speakers: { label: 'Default - Jabra Speak 710 (0b0e:2475)', supported: true },
    alert_speakers: { label: 'Default - Jabra Speak 710 (0b0e:2475)', supported: true }
  },
  availableDevices: {...}
}
```

(Note that `availableDevices` may be empty until a device is opened for use.)

#### Error Parameter
Using a HID device in a browser requires that the user give the browser permission to access the device. When a device is selected for use, if the user has not previously granted access to the selected device, the browser will prompt the user for access. If access is granted, the device will be opened and a `deviceStateChange` event will be emitted. If the user does not grant access, the `deviceSelectionChange` event will be emitted with an `error` parameter, indicating that the device has been properly selected, but is not available for use. When the error parameter is included in the emitted data, the only way to render the device usable is to re-select the device and manually grant permission. Therefore, when the error parameter is present, you must tell the user what to do via your UI.

Similarly, the application **or** the user must grant access to HID devices before they can be used in an Electron desktop configuration. See [Granting Access to HID Devices in Electron Desktop](#granting-access-to-hid-devices-in-electron-desktop).

Example:
```javascript
HID.on('deviceSelectionChange', details => {
    if (details.error) {
        alert('No access to selected device! Re-select device and grant access when prompted');
    }
    console.log('HID: deviceSelectionChange: ', details);
});
```

### `deviceStateChange`
When selections for all device selections match for a given supported HID device, a connection to the device will be established, after which commands may be sent to the device and actions taken on the device will result in events being emitted into the parent appication. At this point the `deviceStateChange` event will be emitted indicating the device is 'open'.

If the device is disconnected or device selections change such that they no longer match, the connection to the device will be closed. At this point the `deviceStateChange` event will be emitted indicating the device is 'closed'.

## Use Cases

### Answering an incoming call
An incoming ringing call can be answered by:
* undocking the headset from the base (if present)
* pressing the MultiFunction button on the headset or base (if present)
* pressing the green Call Start/Answer button on the base or the Call button for single-button devices
* lowering the microphone boom (Evolve2 40 only, providing the device is not already on a call)

Any of these actions will cause 'call_accept' to be emitted. The device must be in a ringing state for an offhook action to be interpreted as 'call_accept'.

### Answering an incoming call while on another call

#### Jabra Engage 50 II
* pressing and holding the Call Start / End button on the Link controller for 1-2 seconds

For all other devices, the device's call hold/resume action should be performed. See Hold/Resume below.

### Originating an outgoing call

If the device goes offhook using any of the methods described above and is **not** in a ringing state, it will emit a 'call_start' operation on the HIDFunctionRequest event. It's then up to your app to take the appropriate action to start a call. If your app cannot successfully start a call in that case, it should send 'call_failure', followed by 'call_failure_finish' 1 second later to return the device to default state. Failure to do so may result in the device being left in the offhook state.

**NOTE that the Engage 65 base must be in "Soft Phone Mode" prior to going offhook**. This can be accomplished by pressing the MultiFunction button on the headset or the Call Answer button on the base for 1 second prior to removing the headset from the base. See the device's User Manual for more details.

```javascript
HID.on('HIDFunctionRequest', (operation) => {
  switch (operation) {
    case 'call_start':
      if (allConditionsMet)
        yourStartCallFunction();
      else {
        HID.invokeHIDFunction('call_failure')
        setTimeout(() => HID.invokeHIDFunction('call_failure_finish', 1000));
      }
      break;
  }
});
```

### Ending an active call
An active call can be ended by:
* replacing the headset on the base
* pressing the MultiFunction button
* pressing the red Call End button on the base or the Call button for single-button devices

### Muting / Unmuting
When on an active call, pressing the device's Mute button will cause 'call_mute' to be emitted. If the device is muted, 'call_unmute' will be emitted. 

On the Evolve2 40, the call can also be muted by raising the microphone boom and unmuted by lowering it.

### Hold / Resume
When the device is engaged in an active call, the call can be placed on hold or resumed by:

#### Jabra Engage 65
- pressing the green Call Answer button on the base
- pressing and holding the Multi-Function button on the headset for 1-2 seconds

#### Jabra Engage 50
- pressing and holding the Call Answer / End button on the Link controller for 1-2 seconds

#### Jabra Speak 750
- pressing the green Call Answer button on the base

#### Jabra Evolve2 40
- pressing and holding the Multi-Function button on the headset for 1-2 seconds

#### Jabra Engage 50 II
- pressing and holding the Multi-Function / Mute button on the Link controller for 1-2 seconds

### Call Reject

An incoming ringing call may be rejected on the device, regardless of whether the device is active on another call or not. Rejecting an incoming call can be accomplished by:

#### Jabra Engage 65
- pressing the red Call End button on the base
- double-clicking the Multi-Function button on the headset

#### Jabra Engage 50
- double-clicking the Call Answer / End button on the base

#### Jabra Speak 750
- pressing the red Call End button on the base

#### Jabra Evolve2 40
- double-clicking the Multi-Function button on the headset

#### Jabra Engage 50 II
- double-clicking the Call Start / End button on the Link controller

### Call Swap
- When preconditions are met, performing a 'call_hold' action on the device (see above) will signal the controlling application to swap between the active and a held call. See [call swap documentation](./docs/swap.md).

## Error Handling
Most errors - device connection, disconnection, power off, etc. - are handled gracefully and logged.

It's natural during app development that you may put the device into a state that is out of sync with your app. In that case, send it a 'reset' to return the device to default state.

### Device Error
If the active/selected device is powered off or disconnected from the Client `device_error` will be emitted on the `HIDFunctionRequest` event. In order to reuse the device, it must be reselected after being reconnected.

### Channel Error (legacy VDI only)
In legacy VDI eLux, if virtual channel communication between the HID SDK and the HID Driver running on the Thin Client is lost for any reason, `channel_error` will be emitted on the `HIDFunctionRequest` event. HID's channel will remain down and not automatically attempt to reconnect. Once your app chooses to reestablish communication, reissue `initializeHIDDevices()` and then reselect the device for all device types.

## Backwards Compatibility
When used in a Citrix eLux VDI environment, this SDK is backwards-compatible with the most recent and one (1) previous version of WebRTC HID Driver for VDI (DLL) (official releases only). This is intended to allow WebRTC HID Driver for VDI upgrades on the Thin Client installed-base to lag behind application updates.

See the [compatibility matrix](./docs/compatibility.md) for WebRTC HID Driver for VDI and WebRTC HID SDK version compatibility information.

**NOTE** As of version 2.3.0, this SDK will use WebHID by default in non-legacy eLux VDI. If it's necessary to have your application use the WebRTC HID Driver for VDI rather than WebHID, set the `useDriver` flag in `driverInfo` to true.

Example:

Make the following change to the sample code in `Initializing the local instance for VDI`:

```javascript
let driverInfo = ipcRenderer.sendSync('getDriverInfo')

  // set 'useDriver' to true to force use of the HID Driver/DLL
  driverInfo = {
      ...driverInfo,
      useDriver: true
  };

const HID = initializeHIDDevices({ mode, driverInfo })
```

## Electron Security
Use of Electron security features such as nodeIntegration, webSecurity and enableRemoteModule have no effect on this SDK's operation. This SDK includes a preload script you can include in your preload script for use with Electron's context isolation feature. See details [here](./docs/context-isolation.md).

## Granting Access to HID Devices in Electron Desktop

As of version 2.4.1, this SDK will use WebHID in desktop configuration. Changes are required to Electron main process code in your application to grant access to HID devices. See [WebHID Electron documentation](./docs/webHIDDesktop.md)

## Known Issues / Limitations
- The same device must be selected as active microphone, speakers and alert speakers via `selectHIDDevice()`.
- Going offhook on the Jabra Engage 65, Speak 750 or Evolve 2/40 may not cause 'call_start' to be sent up to the app due to non-deterministic behaviour of these devices in this scenario. Support tickets (272, 277) have been created with the device vendor.
- In VDI eLux when the WebRTC HID Driver for VDI is used, the Jabra Speak 750 is known to conflict with either the mouse or keyboard when offhook. The issue has been addressed by the vendor in the RP6 / 64-bit version of the eLux OS image. There are no plans to address it in the RP5 / 32-bit version.
- See limitations relating to use of older versions of WebRTC HID Driver for VDI in [compatibility documentation](./docs/compatibility.md).
- In Windows VDI, the LEDs on the Jabra Engage 50 may not be responsive if the device is not at factory default settings. If the Engage 50 LEDs are not changing state during call operations, first disconnect and reconnect the device. If the condition persists, reset the device using the latest available version of Jabra Direct. Note that this SDK always assumes devices are at factory default settings.
- In eLux VDI, it has been found that connecting a HID device to the Thin Client may cause eLux to select it as the system default device. As well, the previously selected device may be muted. Both of these state changes may affect in-progress and future calls until rectified. Resolution is to unmute and re-select devices in the eLux system menu. This is not related to this SDK but standard eLux behaviour.
- If a Jabra Engage 50 is using firmware version 2.3.1, it will not respond correctly to the 'stop_ringing' instruction; the headset will stop blinking, but the Link device will continue to blink indefinitely. The only known way to resolve the situation is to unplug and reconnect the device from the host PC. This issue affects all configurations when firmware 2.3.1 is used. Firmware version 1.24 works correctly and so is recommended in order to avoid this issue. The device vendor has indicated that this issue has been corrected in firmware version 2.10.0, now released.
- Issuing a call hold action from a HID device (e.g. a long-press on a single-buttoned device) may cause Apple Music on the client to start in a Mac environment. This is native MacOS behaviour and not related to this SDK.
- The Jabra Engage 50 II may be muted by the user while the call/device is on hold. However, when on hold, the device does not signal this state change to the SDK to interpret and forward to the controlling app. As a result, the device may end up in a muted state, out of sync with the parent app, once the call is resumed. Once in that state the device can be unmuted from the device.
- Use of Microsoft Teams or any other HID-capable application may interfere with this SDK's ability to reliably control HID devices. To ensure predictable behaviour and avoid undesirable side-effects, MS Teams should not be installed on the same PC where an application using this SDK is in use.

## CHANGELOG
See [CHANGELOG](./CHANGELOG.md).
