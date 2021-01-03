# WaylensCameraSDK
WaylensCameraSDK is a powerful SDK for Waylens Cameras.

* * *
## Requirements

* iOS 10.0+
* Xcode 12.1+
* Swift 5.3+

## Integration

### Step 1: Install dependent libraries.

[CocoaPods](https://cocoapods.org) is a dependency manager for Cocoa projects. For usage and installation instructions, visit their website. To integrate dependent libraries into your Xcode project using CocoaPods, specify it in your `Podfile`:

```ruby
pod 'CocoaAsyncSocket'
pod 'CocoaLumberjack/Swift'
```

### Step 2: Embed the framework.

1. Go to your application's Xcode project. 
2. With the project selected in the navigator area of the Xcode main window, click the File menu and choose Add Files to [project name].
3. Navigate to `WaylensCameraSDK` folder and select `WaylensCameraSDK.framework`, `WaylensFoundation.framework`.
4. Under `Destination`, select `Copy items if needed`.
5. Under `Add to targets`, select your project.
6. Click Add.
7. In the editor area of the main window, under the `General` tab, scroll down to `Frameworks, Libraries, and Embedded Content` and click the + (add) button.
8. Select `WaylensCameraSDK.framework`, `WaylensFoundation.framework`, and then click Add, make all these frameworks are `Embed & Sign`.
9. Under the `Build Settings` tab, find `Enable Bitcode` under `Build Options` section and set it to `No`.
10. In any of your Swift files that reference `WaylensCameraSDK` classes, simply add:

```swift
import WaylensCameraSDK
```

## Usage

### Common
Make sure to import the **WaylensCameraSDK** framework in files that use it.
```swift
import WaylensCameraSDK 
```

### Camera Connection Management

Activate camera discovery service, recommend that calling the `activate()` method just after the application did become active:
```swift
func applicationDidBecomeActive(_ application: UIApplication) {
    WLBonjourCameraListManager.shared.activate()
}
```

Deactivate camera discovery service, recommend that calling the `deactivate()` method when the application will resign active:
```swift
func applicationWillResignActive(_ application: UIApplication) {
    WLBonjourCameraListManager.shared.deactivate()
}
```

Listen to current connected camera change notification:
```swift
NotificationCenter.default.addObserver(yourObserver, selector: #selector(handleCurrentCameraChangeNotification), name: NSNotification.Name.WLCurrentCameraChange, object: nil)
```

Add delegate to handle camera manager's events:
```swift
WLBonjourCameraListManager.shared.add(delegate: yourDelegate)
```

### Camera Communication

Control Camera:
```swift
let camera = WLBonjourCameraListManager.shared.currentCamera
camera?.startRecord()
camera?.stopRecord()
camera?.formatTFCard()
camera?.liveMark()
/* ... */
```

Setup camera's settings delegate, for example, you use it in your view controller:
```swift
override func viewDidAppear(_ animated: Bool) {
    super.viewDidAppear(animated)
    WLBonjourCameraListManager.shared.currentCamera?.settingsDelegate = self
}

override func viewWillDisappear(_ animated: Bool) {
    super.viewWillDisappear(animated)
    
    if WLBonjourCameraListManager.shared.currentCamera?.settingsDelegate === self {
        WLBonjourCameraListManager.shared.currentCamera?.settingsDelegate = nil
    }
}
```

### Video Database Management

Get video clips in camera's video database:
```swift
let camera = WLBonjourCameraListManager.shared.currentCamera
let clipsAgent = camera?.clipsAgent
let markedClips = clipsAgent?.list(of: WLClipListType.bookMark)
let loopClips = clipsAgent?.list(of: WLClipListType.loop)
```

Respond to video clips change events:

```swift
let camera = WLBonjourCameraListManager.shared.currentCamera
camera?.clipsAgent.add(delegate: yourDelegate)
```

Get video clip's thumbnail and download URL:
```swift
let camera = WLBonjourCameraListManager.shared.currentCamera
let vdb = camera?.vdbClient
vdb?.requestDelegate = yourVdbRequestDelegate

vdb?.getThumbnailForClip(aClip.clipID, in: WLVDBDomain(rawValue: UInt32(aClip.clipType)), atTime: time, tag: tag, canBeIgnore: false, with: aClip.vdbID, useCache: true)
// You will receive this thumbnail in vdb's requestDelegate method `func onGet(_ thumbnail: WLVDBThumbnail!)`

vdb?.getDownloadURL(forClip: aClip.clipID, in: WLVDBDomain(rawValue: UInt32(aClip.clipType)), from: startTime, length: length, main: true, sub: true, subn: stream, tag: tag  
// You will receive this URL in vdb's requestDelegate method `func onGetDownloadURL(_ mainurl: String?, mainsize mainsizek: Double, sub subUrl: String?, subsize subsizek: Double, subn subnUrl: String?, subnsize subnsizek: Double, date mainDate: Date, length: UInt32, tag: UInt32)`
```

Get video clip's playback URL:
```swift
let camera = WLBonjourCameraListManager.shared.currentCamera
let vdb = camera?.vdbClient
vdb?.delegate = yourVdbDelegate

if aClip.isMP4(forStream: streamIndex) {
    vdb?.getMP4(forClip: aClip.clipID,
               in: WLVDBDomain(rawValue: UInt32(aClip.clipType)),
               from: aClip.startTime + offset,
               length: 30 * 60.0,
               stream: streamIndex,
               withTag: 1000,
               andID: aClip.vdbID)
} else {
    vdb?.getHLSForClip(aClip.clipID,
                     in: WLVDBDomain(rawValue: UInt32(aClip.clipType)),
                     from: aClip.startTime + offset,
                     length: 30 * 60.0,
                     stream: streamIndex,
                     withTag: 1000,
                     andID: aClip.vdbID)
}
// You will receive this URL in vdb's delegate method `func onGetPlayURL(_ url: String?, time: Double, tag: Int32)`
```

Remove a video clip from camera's video database:
```swift
let camera = WLBonjourCameraListManager.shared.currentCamera
let vdb = camera?.vdbClient
vdb?.removeClip(aClip.clipID, in: WLVDBDomain(rawValue: UInt32(aClip.clipType)), with: aClip.vdbID)
```

Mark a video clip in camera's video database:
```swift
let camera = WLBonjourCameraListManager.shared.currentCamera
let vdb = camera?.vdbClient
vdb?.markClip(aClip.clipID, in: WLVDBDomain(rawValue: UInt32(aClip.clipType)), from: fromTime, length: length, with: aClip.vdbID)
```

## Classes

### Camera Connection Management
WLBonjourCameraListManager

### Camera Communication
WLCameraDevice

### Video Database Management
WLCameraVDBClient
WLCameraVDBClipsAgent
WLVDBClip
WLVDBThumbnail