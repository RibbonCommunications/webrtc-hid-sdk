# Backwards Compatibility

When used in a Citrix VDI eLux environment, the WebRTC HID SDK is backwards-compatible with the most recent and one (1) previous version of WebRTC HID Driver for VDI (DLL) (official releases only). This is intended to allow WebRTC HID Driver for VDI upgrades on the Thin Client installed-base to lag behind application updates.

## Supported WebRTC HID SDK and HID Driver for VDI Versions
Ribbon's standard support model provides full support for the current release and limited support for the previous release. This applies to both the HID SDK and Driver for VDI. When a new version of either SDK or Driver for VDI is released, it becomes the "current release", and so support ends for the N-2 release.

| Type   | Version | Release Date | Compatible With Type | Versions     | Comments |
|:------:|:-------:|:------------:|:--------------------:|:------------:|:--------:|
| SDK    | 2.4.0   | 2023-07-17   | Driver               | 1.4.1, 1.4.2 |          |
| Driver | 1.4.2   | 2023-06-16   | SDK                  | 2.3.0, 2.4.0 |          |
| SDK    | 2.3.0   | 2023-01-20   | Driver               | 1.4.1, 1.4.2 |          |
| Driver | 1.4.1   | 2021-04-14   | SDK                  | 2.3.0, 2.4.0 |          |

If a WebRTC HID SDK is being used with an older but still compatible version of WebRTC HID Driver for VDI, any product limitations imposed by the older version of driver remain in effect unless otherwise specified.