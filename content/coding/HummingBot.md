---
title: HummingBot
draft: false
tags:
  - wsl
  - quantitative
created: 2025-12-27 22:00
modified: 2025-12-27 22:00
---
Ref: [Documentation - Hummingbot](https://hummingbot.org/docs/)
## Install

### System Requirement

|**Component**|**Specifications**|
|---|---|
|**Operating System**|Linux x64 or ARM (Ubuntu 20.04+, Debian 10+), macOS, Windows (WSL2)|
|**Memory**|4 GB RAM per instance|
|**Storage**|5 GB HDD space per instance|
|**CPU**|at least 1 vCPU per instance / controller|

>[!info] Building from source
>If you're a developer looking to build custom strategies or exchange connectors, consider installing Hummingbot from source. There are instructions for macOS, Linux and Windows - see [Source Installation](https://hummingbot.org/client/installation/#source-installation).

### Before Install

**Miniconda**

```bash
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash ~/Miniconda3-latest-Linux-x86_64.sh
```

