---
layout: post
title:  "\"Arduino.h\" file not found. PlatformIO (Clion)"
date:   2023-01-19 21:52:00 +1100
author: harvey
tags: [technology, arduino, solutions]
categories: Solution 
---

# Possible Solutions

## Add Framework
In you `platformio.ini` file add the `framework` definition of `arduino`.
For example: If you were using an ArduinoUNO. 
```yaml
[env:uno]
platform = atmelavr
board = uno
framework = arduino
```

## Re-run PlatformIO initialisation. 
In Clion go to `Tools`&rarr;`PlatformIO`&rarr;`Re-Init`

## Add New CMake Profile
Go to `Preferences`&rarr;`Build,Execution,Deployment`&rarr;`+`
This should add a new profile with the name of your board as defined in your 
`platformio.ini` file.

Once a new profile is created. Go to the top right drop-down in the main screen
of Clion and click on your new MCU profile. 

![Dropdown-Menu](/assets/img/arduino-h.png)

In the above image either an ESP32Feather or an ArduinoUNO can be selected.
