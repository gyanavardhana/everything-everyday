# Windows RAM - What we learned today:

## 1. RAM doesn't wear out like storage:
 - when we bought the laptop, we started with:
   - 8gb ram.
   - now the it is showing 7.2gb used.
   - our thought process on thing why did my ram reduce, even though laptop has the physical 8 gb ram card ?
   - so the answer for the above question is: the ram didn't change, the applications which are already there and new ones started using more in the background usage.
   - RAM capacity doesn't normally shrink because you've used the laptop for several years. Software requirements grow.
   

```text
2023-ish
8 GB RAM
│
├── Windows
├── Games
├── Chrome
└── Background apps
      ↓
   Enough RAM

2026
8 GB RAM
│
├── Windows 11
├── Defender
├── Chrome
├── HP software
├── OneDrive
├── Development tools
├── Background services
├── Drivers
└── Cache
      ↓
   Much less headroom
```

## 2. Installed RAM vs usable RAM
 - out laptop specification says we have 8.0 gb ddr4 ram.
 - but the task manager shows 7.8 gb usabe why ?
 - because some ram is hardware reservred.
 - our machine showed: hardware reserved is 254mb , so this is for firmware/hardware/graphics-related needs.

```text
8.0 GB physical RAM
       ↓
~254 MB hardware reserved
       ↓
~7.8 GB usable by Windows
```

## 3. What does Ram usage actually mean ?
 - we initially saw chrome/vs code handing and examined in task manger that 7.2/7.8 gb, which is a bit scary implying that there is 600mb of ram for us to use.
 - but what we learnt was that ram isn't simply divided into used vs empty.
 - windows has multiple memory categories:
   - Process memory -> memory used by applications/processes.
   - Nonpaged pool(957mb) -> kernel/driver memory that needs to remain in physical RAM.
    - gpt mentioned 957 was not ordinary.
    - kernel/driver mem can consume ram when TM's list doesn't explain it.

### 4. Paged pool vs nonpaged pool
 - we saw both:
   - paged pool: 437mb
   - nonpaged pool: 957mb

 - Paged pool -> kernel mem that can potentially moved to disk/pagefile when necessary.
 - NonPaged pool -> kernel mem that must remain in physical ram.
 - can concentrate more on non paged pool when debugging ram consumption.


### 5. Cached memory != wasted memory
 - we had cached: 1.9gb, after seeing this we thought why doesn't windows just free that 1.9gb ?
 - the answer for that is windows uses otherwise-useful ram as cache.

 ```text
You open a file
       ↓
Windows reads it from disk
       ↓
Keeps useful data in RAM
       ↓
Next access can be faster
```

If an application needs the RAM:

```text
Application needs RAM
       ↓
Windows reclaims cache
       ↓
Application gets RAM
```

### 6. RAMMap is useful when Task Manager isn't enough

We used **RAMMap** because Task Manager showed:

> ~7.2 GB used

but the process list didn't account for it.

RAMMap lets you inspect Windows memory at a deeper level.

We discovered categories like:

```text
Process Private
Metafile
Mapped File
Nonpaged Pool
Paged Pool
Shareable
Standby
```

This helped us realize:

> **A significant portion wasn't ordinary application memory.**

### 7. We discovered ReadyBoost-related allocations

- RAMMap showed large pool tags:

```text
smNp
smCB
smBt
```

- We then used:

```powershell
findstr /m /l /s smNp %SystemRoot%\System32\drivers\*.sys
```

- and similar commands, All three pointed to:

```text
rdyboost.sys
```

- What is `rdyboost.sys`?
  - It's the windows ReadyBoost driver.
  - we initially suspected this might be consuming more kernel mem, but we didn't disable it.
  - we checked for ReadyBosst.sfcache(none), checked for ReadyBoost registry config(none found).
  - we didn't find enough evidence do anything with this.


### 8. We found an HP Component
 - Task Manager showed:
  - `SysInfoCap.exe` -> 132mb 
  - remember -> when you see an unfamiliar process, identify its executable path and owning service and understand whether it's safe to remove or not.


We then investigated it with PowerShell:

```powershell
Get-Process SysInfoCap
```

and:

```powershell
Get-CimInstance Win32_Service
```

We discovered:

```text
HPSysInfoCap
HP System Info HSA Service
HP Inc.
```

So we traced:

```text
Process
 ↓
Executable
 ↓
Company
 ↓
Windows service
```

### 9. We found HP background services:

We ran:

```powershell
Get-Service |
Where-Object {
    $_.DisplayName -match 'HP|Hewlett|OMEN'
}
```

and found:

```text
HP Comm Recovery
HP App Helper
HP Diagnostics
HP Network
HP Omen
HP System Info
HP Insights Analytics
```

- This revealed that the laptop had accumulated quite a few **OEM background services**.
- Some were useful.
- Some were optional.
- remember -> Don't uninstall everything from a manufacturer just cause it says HP, as some provide:
    * hardware controls
    * Fn keys
    * diagnostics
    * thermal management
    * networking features
    * OMEN gaming functionality
- so we seperated and marked some as optiona/safe to delete:
    * Microsoft Clipchamp
    * Maps
    * Microsoft News
    * Microsoft To Do
    * Microsoft Whiteboard
    * Power Automate
    * Simple Solitaire
    * Weather
    * Feedback Hub
    * People
    * Xbox Live
    * Minecraft Education
    * Mail and Calendar
    * Microsoft Teams *(if unused)*
    * Outlook (new) *(if unused)*
    * OneNote *(if unused)*
    * Microsoft 365 Copilot *(if unused)*
    * HP One Agent
    * HP Documentation
    * HP QuickDrop
    * McAfee Personal Security
    


### 10. whole debugging process we went through:


```text
Problem
  │
  ↓
Task Manager
  │
  ├── Is Chrome responsible?
  │       ↓
  │      No
  │
  ↓
RAMMap
  │
  ├── Where is memory going?
  │
  ↓
Pool tags
  │
  ├── Which driver?
  │
  ↓
findstr
  │
  ├── rdyboost.sys
  │
  ↓
Investigate configuration
  │
  ├── No obvious ReadyBoost cache
  │
  ↓
Process investigation
  │
  ├── SysInfoCap.exe
  │
  ↓
Service investigation
  │
  ├── HPSysInfoCap
  ├── HP Insights
  └── other HP services
  │
  ↓
Cleanup
  │
  ↓
Restart
  │
  ↓
Measure again
  │
  ↓
4.7 GB / 7.8 GB
```
