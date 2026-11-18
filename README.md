# S32K Automotive Embedded Mastery Path

A free, structured, module-wise training program for automotive embedded
development on NXP's S32K microcontroller family — entry level to master
level, with real C code in every module and a hands-on project at the end of
each level. CAN bus, AUTOSAR, and ISO 26262 functional-safety context is
woven in throughout, because that context is what separates automotive
firmware from hobby embedded work. Hardware is optional: follow along at the
code/concepts level, use an S32K144EVB/S32K344 devkit, or explore the Renode
emulator.

**Live site:** https://learnmasterypath.com/s32k-mastery-path/

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

- [Embedded Systems Mastery Path](https://learnmasterypath.com/embedded-mastery-path/)
- [C Mastery Path](https://learnmasterypath.com/c-mastery-path/)
- [C++ Mastery Path](https://learnmasterypath.com/cpp-mastery-path/)
- [Python Mastery Path](https://learnmasterypath.com/python-mastery-path/)
- [Java Mastery Path](https://learnmasterypath.com/java-mastery-path/)
- [JavaScript Mastery Path](https://learnmasterypath.com/javascript-mastery-path/)
- [TypeScript Mastery Path](https://learnmasterypath.com/typescript-mastery-path/)
- [Shell Mastery Path](https://learnmasterypath.com/shell-mastery-path/)
- [Go Mastery Path](https://learnmasterypath.com/go-mastery-path/)
- [Rust Mastery Path](https://learnmasterypath.com/rust-mastery-path/)
- [SQL Mastery Path](https://learnmasterypath.com/sql-mastery-path/)
- [Ruby Mastery Path](https://learnmasterypath.com/ruby-mastery-path/)
- [PHP Mastery Path](https://learnmasterypath.com/php-mastery-path/)
- [Kotlin Mastery Path](https://learnmasterypath.com/kotlin-mastery-path/)
- [Swift Mastery Path](https://learnmasterypath.com/swift-mastery-path/)
- [Scala Mastery Path](https://learnmasterypath.com/scala-mastery-path/)
- [R Mastery Path](https://learnmasterypath.com/r-mastery-path/)
- [Dart Mastery Path](https://learnmasterypath.com/dart-mastery-path/)
- [PowerShell Mastery Path](https://learnmasterypath.com/powershell-mastery-path/)
- [AI & ML Mastery Path](https://learnmasterypath.com/ai-ml-mastery-path/)
- [LLM Development Mastery Path](https://learnmasterypath.com/llm-dev-mastery-path/)
- [RAG Mastery Path](https://learnmasterypath.com/rag-mastery-path/)
- [AWS Mastery Path](https://learnmasterypath.com/aws-mastery-path/)
- [Azure Mastery Path](https://learnmasterypath.com/azure-mastery-path/)
- [Google Cloud Mastery Path](https://learnmasterypath.com/gcp-mastery-path/)
- [IBM Cloud Mastery Path](https://learnmasterypath.com/ibm-cloud-mastery-path/)
- [Adobe Creative Cloud Mastery Path](https://learnmasterypath.com/adobe-mastery-path/)
