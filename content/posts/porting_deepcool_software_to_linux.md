+++
title = "Porting DeepCool communication to Linux to get AG620 CPU cooler stats display working"
date = "2024-08-26T17:31:15+05:30"
author = "Dhamith Hewamullage"
authorTwitter = "" #do not include @
cover = "/deepcool-ag620-digital.jpg"
tags = ["Linux", "Go"]
keywords = ["Linux", "CPU Coolers", "Go", "DeepCool", "AG620 Digital"]
description = "Porting DeepCool USB communication with AG620 Cooler to get its display working under Linux"
showFullContent = false
readingTime = true
hideComments = false
+++

Recently I moved away from Intel to AMD with `AMD Ryzen 7 5700X CPU`. With that I had to get a new CPU cooler as well. Between the coolers I looked into, I picked [DeepCool AG620 Digital](https://www.deepcool.com/products/Cooling/cpuaircoolers/AG620-DIGITAL-BK-Temperature-Display-Cooler-1700-AM5/2023/17664.shtml) because it has a little display which shows current CPU temp and usage as a percentage. It uses one of the USB 2.0 headers in the motherboard to communicate. 

![](/cooler_promo.png)

Unfortunately, I found out soon enough that the DeepCool's software only works on Windows. One of the reason I moved to the new CPU is to squeeze out few more VMs in Proxmox than my older Core i3 CPU. So the feature that I was excited about, will not work half of the time I use my PC. 

I anyway ran `lsusb` in Proxmox to check whether the OS can recognize the attached cooler. It did recognize the cooler. 

![](/lsusb_output.png)

Next step was to figure out how the DeepCool's software is communicating with the CPU cooler. For that I used Wireshark on Windows. I used the `vendor_id` from the `lsusb` output to isolate the communication between the host and the cooler using the filter `usb.idVendor == 0x3633`. With the output I figured out the `device_address` and put on another filter `usb.device_address == 3`. With that filter in place, it was visible that DeepCool's software was sending `URB_INTERRUPT` to the cooler with CPU temps and usage. Looking into the HID data being sent to the cooler and what values being displayed in the cooler, I was able to decode the values needs to be sent in order to display the values.

```
When displaying CPU temp as 54c:
10 13 00 05 04 00 00 00 ... 00 

When displaying CPU usage as 02%:
10 4c 00 00 02 00 00 00 ... 00
```

Second byte of the data contains the what type of value is being sent. `0x13` for temperature and `0x4c` for usage.

Third and the fourth bytes contains the first and second digits of the value being sent. `0x05` and `0x04` for the temperature `54c`. And `0x00` and `0x02` for CPU usage of `02%`. 

**Update:** fifth byte is used for turning on blinking for alerting as I found out during my `testing`.

With the information I was able to collect with my tiny experiment, I wrote a Go program to get the display on the cooler working under Linux. I checked it with Ubuntu 24.04 and Proxmox and the temperature and CPU usage values were displaying on the cooler just as on Windows. I set this up as a service and all was done. 

While I used many ported applications and drivers in the past, this was the first time I was able to work something out for myself. It would be better if DeepCool themselves can release a version for Linux, but this was a good little exercise into monitoring and implementing USB communication.

```go
package main

import (
	"fmt"
	"log"
	"os"
	"strconv"
	"strings"
	"time"

	"github.com/dhamith93/systats"
	"github.com/sstallion/go-hid"
)

func getCpuTemp() (int, error) {
	sensorData, err := os.ReadFile("/sys/class/hwmon/hwmon0/device/hwmon/hwmon0/temp1_input")
	if err != nil {
		return -1, err
	}
	sensorMilliC, err := strconv.Atoi(string(strings.Trim(string(sensorData), "\n")))
	if err != nil {
		return -1, err
	}
	return sensorMilliC / 1000, nil
}

func getLoadAvg() (int, error) {
	ss := systats.New()
	cpu, err := ss.GetCPU()
	if err != nil {
		return -1, err
	}
	return cpu.LoadAvg, nil
}

func numberToArray(num int) []int {
	strNum := strconv.Itoa(num)
	result := make([]int, len(strNum))
	for i, char := range strNum {
		result[i] = int(char - '0')
	}
	return result
}

func main() {
	var vendor_id, product_id, temp_mode, util_mode uint16 = 0x3633, 0x0008, 0x13, 0x4c
	b := make([]byte, 65)

	if err := hid.Init(); err != nil {
		log.Fatal(err)
	}

	d, err := hid.OpenFirst(vendor_id, product_id)
	if err != nil {
		log.Fatal(err)
	}
	defer hid.Exit()

	d.SetNonblock(true)
	var value int
	checking := "temp"
	for {
		if checking == "temp" {
			value, err = getCpuTemp()
			if err != nil {
				log.Fatal(err)
			}
			b[1] = byte(temp_mode)
			checking = "util"
		} else {
			value, err = getLoadAvg()
			if err != nil {
				log.Fatal(err)
			}
			b[1] = byte(util_mode)
			checking = "temp"
		}
		arr := numberToArray(value)

		switch len(arr) {
		case 1:
			b[3] = byte(arr[0])
		case 2:
			b[3] = byte(arr[0])
			b[4] = byte(arr[1])
		}

		// TODO: implement `alert` threshold and send b[5] = 0x1

		b[0] = 0x10
		_, err = d.Write(b)
		if err != nil {
			log.Fatal(err)
		}
		time.Sleep(time.Second * 5)
	}
}

```

