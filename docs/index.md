# S32K Automotive Embedded Mastery Path

A structured, module-wise training program for **automotive embedded
development on NXP's S32K microcontroller family** — from your first
register write to designing production-grade ECU firmware. Real C code in
every module, with the automotive context (CAN bus, AUTOSAR, ISO 26262
functional safety) woven in from the first lesson, because that context is
what separates automotive firmware from hobby embedded work.

**Hardware is optional to start.** Every lesson is written so you can follow
the code and concepts fully without a board. When you want to run things for
real, an [S32K144EVB](https://www.nxp.com/design/design-center/development-boards-and-designs/S32K144EVB)
or S32K344 evaluation board (~$50–200) is the reference target, and the
open-source [Renode](https://renode.io/) emulator is a free middle ground —
lesson 1 covers all three options honestly.

## How the program is organized

| Level | Focus | Modules |
|-------|-------|---------|
| [Level 1 · Entry](level-1/index.md) | The automotive landscape, S32 Design Studio, clocks, GPIO, UART, ADC, CAN fundamentals, timers/PWM, safety basics | 9 topics + 1 capstone |
| [Level 2 · Intermediate](level-2/index.md) | FlexCAN deep dive, eDMA, LPSPI/LPI2C, FreeRTOS, low-power modes, flash/EEPROM, bootloaders, UDS diagnostics | 9 topics + 1 project |
| [Level 3 · Advanced](level-3/index.md) | AUTOSAR Classic, RTD/MCAL, CAN FD & DBC, LIN, safety mechanisms, MPU, secure boot/HSE, XCP | 9 topics + 1 project |
| [Level 4 · Master](level-4/index.md) | Multi-core lockstep, ASIL-D workflow, MISRA, OTA stacks, cybersecurity, HIL/CI, production ECU architecture | 9 topics + 1 capstone |

## How to use this site

- Work through each level in order — later modules assume earlier ones.
- Code is real C targeting the S32K family: examples are labeled
  **SDK-style** (NXP S32 SDK / RTD API shapes) or **register-level**
  (straight out of the reference manual) so you always know what layer
  you're looking at.
- Each level ends with a project that combines everything learned in that
  level.
- Use the search bar (top of the page) to jump straight to a topic.

New to microcontrollers entirely? The
[Embedded Systems Mastery Path](https://sigilipelli.github.io/embedded-mastery-path/)
covers general MCU basics with a free browser simulator, and the
[C Mastery Path](https://sigilipelli.github.io/c-mastery-path/) covers the C
language itself — both pair well with this course.

Start here → [Level 1 · Entry](level-1/index.md)

🎥 **Prefer video?** Watch the [Mastery Path video series](https://youtube.com/@sigilipelli) on YouTube — Shorts and full walkthroughs of these lessons.

## More from the Mastery Path series

Free, structured, module-wise training across 31 other languages and platforms:

<div class="mastery-grid-wrap">
<p class="mastery-grid-category">Languages</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/python-mastery-path/">🐍 Python</a>
  <a href="https://sigilipelli.github.io/java-mastery-path/">☕ Java</a>
  <a href="https://sigilipelli.github.io/javascript-mastery-path/">🟨 JavaScript</a>
  <a href="https://sigilipelli.github.io/typescript-mastery-path/">🔷 TypeScript</a>
  <a href="https://sigilipelli.github.io/shell-mastery-path/">🐚 Shell/Bash</a>
  <a href="https://sigilipelli.github.io/powershell-mastery-path/">💻 PowerShell</a>
  <a href="https://sigilipelli.github.io/c-mastery-path/">🇨 C</a>
  <a href="https://sigilipelli.github.io/cpp-mastery-path/">➕ C++</a>
  <a href="https://sigilipelli.github.io/go-mastery-path/">🐹 Go</a>
  <a href="https://sigilipelli.github.io/rust-mastery-path/">🦀 Rust</a>
  <a href="https://sigilipelli.github.io/sql-mastery-path/">🗄️ SQL</a>
  <a href="https://sigilipelli.github.io/ruby-mastery-path/">💎 Ruby</a>
  <a href="https://sigilipelli.github.io/php-mastery-path/">🐘 PHP</a>
  <a href="https://sigilipelli.github.io/kotlin-mastery-path/">🟣 Kotlin</a>
  <a href="https://sigilipelli.github.io/swift-mastery-path/">🐦 Swift</a>
  <a href="https://sigilipelli.github.io/dart-mastery-path/">🎯 Dart</a>
  <a href="https://sigilipelli.github.io/scala-mastery-path/">🔴 Scala</a>
  <a href="https://sigilipelli.github.io/r-mastery-path/">📊 R</a>
</div>
<p class="mastery-grid-category">Cloud Platforms</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/aws-mastery-path/">☁️ AWS</a>
  <a href="https://sigilipelli.github.io/azure-mastery-path/">☁️ Azure</a>
  <a href="https://sigilipelli.github.io/gcp-mastery-path/">☁️ GCP</a>
  <a href="https://sigilipelli.github.io/ibm-cloud-mastery-path/">☁️ IBM Cloud</a>
  <a href="https://sigilipelli.github.io/adobe-mastery-path/">🎨 Adobe</a>
</div>
<p class="mastery-grid-category">AI / ML / LLM</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/ai-ml-mastery-path/">🤖 AI/ML</a>
  <a href="https://sigilipelli.github.io/llm-dev-mastery-path/">🧠 LLM Dev</a>
  <a href="https://sigilipelli.github.io/rag-mastery-path/">📚 RAG</a>
  <a href="https://sigilipelli.github.io/edge-ai-mastery-path/">📱 Edge AI</a>
</div>
<p class="mastery-grid-category">Embedded Systems</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/embedded-mastery-path/">🔌 Embedded</a>
  <a href="https://sigilipelli.github.io/embedded-linux-mastery-path/">🐧 Embedded Linux</a>
  <a href="https://sigilipelli.github.io/embedded-python-mastery-path/">🐍 Embedded Python</a>
  <a href="https://sigilipelli.github.io/freertos-mastery-path/">⏱️ FreeRTOS</a>
</div>
</div>
