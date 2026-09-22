---
title: "systats v0.4.1: Container-Aware Go Module to Scrape Linux Stats"
date: 2026-09-22T10:23:36+05:30
tags: [go, linux, containers, kubernetes, observability]
keywords: [go, linux, containers, kubernetes, observability]
author: "Dhamith Hewamullage"
authorTwitter: "" #do not include @
showFullContent: false
readingTime: true
hideComments: false
---

When I [first wrote about systats](https://blog.dhamith.me/posts/systats-go-module-to-collect-linux-system-metrics/)
in 2022, it was the metrics code I'd pulled out of [SyMon](https://github.com/dhamith93/SyMon):
a list of things it could read from `/proc`. Since then it's grown in a more
specific direction: stats for Go services that need to report on themselves
and the box they run on, whether that's a health endpoint, a node agent, or
an edge device.

This post covers what's changed between v0.2.0 and v0.4.1. The biggest change
is that it now knows when it's running in a container.

## Container-aware stats

By default, `/proc` describes the host, not your container. Without this, a
pod capped at 512MB reports the node's full RAM as its total. Set one field
and `GetMemory`, `GetCPU` and `GetPressure` read the calling process's own
cgroup (v1 and v2, auto-detected) instead:

```go
syStats := systats.New()
syStats.ContainerAware = true

mem, _ := syStats.GetMemory(systats.Megabyte)
// mem.Limited == true, mem.Total == 512

cpu, _ := syStats.GetCPU()
// cpu.AllocatedCores == 0.5 for a Kubernetes "500m" limit
// cpu.LoadAvg is a % of that allocation, not of every host core
```

It's off by default, and if there's no cgroup limit it falls back to host
numbers. `Limited` tells you which you got. It also works for plain systemd
units with `MemoryMax=` or `CPUQuota=`, not only containers. `Load1`/`Load5`/`Load15`
stay host-wide, since there's no cgroup equivalent.

## Pressure stall information

CPU percentage and load average can't tell a busy machine from a thrashing
one. PSI can. `/proc/pressure` reports how much time tasks spent *waiting*
on CPU, memory or I/O:

```go
p, _ := syStats.GetPressure()
if p.Available && p.Memory.Some.Avg60 > 10 {
    // over 10% of the last minute was spent stalled on memory
}
```

`Some` is the early warning (at least one task stalled); `Full` means nothing
ran at all, which is what users feel as slowness. With `ContainerAware` on
cgroup v2, you get your own cgroup's pressure (v1 has no PSI, so it falls
back host-wide). On kernels older than 4.20, `Available` is false rather
than handing you zeros that look real.

## No more shelling out for stats

`ps`, `df`, `ip`, `lsof` and `whereis` are gone, replaced by reads from
`/proc` and `/sys`, `net.Interfaces()` and `unix.Statfs`. Stats collection
now works on distroless and scratch images. Two calls still use external
tools: `IsServiceRunning` (`systemctl`) and the logged-in users list in
`GetSystem` (`who`). Both now run under `LC_ALL=C` with a 5s timeout.

That second one is what v0.4.1 fixes. On an image without `who`, the
"executable file not found" error message was being parsed as a login
line, producing a user named `exec:` logged in from host `not`. It now
returns an empty list.

## New stats

- **Disk I/O**: per-device counters from `/proc/diskstats`, with
  `RatesSince` to turn two samples into throughput, IOPS and iostat-style
  `%util`.
- **Temperatures**: per-sensor hwmon readings, useful on Raspberry Pis and
  other edge boxes. VMs just return an empty slice.
- **TCP connection states**: counts by state across IPv4 and IPv6, for
  spotting `TIME_WAIT` buildup.
- **Protocol counters**: `/proc/net/snmp` and `netstat` merged, with an `ok`
  that separates "this kernel doesn't have it" from "it's zero".
- **Single-process lookup**: `GetProcess(pid)` returns name, state, threads,
  open FDs and I/O for one process, e.g. your own via `os.Getpid()`.
- **Better CPU topology**: `PhysicalCores` and `Sockets`, correct on
  multi-socket and hybrid P+E-core machines, plus traditional load averages.

## Friendlier to real services

- **Contexts**: the seven methods that sample, shell out or touch the
  network have `WithContext` variants. `IsServiceRunningWithContext` returns
  an error too, so "stopped" and "couldn't check" are finally different.
- **Concurrency**: a configured `SyStats` is safe to share across
  goroutines, now documented and covered by a race-detector test.
- **Injectable paths**: every path, from `ProcPath` to `SysClassNetPath`, is
  a struct field, so tests can point at fixture trees and still run in
  parallel.
- **Tunable CPU sampling**: `CPUSampleWindow` defaults to 300ms; shorten it
  if your health check can't wait that long.
- **Typed parameters**: `Unit`, `SortOrder` and `CPUMode` are real types, so
  a typo like `"memroy"` is a compile error instead of a silent fallback.

## See it

`systats` now includes an `example/` that renders everything as a single
offline HTML dashboard, with a banner when a cgroup limit is detected.
From a clone of the repo:

```bash
go run ./example -serve :8080
```

[systats example dashboard - bare-metal server](/reports/dashboard-bare-metal.html)

[systats example dashboard - VM](/reports/dashboard-vm.html)

[systats example dashboard - Docker container](/reports/dashboard-docker.html)

## Upgrading from v0.2.0

There are a few breaking changes:

- JSON output uses lowerCamelCase field names (`"rxBytes"`, not `"RxBytes"`).
- `CPU.NoOfCores` now means logical CPUs; use `PhysicalCores` for the old
  meaning.
- `Memory`, `Swap` and `DiskUsage` sizes are `float64`, and `Megabyte`/`Kilobyte`
  values have changed because the old conversions were wrong.
- `Unit`, `SortOrder` and `CPUMode` are defined types. Untyped literals still
  work, but a `string` variable needs `systats.Unit(...)`.
- `Disk.Convert` returns an error for unknown units.
- `GetTopProcesses` rejects unknown sort orders; use `SortByCPU` / `SortByMemory`.
- Go 1.18 or newer is required.

## When not to use it

systats is deliberately narrow. It's Linux-only and reports on its *own*
cgroup. If you need macOS/Windows or want to enumerate other containers,
use [gopsutil](https://github.com/shirou/gopsutil). If you want every raw
`/proc` field, use [prometheus/procfs](https://github.com/prometheus/procfs).

```bash
go get github.com/dhamith93/systats@v0.4.1
```

The full [CHANGELOG](https://github.com/dhamith93/systats/blob/main/CHANGELOG.md)
has the details. Issues and PRs welcome: https://github.com/dhamith93/systats