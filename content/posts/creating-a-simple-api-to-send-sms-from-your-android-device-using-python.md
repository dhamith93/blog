---
title: "Creating a Simple Api to Send Sms From Your Android Device Using Python"
date: 2021-06-20T20:49:29+05:30
draft: false
---

## Background

What we are trying to do here is to create a simple python program to connect to the android device using ADB (Android Debug Bridge) and call the SMS service to send a SMS without doing any programming on the Android side. Obviously this is not going to replace a production level SMS service but it can be useful in smaller scale projects.

## Prerequisites

You should have python installed. And you should download the android platform tools, extract it, and add the path to the path environment variable before we start.

* Linux: https://dl.google.com/android/repository/platform-tools-latest-linux.zip
* macos: https://dl.google.com/android/repository/platform-tools-latest-darwin.zip
* Windows: https://dl.google.com/android/repository/platform-tools-latest-windows.zip

And you have to have an Android device with USB Debugging enabled. (Source: Enable developer options and USB debugging)

## Getting the right shell command

One of the tricky parts of this is to figure out the correct shell command to send to the device. This involves a tiny bit of digging into android source to figure out the right method. You can find below the commands to Android version 8 – 10 and 11. Version 12 is also similar to 11 at the moment. If you are using different versions, you can follow this guide on how to find the correct command: https://gist.github.com/Ademking/5351ed43a7c48575fe5e6de477d9781f

```
# Android version 8 - 10
command = 'service call isms 7 i32 0 s16 "com.android.mms.service" s16 "[receiver]" s16 "null" s16 "[msg]" s16 "null" s16 "null"'

#Android version 11+
command = 'service call isms 6 i32 0 s16 "com.android.mms.service" s16 "null" s16 "[receiver]" s16 "null" s16 "[msg]" s16 "null" s16 "null" s16 "null" s16 "null" s16 "null"'
```

## ADB handler

I’m going to create the Adb_Handler class first. Once it is done we can set it up to call it from either API  call or through another method.

```bash
mkdir python_sms && cd python_sms
vim Adb_handler.py
```

```python
# Adb_Handler.py
import sys
import subprocess
import re

class Adb_Handler:
    command = ‘’
    def adbExists(self):
        # check if adb command works

    def getDeviceList(self):
        # get the connected devices list

    def sendSms(self, deviceId, receiver, msg):
        # send the sms
```

```python
# Adb_Handler.py
def adbExists(self):
     if os.name == 'nt':
         cmd = subprocess.run(['where', 'adb'], stdout=subprocess.PIPE)
     else:
         cmd = subprocess.run(['which', 'adb'], stdout=subprocess.PIPE)
     result = cmd.stdout.decode('utf-8')
     return (len(result) > 0 and (result.splitlines()[0] != 'adb not found' or result.splitlines()[0] != 'INFO: Could not find files for the given pattern(s).'))
```

Before we move on to the SMS bits, we have to check if the adb command exists and it is working correctly. In this function, we are running either where (windows) or which (*nix) command depending on the operating system to check if the adb installed and added to the path.

```python
# Adb_Handler.py
def getDeviceList(self):
    if not self.adbExists(self):
        return []
    cmd = subprocess.run(['adb', 'devices'], stdout=subprocess.PIPE)
    result = cmd.stdout.decode('utf-8').splitlines()
    result = result[1:len(result) - 1]
    devices = []

    for device in result:
        devices.append(re.split(r'\t+', device.rstrip('\t'))[0])

    return devices
```

Once the adb check is done, this will connect adb and return  a list of connected devices. The output will come as a string, therefore the output will be cleaned up using regex after removing the first line. Sample output from the adb command:

```
$ adb devices
List of devices attached
R58M792AYHB device
```
```python
# Adb_Handler.py
def sendSms(self, deviceId, receiver, msg):
     command = self.command.replace('[receiver]', receiver).replace('[msg]', msg)
     cmd = subprocess.run(
         [
             'adb',
             '-s',
             deviceId,
             'shell',
             command
         ],
         stdout=subprocess.PIPE
     )
     return (cmd.stdout.decode('utf-8').splitlines()[0] == 'Result: Parcel(00000000    \'….\')')
```

First we should replace the command class variable in the Adb_Handler class with the command given in the “Getting the right shell command” section. Note that the ‘[receiver]’ and ‘[msg]’ should not be changed manually. ex.

```python
# Adb_Handler.py
command = ‘service call isms 6 i32 0 s16 "com.android.mms.service" s16 "null" s16 "[receiver]" s16 "null" s16 "[msg]" s16 "null" s16 "null" s16 "null" s16 "null" s16 "null"’
```

Then let’s look into the sendSms function. Here the function gets the receiver phone number and the message. It will replace those placeholders with the real data in the command variable and execute it. Note that, here we can only check if the command successfully sent to the device, not if it actually sent the message because we cannot get such an output from ADB.

We have completed the work on the ADB side of the program. Let’s test this before moving on to the API part. At this point you should connect your USB debugging enabled device. Once that’s done, let’s write a simple main function to test this. Note this will cost you money depending on your mobile plan.

```python
# main.py
from Adb_Handler import Adb_Handler as adbHandler
 def main():
     devices = adbHandler.getDeviceList(adbHandler)
     if len(devices) > 0:
         adbHandler.sendSms(adbHandler, devices[0], 'your_number_here', 'Hello, World!')
 main()
```

