How to install:
---
Install nessacery libraries:

- [SDL2](https://www.libsdl.org/)
- [FFmpeg](https://ffmpeg.org/)
- [LIBUSB](https://github.com/libusb/libusb)
- [LibSodium](https://doc.libsodium.org/)
- [QML](https://doc.qt.io/QMLLive/qmllive-installation.html) (Only needed if you want the example application)




---
# WiFi Broadcast FPV client for QML

This is a fork of fpv4win (an app for Windows that packages multiple components together to decode an H264/H265 video feed broadcasted by wfb-ng over the air)

Basically I just made the documentation and updated parts of it so it works with QT6. The main functionality credits all go the the contributer of fpv4win!

# See the How-to-Guide on how to implement this project
[Setup Guide](READ_HOW_TO_USE.md)

---
### Dependencies:
- [devourer](https://github.com/openipc/devourer): A userspace rtl8812au driver initially created by [buldo](https://github.com/buldo) and converted to C by [josephnef](https://github.com/josephnef) .
- [wfb-ng](https://github.com/svpcom/wfb-ng): A library that allows broadcasting the video feed over the air.
- [rtl8812au-monitor-pcap] (https://github.com/TalusL/rtl8812au-monitor-pcap.git): Hopefully only used on Windows to create a .pcap file? 

Supported rtl8812au WiFi adapter only.

It is recommended to use with [OpenIPC](https://github.com/OpenIPC) FPV

### Before using this in QML 
- 1. Download [Zadig](https://github.com/pbatard/libwdi/releases/download/v1.5.0/zadig-2.8.exe)
- 2. Repair the libusb driver (you may need to enable [Options] -> [List All Devices] to show your adapter).
- 3. Install [vcredist_x64.exe](https://aka.ms/vs/17/release/vc_redist.x64.exe)

