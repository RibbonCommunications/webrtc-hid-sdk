# WebHID in Electron (non-VDI)

WebHID is available for use in Electron in versions 16+.

When WebHID is used in a Chrome-based browser, the user is presented a popup menu to select and grant permission to the Browser to access and use specific HID devices. Since this is not easily done in Electron, the maintainers of Electron have added controls to the Main process.

Note that what follows is a summary of Electron's documentation. For further details it should be referenced directly at `https://github.com/electron/electron/blob/<your-electron-major-version>-x-y/docs/api/session.md`

## Enabling access to HID Devices in Electron's Main Process

There are three ways to control access to HID devices in the Main process. All three are part of the `win.webContents.session` object, where `win` is your app's main BrowserWindow.

### setDevicePermissionHandler API

This API can be used on its own to grant/deny access to HID devices. It must return `true` for any HID device to be used. The primary benefit of this method is that no other code changes are required. Hence, **this is the recommended method**.

```
// return true for any Jabra devices based on their vendorId
win.webContents.session.setDevicePermissionHandler(details => details.device.vendorId === 0x0b0e)

// return true for any HID device
win.webContents.session.setDevicePermissionHandler(() => true)
```
### setPermissionCheckHandler API

This API grants or denies the ability for your app to make further permission requests (see Electron's documentation for a list of requests). Meaning that, **on its own, this API does not grant access to HID devices**, it only grants permission to your app to *request* access to HID devices. One of the other two methods described in this document must also be used.

If your app implements this API already, then it must return `true` when HID permissions are requested. If your app does **not** implement this API already, then **it's not necessary to add it**, as all permission requests are granted by default.

Example:
```
app.whenReady().then(() => {
  win = new BrowserWindow()

  win.webContents.session.setPermissionCheckHandler((webContents, permission, requestingOrigin, details) => {
    if (permission === 'hid') {
      // Add logic here to determine if permission should be given to allow HID selection
      return true
    }
    // Add logic here to allow/deny other permission requests
  })
```

## select-hid-device Event

Electron also provides a `select-hid-device` event that fires when a HID device is selected for use. The event provides details of the environment and device being requested, and allows a callback function to be called based on these factors. The callback is to be called with the device's deviceId in order to grant access to it. This method is an **alternative** to `setDevicePermissionHandler`.  

You may choose to implement a popup window that allows the user to grant/deny access to the device when this event fires.

**Note** that this event **only** fires when a device is selected through user action in a UI, it does not grant access to devices that are selected programmatically. For this reason, `setDevicePermissionHandler` is preferred.

Example, enabling access to Jabra devices:
```
  win.webContents.session.on('select-hid-device', (event, details, callback) => {
    event.preventDefault()
    const selectedDevice = details.deviceList.find(device => device.vendorId === 0x0b0e)
    callback(selectedDevice?.deviceId)
  })
```
