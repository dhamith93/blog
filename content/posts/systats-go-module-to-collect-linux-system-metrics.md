---
title: "systats: Go Module to Collect Linux System Metrics"
date: 2022-03-30T21:20:52+05:30
draft: false
---
[systats](https://github.com/dhamith93/systats) is written as a part of my ongoing project to create a system monitoring and alerting system, [SyMon](https://github.com/dhamith93/SyMon). Initially, metrics collection was done as a part of SyMon codebase, but to make the maintainability and compatibility simple, it was taken out as a separate module, systats.

Currently tested on Ubuntu and CentOS.

Repo: https://github.com/dhamith93/systats

# Supported system metrics:
* Operating system
  * Distro
  * Hostname
  * Kernel
  * Current logged in users
  * Uptime
  * Last boot date/time
  * Timezone
* CPU
  * Model
  * Frequency
  * Load average 
  * Overall
    * Per core
  * No. of cores
* Memory/SWAP
  * Total
  * Available
  * Usage
* Disks
  * File system
  * Mount point
  * Total space
  * Usage
  * Inode info
* Network
  * Interfaces
  * IPs
  * Total usage
  * If port is open
  * Connectivity to external hosts
  * Count of established TCP connections
* Processes
  * Top n processes sorted by CPU/Memory usage
* Services
  * Returns if service is running

## Usage

```bash
# Import the module
go get -u github.com/dhamith93/systats
```

```go
import (
    "fmt"
    
    "github.com/dhamith93/systats"
)

func main() {
    // Initialize 
    syStats := systats.New()
    // Call the relevant methods to get the metrics
    system, err := systats.GetSystem()
    if err != nil {
        panic(err)
    }

    fmt.Println(system)
}
```

