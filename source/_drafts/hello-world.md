---
title: Hello World
---
Welcome to [Hexo](https://hexo.io/)! This is your very first post. Check [documentation](https://hexo.io/docs/) for more info. If you get any problems when using Hexo, you can find the answer in [troubleshooting](https://hexo.io/docs/troubleshooting.html) or you can ask me on [GitHub](https://github.com/hexojs/hexo/issues).

## Quick Start

### Create a new post

``` bash
$ hexo new "My New Post"
```

More info: [Writing](https://hexo.io/docs/writing.html)

### Run server

``` bash
$ hexo server
```

More info: [Server](https://hexo.io/docs/server.html)

### Generate static files

``` bash
$ hexo generate
```

More info: [Generating](https://hexo.io/docs/generating.html)

### Deploy to remote sites

``` bash
$ hexo deploy
```

More info: [Deployment](https://hexo.io/docs/one-command-deployment.html)


{

    var isNightVisionInDrivingModeSettingsAvailable: Bool {
        if camera.local?.productSerie == WLProductSerie.saxhorn {
            return true
        }
        return camera.firmwareShort?.isNewerOrSameVersion(to: "1.8.0") == true
    }

    var isMarkSpaceSettingsAvailable: Bool {
        if camera.local?.productSerie == WLProductSerie.saxhorn {
            return true
        }
        return camera.firmwareShort?.isNewerOrSameVersion(to: "1.9.0") == true
    }

    var isAudioPromptSettingsAvailable: Bool {
        if camera.local?.productSerie == WLProductSerie.saxhorn {
            return true
        }
        return camera.firmwareShort?.isNewerOrSameVersion(to: "1.9.3") == true
    }

    var isNetworkDiagnosisAvailable: Bool {
        if camera.local?.productSerie == WLProductSerie.saxhorn {
            return true
        }
        return camera.firmwareShort?.isNewerOrSameVersion(to: "1.10") == true
    }

    var isAutoHDRAvailable: Bool {
        if camera.local?.productSerie == WLProductSerie.saxhorn {
            return false
        }
        return camera.firmwareShort?.isNewerOrSameVersion(to: "1.12.0") == true
    }

    var isAutoNightVisionAvailable: Bool {
        if camera.local?.productSerie == WLProductSerie.saxhorn {
            return false
        }
        return camera.firmwareShort?.isNewerOrSameVersion(to: "1.12.0") == true
    }

    var isMultiDrivingDetectionMethodsAvailable: Bool {
        if camera.local?.productSerie == WLProductSerie.saxhorn {
            return true
        }
        return camera.firmwareShort?.isNewerOrSameVersion(to: "1.12.0") == true
    }

    var isNotAutoEnterSleepAfterLoseConnectionAvailable: Bool {
        if camera.local?.productSerie == WLProductSerie.saxhorn {
            return true
        }
        return camera.firmwareShort?.isNewerOrSameVersion(to: "1.12.0") == true
    }

    var isGPSInfoInVideoAvailable: Bool {
        if camera.local?.productSerie == WLProductSerie.saxhorn {
            return false
        }
        return camera.firmwareShort?.isNewerOrSameVersion(to: "1.13.06") == true
    }

    var isMultiInstallationModesAvailable: Bool {
        if camera.local?.productSerie == WLProductSerie.saxhorn {
            return false
        }
        return ( camera.firmwareShort?.isNewerOrSameVersion(to: "1.13.06") == true) && (camera.local?.isSupportedUpsideDown == true)
    }

    var isParkSleepDelayAvailable: Bool {
        if camera.local?.productSerie == WLProductSerie.saxhorn {
            return camera.firmwareShort?.isNewerOrSameVersion(to: "1.2.01") == true
        }
        return camera.firmwareShort?.isNewerOrSameVersion(to: "1.14.0") == true
    }

    var isProtectionVoltageAvailable: Bool {
        if camera.local?.productSerie == WLProductSerie.saxhorn {
            return camera.firmwareShort?.isNewerOrSameVersion(to: "1.2.01") == true
        }
        return camera.firmwareShort?.isNewerOrSameVersion(to: "1.14.0") == true
    }

    var isAPNSettingsAvailable: Bool {
        if camera.local?.productSerie == WLProductSerie.saxhorn {
            return camera.firmwareShort?.isNewerOrSameVersion(to: "1.2.01") == true
        }
        return camera.firmwareShort?.isNewerOrSameVersion(to: "1.14.0") == true
    }

    var isSubStreamOnlyAvailable: Bool {
        if camera.local?.productSerie == WLProductSerie.saxhorn {
            return false
        }
        return camera.firmwareShort?.isNewerOrSameVersion(to: "1.14.0") == true
    }

    var isRiskDriveEventAvailable: Bool {
        if camera.local?.productSerie == WLProductSerie.saxhorn {
            return true
        }
        return (camera.firmwareShort?.isNewerOrSameVersion(to: "1.14.0") == true) && (camera.local?.isSupportedRiskDriveEvent == true)
    }

    var isWlanModeAvailable: Bool {
        if camera.local?.productSerie == WLProductSerie.saxhorn {
            return false
        }
        return (camera.firmwareShort?.isNewerOrSameVersion(to: "1.14.0") == true) && (camera.local?.isSupportedWlanMode == true)
    }

    var isVimMirrorAvailable: Bool {
        return camera.local?.productSerie == WLProductSerie.saxhorn
    }

    var isRecordConfigAvailable: Bool {
        return camera.local?.productSerie == WLProductSerie.saxhorn
    }

    var isPowerCordTestAvailable: Bool {
        if camera.local?.productSerie == WLProductSerie.saxhorn {
            return true
        }
        return camera.firmware?.isNewerOrSameVersion(to: "2.28.10.34.148") == true
    }

}