Once you run this, if everything is correctly set, you should receive a text from yourself. If you didn’t receive it, please check this by running the ADB command from a terminal directly.

## API

Let’s move on to creating the API.

```python
# Server.py
import cgi
import urllib
import sys
import threading
import re
from http.server import BaseHTTPRequestHandler, HTTPServer
from Adb_Handler import Adb_Handler as AdbHandler

class Server(BaseHTTPRequestHandler):
    key = ''
    deviceId = ''
    adbHandler = AdbHandler

    def start(self, ip, port, key, deviceId):
        # start the server

    def stop(self):
        # stop the server

    def do_POST(self):
        # handle post requests
```

That is the basic structure of the Server class.

```python
# Server.py
def start(self, host, port, key, deviceId):
     self.key = key
     self.deviceId = deviceId
     print('Starting server…')
     server_address = (host, port)
     self.httpd = HTTPServer(server_address, Server)
     # if using https
     # httpd.socket = ssl.wrap_socket(httpd.socket, server_side=True, certfile='', keyfile='', ssl_version=ssl.PROTOCOL_TLS)
     thread = threading.Thread(target = self.httpd.serve_forever)
     thread.daemon = True
     thread.start()
     print('Server started on: ' + host + ':' + str(port))
```

We are sending the ip, port, key, and the ADB deviceId to the start function to start the server listening on the given port. You can use the commented out bits to set up HTTPS. It will require a cert file and a key file. I’m not going to set it up to keep things simple.

```python
# Server.py
def stop(self):
     self.httpd.shutdown()
     self.httpd.server_close()
     print('Server stopped')
```

This is used to stop the server.

```python
# Server.py
def do_POST(self):
 self.send_response(200)
 self.send_header('Content-type', 'application/json')
 self.end_headers()
 ctype, pdict = cgi.parse_header(self.headers['content-type'])
 if ctype == 'application/x-www-form-urlencoded':
     length = int(self.headers['content-length'])
     postVars = urllib.parse.parse_qs(self.rfile.read(length))
 else:
     postVars = {}
 message = ''
 if bytes('key', 'utf8') in postVars:
     try:
         key = postVars[bytes('key', 'utf8')][0].decode('utf-8')
         msg = postVars[bytes('msg', 'utf8')][0].decode('utf-8')
         rec = postVars[bytes('rec', 'utf8')][0].decode('utf-8')
     except:
         message = '{ "status":"error_decoding_params" }'
 else:
     message = '{ "status":"no_auth" }'
 if len(message) > 0:
     self.wfile.write(bytes(message, 'utf8'))
     return
 if key == self.key:
     if len(msg) == 0:
         message = '{ "status":"EMPTY_MESSAGE" }'
     elif len(msg) > 160:
         message = '{ "status":"MESSAGE_EXCEEDS_160_CHAR_LIMIT" }'
     else:
         if (self.adbHandler.sendSms(AdbHandler, self.deviceId, rec, msg)):
             message = '{ "status":"REQUEST_PROCESSED" }'
         else:
             message = '{ "status":"ERROR_PROCESSING_REQUEST" }'
 else:
     message = '{ "status":"WRONG_AUTH" }'
 self.wfile.write(bytes(message, 'utf8'))
```

This is the function which handles the post request coming to the server. We are expecting a x-www-form-urlencoded request with below parameters.

```
key: this is to verify the incoming request
msg: text message
rec: phone number of the receiver
```

We are validating those inputs and send errors if we found any issues. One of the important checks here is the message length. We are currently setting it as 160 as it is the standard message length. If input validation is done without any issues, we are calling the sendSms function from the Adb_Handler class.

That is it for the Server class. This is a very basic implementation. You can improve it as per your requirements or you can use an existing framework to set this up. End of the day what needs to be done is sending the validated input to Adb_Handler.sendSms() function.

Let’s modify the main function as below to start the server.

```python
# main.py
from Adb_Handler import Adb_Handler as adbHandler
from Server import Server as server
def main():
    devices = adbHandler.getDeviceList(adbHandler)
    if len(devices) > 0:
        server.start(server, host='0.0.0.0', port=5000, key='SHARED_KEY_HERE', deviceId=devices[0])
        print('Server started. enter q to exit')
        while 1:
            if input('\n> ').lower().startswith('q'):
                server.stop(server)
                break
if __name__ == '__main__':
    main()
```

Here we are starting the server with the host, port, and the key shared with the clients. Also we added a way to let users gracefully stop the server by entering `q` as an input without force stopping with ctrl+c.

Now the server is up and running, we can check the API using curl or postman. Here I’m using curl.

```bash
curl --location --request POST 'http://localhost:5000' \
 --header 'Content-Type: application/x-www-form-urlencoded' \
 --data-urlencode 'key=SHARED_KEY_HERE' \
 --data-urlencode 'msg=Hello, World!' \
 --data-urlencode 'rec=0123456789'
```

If all goes okay, it will send you the message below.

![](/python_sms_end.png)

Like I said, this can be improved further and the Adb_handler class can be used with GUI tools or any other command line programs to send SMS (IOT with Raspberry Pi comes to mind). Let me know you find any issues with the code or do you have any improvements.

You can find the full code for the project here: https://github.com/dhamith93/python-sms


