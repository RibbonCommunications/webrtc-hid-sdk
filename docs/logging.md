# HID SDK Logs

The SDK produces logs that report device state changes, software actions and decisions, instructions emitted to and received from the app, etc.

Logs are not written to file or visible on the console by default, but can be enabled in various ways depending on where the logs are produced - in Electron's Main process or its Renderer.

## Electron Main

There are a few ways to view logs generated by this SDK in Electron's main process (primarily for use in a legacy VDI configuration).

1- Logs can be viewed in a command line/terminal environment such as Bash or Powershell. In that environment, the `DEBUG` environment variable must be set to 'webrtc-hid'. In Bash the command for this would be `export DEBUG=webrtc-hid`, in Powershell the command would be `$env:DEBUG=”webrtc-hid”`. Once the environment variable is set, then the app must be launched from that **same** terminal (e.g. `npm start` or `yourApp.exe`). This SDK's Main process logs will then be output to the terminal.

2- The [`setLogger` API](./README_v1.MD#setloggercustomlogger-optional) can be used to pass in a custom logger such as [electron-log](https://github.com/megahertz/electron-log). If `setLogger` is used, the previous method to view logs in terminal is disabled.

3- Main process logs can be sent to the Renderer for handling. In this case the `setLogger` API would be used to register a function that would send the log to the Renderer process via IPC. Note that `setLogger` expects an object (such as `console` or `electron-log`) that contains a `.log` method.

Example:

```javascript
// Electron main process
const { app, BrowserWindow } = require('electron')
const { setLogger } = require('@rbbn/webrtc-hid')

app.once('ready', () => {
    const mainWindow = new BrowserWindow(...)

    setLogger({
        log(msg) {
            if (mainWindow) mainWindow.webContents.send('HIDLog', msg)
        }
    })
    ...
})

app.on('window-all-closed', () => {
    mainWindow = null
    ...
})

// Electron renderer process
import { ipcRenderer } from 'electron'

ipcRenderer.on('HIDLog', (event, msg) => {
    handleLog(msg) // see handleLog implementation below
})
```

## Electron Renderer / Browser

How and where logs are viewed in the Renderer process is controlled via the `rendererLogs` parameter of the [initializeHIDDevices()](../README.MD#initializehiddevicesconfig) API.

The `rendererLogs` parameter can be either a Boolean (`true`/`false`) or a function. Its default value is `false`, meaning that logs from the renderer process are not visible anywhere. If set to `true`, logs will output at 'info' level to the renderer process console.

If the `rendererLogs` parameter is a function, then logs produced by the HID SDK will be passed to that function for handling. Given that the majority of the logs produced by this SDK are [redux](https://redux.js.org/) actions, the recommended way of handling them is as follows:

```javascript
let storedLogs = []
const maxLogs = 1000

// define a custom log handler
const handleLog = (msg, details = {}) => {
  // if this is a redux 'action' log, only log the payload
  const log = msg.trim() === 'action' ? details.payload : msg

  // at this point, 'log' will always be a simple string
  console.log(log)

  // optionally store the log for later retrieval/processing:
  if (storedLogs.length >= maxLogs) {
    storedLogs = storedLogs.slice(1) // delete storedLogs[0], which will be the oldest
  }
  storedLogs.push(log) // add new log to end of array
}

initOptions.rendererLogs = handleLog

const HID = initializeHIDDevices(initOptions)
```

## Downloading Stored Logs

If logs are stored per the previous example, the following methods may be used to download them to file.

### Electron

Log data from the Renderer process must first be sent to the Main process via IPC, and then a Main process IPC handler can write them to file using standard `fs`:

```javascript
  // Electron renderer process
  const filename = `${Date.now().toString()}_hid_logs.txt`

  // format the logs for consumption
  const formattedLogData = storedLogs.map(JSON.stringify).join('\n')

  ipcRenderer.send('saveHIDLogs', filename, formattedLogData)

  // Electron main process
  ipcMain.on('saveHIDLogs', (event, filename, formattedLogData) => {
    const downloads = app.getPath('downloads')
    const filePath = `${downloads}/${filename}`
    fs.writeFileSync(`${filePath}`, formattedLogData)
  })
```

### Browser

The recommended way to write the data to file in a browser is the same as what's used to download WebRTC JS SDK logs:

```javascript
  const filename = `${Date.now().toString()}_hid_logs.txt`

  // format the logs for consumption
  const formattedLogData = storedLogs.map(JSON.stringify).join('\n')

  const blob = new Blob([formattedLogData])

  // create a button that will save the Blob as a file when clicked
  const button = document.createElement('a')
  button.href = URL.createObjectURL(blob)
  button.download = filename
  button.click() // auto-click the button
```
