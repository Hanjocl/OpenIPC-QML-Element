# Installation Guide
This guide will walk you through the steps to install this into your current QML project, like using it as a library.

## Prepping the hardware
1. Download [Zadig](https://github.com/pbatard/libwdi/releases/download/v1.5.0/zadig-2.8.exe)
2. 'Repair' the libusb driver (you may need to enable [Options] -> [List All Devices] to show your adapter).

3. Install if not already installed [vcredist_x64.exe](https://aka.ms/vs/17/release/vc_redist.x64.exe)

## Installing this as a library
1. Add this repo to your current project (I recommend doing it as a git submodule)
2. Add the following lines to your main CMakeLists.txt

    ```cmake
    add_subdirectory(OpenIPC_QML)

    # After add_exectutable
    target_link_libraries(MyApp
        PRIVATE
            OpenIPC_QML
    )
    ```
3. add the following to lines to your main.cpp to make the qml modules available (see src/main.cpp for an example implementation)

    ```cpp
    #include "src/QmlNativeAPI.h"
    #include <player/QQuickRealTimePlayer.h>
    #include <QQuickWindow>

    int main() {
        // Before 'QQmlApplicationEngine engine' & 'QGuiApplication app(argc, argv)'
        QCoreApplication::setAttribute (Qt::AA_UseDesktopOpenGL);    
        QQuickWindow::setGraphicsApi(QSGRendererInterface::OpenGL);
        QQuickWindow::setSceneGraphBackend("opengl");

        // After 'QQmlApplicationEngine engine' & 'QGuiApplication app(argc, argv)'
        qmlRegisterType<QQuickRealTimePlayer>("realTimePlayer", 1, 0, "QQuickRealTimePlayer");
        auto &qmlNativeApi = QmlNativeAPI::Instance();
        engine.rootContext()->setContextProperty("NativeApi", &qmlNativeApi);

        // Rest of engine init & load

        // See main.cpp in src folder for example
    }
    ```

---
# How-to-use in QML
The following guide provides a list of plug-n-play modules for QML and go over the commands used in them. 
It just takes all the thing from the example qml and puts them in a list...

### The PLUG-N-PLAY solution
Use the following QML code modules to make it work!

#### 1. Selection box for codec
```qml
ComboBox {
    id: selectCodec
    width: parent.width
    model: ['AUTO','H264','H265']
    currentIndex: 0
    Component.onCompleted: {
        let codec = NativeApi.GetConfig()["config.codec"];
        if (codec&&codec !== '') {
            currentIndex = model.indexOf(codec);
        }
    }
}
```
#### 2. Selection box for channelWidth
```qml
ComboBox {
    id: selectBw
    width: parent.width
    model: [
        '20',
        '40',
        '80',
        '160',
        '80_80',
        '5',
        '10',
        'MAX'
    ]
    currentIndex: 0
    Component.onCompleted: {
        let chw = NativeApi.GetConfig()["config.channelWidth"];
        if (chw&&chw !== '') {
            currentIndex = Number(chw);
        }
    }
}
```

#### 3. Selection box for Channel
```qml
ComboBox {
    id: selectChannel
    width: parent.width
    model: [
        '1','2','3','4','5','6','7','8','9','10','11','12','13',
        '32','36','40','44','48','52','56','60','64','68','96','100','104','108','112','116','120',
        '124','128','132','136','140','144','149','153','157','161','169','173','177'
    ]
    currentIndex: 39
    Component.onCompleted: {
        let ch = NativeApi.GetConfig()["config.channel"];
        if(ch&&ch!==''){
            currentIndex = model.indexOf(ch);
        }
    }
}
```

#### 4. Selection box for dongle based on adresses 
```qml
ComboBox {
    id: selectDev
    width: 190
    model: ListModel {
        id: comboBoxModel
        Component.onCompleted: {
            var dongleList = NativeApi.GetDongleList();
            for (var i = 0; i < dongleList.length; i++) {
                comboBoxModel.append({text: dongleList[i]});
            }
            selectDev.currentIndex = 0; // Set default selection
        }
    }
    currentIndex: 0
}
```

#### 5. Key selector & file dialog
```qml
Column {
    FileDialog {
        id: fileDialog
        title: "Select key File"
        nameFilters: ["Key Files (*.key)"]

        onAccepted: {
            keySelector.text = file;
            keySelector.text = keySelector.text.replace('file:///','')
        }
    }
    Button {
        width: 190
        id:keySelector
        text: "gs.key"
        onClicked: fileDialog.open()
        Component.onCompleted: {
            let key = NativeApi.GetConfig()["config.key"];
            if (key && key !== '') {
                text = key;
            }
        }
    }
}
```

#### 6. Start button to start live stream
```qml
Rectangle {
    // Size of the background adapts to the text size plus some padding
    width: 180
    height: actionStartText.height + 10
    color: "#2fdcf3"
    radius: 10

    Text {
        id: actionStartText
        property bool started : false;
        x: 5
        anchors.centerIn: parent
        text: started?"STOP":"START"
        font.pixelSize: 32
        color: "#ffffff"
    }
    MouseArea{
        cursorShape: Qt.PointingHandCursor
        anchors.fill: parent
        Component.onCompleted: {
            NativeApi.onWifiStop.connect(()=>{
                actionStartText.started = false;
                player.stop();
            });
        }
        onClicked: function(){
            if(!actionStartText.started){
                actionStartText.started = NativeApi.Start(
                    selectDev.currentText,
                    Number(selectChannel.currentText),
                    Number(selectBw.currentIndex),
                    keySelector.text,
                    selectCodec.currentText
                );
            }else{
                NativeApi.Stop();
                player.stop();
                if(recordTimer.started){
                    recordTimer.clickEvent();
                }
            }
        }
    }
}
```

#### 7. QMl Live video feed display
```qml
QQuickRealTimePlayer {
        x: 0
        y: 0
        id: player
        width: parent.width - 200
        height:parent.height
        property var playingFile
        Component.onCompleted: {
            NativeApi.onRtpStream.connect((sdpFile)=>{
                playingFile = sdpFile;
                play(sdpFile)
            });
            onPlayStopped.connect(()=>{
                stop();
                play(playingFile)
            });
        }
        TipsBox{
            id:tips
            z:999
            tips:''
        }
        Rectangle {
            width: parent.width
            height:30
            anchors.bottom : parent.bottom
            color: Qt.rgba(0,0,0,0.3)
            border.color: "#55222222"
            border.width: 1
            Row{
                height:parent.height
                padding:5
                spacing:5
                Text {
                    anchors.verticalCenter: parent.verticalCenter
                    text: "0bps"
                    font.pixelSize: 12
                    width:60
                    horizontalAlignment: Text.Center
                    color: "#ffffff"
                    Component.onCompleted: {
                        player.onBitrate.connect((btr)=>{
                            if(btr>1000*1000){
                                text = Number(btr/1000/1000).toFixed(2) + 'Mbps';
                            }else if(btr>1000){
                                text = Number(btr/1000).toFixed(2) + 'Kbps';
                            }else{
                                text = btr+ 'bps';
                            }
                        });
                    }
                }


            }
            Row{
                anchors.right:parent.right
                height:parent.height
                padding:5
                spacing:5
                Rectangle {
                    height:20
                    width:30
                    radius:5
                    color: "#55222222"
                    border.color: "#88ffffff"
                    border.width: 1
                    Text {
                        horizontalAlignment: Text.Center
                        anchors.verticalCenter: parent.verticalCenter
                        anchors.horizontalCenter: parent.horizontalCenter
                        text: "JPG"
                        font.pixelSize: 12
                        color: "#ffffff"
                    }
                    MouseArea {
                        cursorShape: Qt.PointingHandCursor
                        anchors.fill: parent
                        onClicked:{
                            let f = player.captureJpeg();
                            if(f!==''){
                                tips.showPop('Saved '+f,3000);
                            }else{
                                tips.showPop('Capture failed! '+f,3000);
                            }
                        }
                    }
                }
                Rectangle {
                    height: 20
                    width: 50
                    radius: 5
                    color: "#55222222"
                    border.color: "#88ffffff"
                    border.width: 1
                    Text {
                        visible:!recordTimer.started
                        horizontalAlignment: Text.Center
                        anchors.verticalCenter: parent.verticalCenter
                        anchors.horizontalCenter: parent.horizontalCenter
                        text: "MP4"
                        font.pixelSize: 12
                        color: "#ffffff"
                    }
                    RecordTimer{
                        id:recordTimer
                        width:parent.width
                        height: parent.height
                        property bool started:false
                        function clickEvent() {
                            if(!recordTimer.started){
                                recordTimer.started = player.startRecord();
                                if(recordTimer.started){
                                    recordTimer.start();
                                }else{
                                    tips.showPop('Record failed! ',3000);
                                }
                            }else{
                                recordTimer.started = false;
                                let f = player.stopRecord();
                                if(f!==''){
                                    tips.showPop('Saved '+f,3000);
                                }else{
                                    tips.showPop('Record failed! ',3000);
                                }
                                recordTimer.stop();
                            }
                        }
                    }
                    MouseArea {
                        cursorShape: Qt.PointingHandCursor
                        anchors.fill: parent
                        onClicked:{
                            recordTimer.clickEvent();
                        }
                    }
                }
            }
        }
    }
```

#### 8. Shows the debug logs in a ListView
```qml
ListView {
    z:1
    anchors.top :logTitle.bottom
    anchors.fill: parent
    anchors.margins:5
    model: ListModel {}
    delegate: contactDelegate
    Component.onCompleted: {
        NativeApi.onLog.connect((level,msg)=>{
            model.append({"level": level, "msg": msg});
            positionViewAtIndex(count - 1, ListView.End)
        });
    }
}
```
---
### 1. NativeApi (`NativeApi` module)
This module is the control interface to all functionality. See it as the command centre. It supports the following commands:

#### Get a list of connect dongles at start-up of application.
    ```qml
    NativeApi.NativeApi.GetDongleList()
    ```
#### Does some magic to make the stream work (Haven't looked into it as it just works)
    ```qml
    NativeApi.onRtpStream.connect((sdpFile)=>{
                playingFile = sdpFile;
                play(sdpFile)
            });
            onPlayStopped.connect(()=>{
                stop();
                play(playingFile)
            });
    ```
#### Gets the currently selected channel config
    ```qml
    NativeApi.GetConfig()["config.channel"]
    ```
#### Gets the received Wifi Frames
    ```qml
    NativeApi.wifiFrameCount
    ```
#### Gets the total received RTP packages
    ```qml
    NativeApi.rtpPktCount
    ```
#### Shows debug messages
    ```qml
    Component.onCompleted: {
        NativeApi.onLog.connect((level,msg)=>{
            model.append({"level": level, "msg": msg});
            positionViewAtIndex(count - 1, ListView.End)
        });
    }
    ```




---
### 2. QQuickRealTimePlayer (`realTimePlayer` module)
This module is the actuall displays the live feed once it is available through the NativeApi.

```qml
import realTimePlayer 1.0

QQuickRealTimePlayer {
    id: player
    width: 800
    height: 600

    Component.onCompleted: {
        NativeApi.onRtpStream.connect((sdpFile) => {
            play(sdpFile)
        });
    }

    onPlayStopped: {
        play("last_used_file.sdp")
    }
}
```

---
## Throubleshooting
This is a guide for general problem I ran into when solving this mess.

### 1. Modules not installed

**Symptoms:**
- QML complains about modules not being installed on windows

**Solutions:**
- Ensure all DLLs are installed by using the following command in a termainal after build the project

  ```powershell
  C:\PATH_TO\windeployqt.exe --qmldir C:\PATH_TO_QML_FILES\qml C:\PATH_TO\generated_executable.exe
  ```