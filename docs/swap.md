# Call Swap

## Definition and Precondition
A "call swap" is the act of toggling between an active call and a held call. Therefore, the precondition exists that before a call swap can be performed, your app must be engaged on an active call and have at least one call on hold.

## Requirements
Be aware of the limitations described in [driver compatibility documentation](./compatibility.md).

## Performing a call swap
Performing a call swap from your application simply involves putting the active call on hold (sending webrtc-hid a call hold request), and then immediately resuming a held call (sending webrtc-hid a call resume action). This will in essence cause webrtc-hid to perform a hookflash, causing the attached device to toggle between the active and held call.

It's important to note that in a WebRTC-only world, this has little meaning. You could perform a hold/resume in your app and never send the HID SDK anything, and the result would be the same (remember that attaching audio to a device is done elsewhere, not by this SDK). The idea that the managed device has a call on hold is meaningful only if physical telephone handsets or lines are connected to the HID device (as is possible with the Jabra Engage 65 and Jabra PRO 9450), in which case hookflash signals could be sent from the device to the telephony switch (i.e. PBX) that manages calls. In order to work in this configuration, webrtc-hid provides full call swap functionality.

Performing a call swap from a HID device involves the same user action as putting a call on hold or resuming a held call (typically a long-press on a single-buttoned device such as the Engage 50, or pressing the green Call button on a two-button device like the Jabra Speak 710). 

## Enabling Call Swap
The way that webrtc-hid knows to interpret the hold/resume user action as the user's desire to perform a call swap, as opposed to a normal hold or resume, is determined by whether the preconditions are met or not. In other words, if the app and device are engaged on a call and there's a call on hold, webrtc-hid will interpret the hold/resume physical action as the intention to swap calls, and send a 'call_swap' operation on the `HIDFunctionRequest()` event. If the preconditions are not met, the hold/resume physical action will simply be interepreted as hold or resume. **This means that when preconditions are met, an active call cannot be put on hold nor resumed from the device.**

It's important to note that before webrtc-hid can correctly distinguish between the user's desire to hold/resume or swap calls, your app must inform webrtc-hid when there are calls on hold, and when this condition clears. The HID SDK is aware of whether or not its managed device is actively on a call or on hold, but it cannot know if your app has other calls on hold "in the background". Therefore, your app must tell the HID SDK when this is the case - both when the condtion starts and ends.  This is done by sending a `'calls_on_hold'` message type on the `invokeHIDOperation()` API. It requires a boolean (true/false) parameter. Your app should call this API with `true` when calls are put on hold in your app, and `false` when there are no longer calls on hold in the app. This effectively enables and disables webrtc-hid's call swap functionality. Since webrtc-hid's internal 'calls_on_hold' value defaults to false, if it's never set to true by your app, webrtc-hid will never signal a call swap and will always signal hold/resume.

### Usage
```
invokeHIDFunction('calls_on_hold', <boolean>)
```

### Examples

#### Keeping webrtc-hid updated with held call information
One (very inefficient) way to keep the HID SDK updated with held call info would be to monitor it in your app like so (not recommended, example only):

```
let lastState = false

setInterval(() => {
  const areCallsOnHold = Boolean( findHeldCalls() )

  if (areCallsOnHold !== lastState) {
    lastState = areCallsOnHold

    invokeHIDFunction('calls_on_hold', areCallsOnHold)
  }
}, 1000)
```

A better way would be to do these checks at the point in your app where it starts, ends, holds or resumes calls.

One thing to watch out for is that 'calls_on_hold' is intended to reflect calls on hold outside of the current/active one, which can be trickier than it sounds. Case in point:
1. Your app makes or receives a call
2. The first call is put on hold
3. A 2nd call is started
4. The 2nd call ends
5. The user attempts to resume the held call from the device

At step 2, your app sends calls_on_hold = true, so after step 3, the user would have been able to swap between the held and active calls if they chose to.
At step 4, calls_on_hold is still true, but there's only 1 active call in the system, so it makes no sense for the HID SDK to signal a swap. In this case, at the point the 2nd call ended, you'd have to check to ensure there's only 1 call left in the system, and that it's on hold, and in this case cancel the calls_on_hold, so that at step 5 the user is able to resume the held call. Otherwise, the HID SDK will signal a call swap at step 5, which would be inappopriate. In other words, handling this correctly would have been to set calls_on_hold = true at step 3 and cleared it after step 4. **However**, setting calls_on_hold = false tells the HID SDK there are no calls on hold at all, so to get the device back into the correct state (so it can signal a call_resume) you'd need to resend 'call_hold'.

Sample code to handle this scenario:
```
function handleCallEnd (endingCall) {

    // if the call being ended is muted, unmute device first
    if (isCallMuted(endingCall)) {
        invokeHIDFunction('call_unmute')
    }

    const otherCalls = allCalls.filter(call => call.id !== endingCall.id)

    // if there are no other active calls, hangup the device
    if (otherCalls.length === 0) {
        invokeHIDFunction('call_end')

    // if there's one active call left and it's on hold,
    // send calls_on_hold false followed by call_hold
    } else if (otherCalls.length === 1 && isCallOnLocalHold(otherCalls[0])) {
        invokeHIDFunction('calls_on_hold', false)
        setTimeout(() => {
            invokeHIDFunction('call_hold')
        }, 1000)
    }
    // else... handle other conditions as determined by your app
}
```

#### Handling the emitted 'call_swap' event in the application
```
HID.on('HIDFunctionRequest', operation => {
  switch (operation):
    case 'call_swap': {
      const activeCall = findActiveCall()
      const heldCalls = findHeldCalls()

      if (activeCall && heldCalls.length === 1) {
        holdCall(activeCall)
        resumeCall(heldCalls[0])
      }

      break
    }
}
```