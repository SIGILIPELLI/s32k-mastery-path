# S32K Automotive Embedded Mastery Path

A free, structured, module-wise training program for automotive embedded
development on NXP's S32K microcontroller family — entry level to master
level, with real C code in every module and a hands-on project at the end of
each level. CAN bus, AUTOSAR, and ISO 26262 functional-safety context is
woven in throughout, because that context is what separates automotive
firmware from hobby embedded work. Hardware is optional: follow along at the
code/concepts level, use an S32K144EVB/S32K344 devkit, or explore the Renode
emulator.

**Live site:** https://sigilipelli.github.io/s32k-mastery-path/

## Contents

- **Level 1 · Entry** — the automotive embedded landscape & the S32K family, S32 Design Studio toolchain, clocks/power/boot, GPIO & pin muxing, UART, ADC & sensor inputs, CAN fundamentals, timers & PWM, safety & robustness basics, capstone (CAN sensor node)
- **Level 2 · Intermediate** — FlexCAN deep dive & filtering, eDMA, LPSPI/LPI2C, FreeRTOS on S32K, low-power modes, flash/FlexNVM & EEPROM emulation, S32K1→S32K3 migration, bootloader basics, UDS diagnostics, multi-node CAN project
- **Level 3 · Advanced** — AUTOSAR Classic (MCAL/BSW/RTE), Real-Time Drivers & MCAL config, CAN FD & DBC workflows, LIN & FlexIO, safety mechanisms (SBC, external watchdogs), MPU & freedom from interference, secure boot & HSE, automotive Ethernet, XCP
- **Level 4 · Master** — multi-core S32K3 lockstep, ASIL-D development workflow, MISRA & static analysis, OTA/reprogramming stacks, ISO/SAE 21434 & SecOC, AUTOSAR OS & timing analysis, HIL testing & CI, production ECU architecture, manufacturing & EOL test, zone-ECU capstone

## Local development

```bash
python3 -m venv .venv
.venv/bin/pip install mkdocs-material
.venv/bin/python -m mkdocs serve
```

## Related

- [Embedded Systems Mastery Path](https://sigilipelli.github.io/embedded-mastery-path/)
- [C Mastery Path](https://sigilipelli.github.io/c-mastery-path/)
- [C++ Mastery Path](https://sigilipelli.github.io/cpp-mastery-path/)
- [Python Mastery Path](https://sigilipelli.github.io/python-mastery-path/)
- [Java Mastery Path](https://sigilipelli.github.io/java-mastery-path/)
- [JavaScript Mastery Path](https://sigilipelli.github.io/javascript-mastery-path/)
- [TypeScript Mastery Path](https://sigilipelli.github.io/typescript-mastery-path/)
- [Shell Mastery Path](https://sigilipelli.github.io/shell-mastery-path/)
- [Go Mastery Path](https://sigilipelli.github.io/go-mastery-path/)
- [Rust Mastery Path](https://sigilipelli.github.io/rust-mastery-path/)
- [SQL Mastery Path](https://sigilipelli.github.io/sql-mastery-path/)
- [Ruby Mastery Path](https://sigilipelli.github.io/ruby-mastery-path/)
- [PHP Mastery Path](https://sigilipelli.github.io/php-mastery-path/)
- [Kotlin Mastery Path](https://sigilipelli.github.io/kotlin-mastery-path/)
- [Swift Mastery Path](https://sigilipelli.github.io/swift-mastery-path/)
- [Scala Mastery Path](https://sigilipelli.github.io/scala-mastery-path/)
- [R Mastery Path](https://sigilipelli.github.io/r-mastery-path/)
- [Dart Mastery Path](https://sigilipelli.github.io/dart-mastery-path/)
- [PowerShell Mastery Path](https://sigilipelli.github.io/powershell-mastery-path/)
- [AI & ML Mastery Path](https://sigilipelli.github.io/ai-ml-mastery-path/)
- [LLM Development Mastery Path](https://sigilipelli.github.io/llm-dev-mastery-path/)
- [RAG Mastery Path](https://sigilipelli.github.io/rag-mastery-path/)
- [AWS Mastery Path](https://sigilipelli.github.io/aws-mastery-path/)
- [Azure Mastery Path](https://sigilipelli.github.io/azure-mastery-path/)
- [Google Cloud Mastery Path](https://sigilipelli.github.io/gcp-mastery-path/)
- [IBM Cloud Mastery Path](https://sigilipelli.github.io/ibm-cloud-mastery-path/)
- [Adobe Creative Cloud Mastery Path](https://sigilipelli.github.io/adobe-mastery-path/)
