# MOS intel-gpu-top

mos-intel-gpu-top provides a **MOS plugin** that integrates the `intel-gpu-top`
utility into the MOS ecosystem.

---

## Overview

This repository contains the **MOS plugin implementation**, optional helper
functions, configuration files (such as `settings.json`), and a packaged
`intel-gpu-top` binary required to expose Intel GPU monitoring functionality
within MOS.

The plugin allows MOS to provide GPU usage and performance information on
systems with supported Intel GPUs.

---

## Build & Automation

This repository includes a **GitHub Actions workflow** used to build and package
the plugin and its associated binaries for MOS.

The build process is fully automated and produces artifacts that can be
installed through the MOS Hub.

---

## Licensing

The contents of this repository (plugin code, build scripts, configuration,
and automation) are licensed under **GPL-3.0**.

`intel-gpu-top` itself is licensed under its respective upstream license.

---

## Third-Party Software

This repository builds and packages third-party open-source software.
Packaged components remain licensed under their original upstream licenses.

Refer to `THIRD_PARTY.md` for details.
