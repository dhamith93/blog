---
title: "Shortcut: Share Files Clipboard With Devices in Same Network"
date: 2021-11-19T21:07:43+05:30
draft: false
---


## Usage

Download: https://github.com/dhamith93/shortcut/releases

Run the `./shortcut` executable. A browser window will be opened with the URL to connect from other devices.

Currently supports Linux/macos/Windows.

Default port is `:5500`, but it can be changed by editing the config.json file. Make sure to add the semicolon before the port when changing.

Files can be dragged and dropped/uploaded from the browser and can be uploaded/downloaded by anyone visiting the URL from the same network from any device.

Device that runs the executable acts as a centralized server, and the files will be uploaded to the `public/files/` dir in the executable path. These files will be removed once the executable is stopped running or the next time it is running if the executable crashed/force closed.

Clipboard function can be used to share links/messages between connected devices real time.

## Available configuration through `config.json`

```
Port: string (:port),
MaxClipboardItemCount: int,
MaxFileSize: string (xMB),
MaxDeviceCount: int,
PreserveClipboardOnExit: bool
```

![](/shortcut_full.png)

## Compiling from source
Install Go https://golang.org/doc/install

Clone codebase https://github.com/dhamith93/shortcut

Run bellow command to build.

```bash
make clean && make build
```

Or run below command to run without building.

```bash
make run
```

