---
title: "Final Quiz"
description: "Test what you have learned across all four levels of S32K Automotive Embedded Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "What does this print?\n```c\n#include <stdio.h>\n#include <stdint.h>\nint main(void){uint8_t a=200,b=100; uint8_t s=a+b; uint16_t w=(uint16_t)a+b; int8_t n=(int8_t)200; printf(\"%u %u %d\\n\",s,w,n);}\n```\n",
    "options": [
      "44 300 -56",
      "44 44 200",
      "300 300 -56",
      "44 300 200"
    ],
    "correctIndex": 0,
    "explanation": "uint8_t arithmetic wraps modulo 256; casting 200 to int8_t gives -56 on two's complement targets. (Host-C model of the arithmetic; no S32K hardware was used.) The code was compiled, run and its output checked."
  },
  {
    "question": "What does this print?\n```c\n#include <stdio.h>\n#include <stdint.h>\n#define BIT(n) (1u<<(n))\nint main(void){uint32_t reg=0xF0; reg&=~(BIT(7)|BIT(4)); reg|=BIT(1); printf(\"0x%02X %d\\n\",reg,(reg&BIT(5))!=0);}\n```\n",
    "options": [
      "0xF2 1",
      "0x72 1",
      "0x62 0",
      "0x62 1"
    ],
    "correctIndex": 3,
    "explanation": "Clear bits 7 and 4, set bit 1; bit 5 remains set. (Host-C model of the arithmetic; no S32K hardware was used.) The code was compiled, run and its output checked."
  },
  {
    "question": "What is the CAN bus load for this schedule at 500 kbit/s? (frames are 111 bits, 10 per 10 ms)\n```c\n#include <stdio.h>\n#include <stdint.h>\nint main(void){double bits=111*10; double cap=500000*0.010; printf(\"%.1f%%\\n\",100*bits/cap);}\n```\n",
    "options": [
      "222%",
      "2.2%",
      "11.1%",
      "22.2%"
    ],
    "correctIndex": 3,
    "explanation": "1110 bits in a 10 ms window with 5000 bits of capacity. (Host-C model of the arithmetic; no S32K hardware was used.) The code was compiled, run and its output checked."
  },
  {
    "question": "How many bits is the CAN bit time for 500 kbit/s, and how many time quanta at an 8 MHz time-quantum clock?\n```c\n#include <stdio.h>\n#include <stdint.h>\nint main(void){double t=1e6/500000.0; printf(\"%.0f us %.0f tq\\n\",t,t*8);}\n```\n",
    "options": [
      "4 us 32 tq",
      "2 us 16 tq",
      "2 us 8 tq",
      "1 us 8 tq"
    ],
    "correctIndex": 1,
    "explanation": "Bit time = 1/baud; tq count = bit time x tq clock. (Host-C model of the arithmetic; no S32K hardware was used.) The code was compiled, run and its output checked."
  },
  {
    "question": "What is the sample point for 16 tq with sync 1, PROP+PS1 = 12 and PS2 = 3?\n```c\n#include <stdio.h>\n#include <stdint.h>\nint main(void){printf(\"%.1f%%\\n\",100.0*(1+12)/16);}\n```\n",
    "options": [
      "75.0%",
      "93.8%",
      "12.0%",
      "81.2%"
    ],
    "correctIndex": 3,
    "explanation": "Sample point = (sync + seg1) / total, here 13/16 (81.25%, printed as 81.2% by round-half-even). (Host-C model of the arithmetic; no S32K hardware was used.) The code was compiled, run and its output checked."
  },
  {
    "question": "Why is a CRC alone not enough to secure a CAN message?",
    "options": [
      "Anyone can recompute it; authentication needs a keyed MAC with freshness (SecOC)",
      "CAN forbids it",
      "It is too long",
      "It is slow"
    ],
    "correctIndex": 0,
    "explanation": "CRC detects accidental errors only."
  },
  {
    "question": "What is lowest-ID-wins arbitration an example of?",
    "options": [
      "Collision loss",
      "Random backoff",
      "Token passing",
      "Non-destructive bitwise arbitration, with dominant bits overriding recessive"
    ],
    "correctIndex": 3,
    "explanation": "The winner's frame is not corrupted."
  },
  {
    "question": "What do you need to enable before using a peripheral on S32K1?",
    "options": [
      "A debugger",
      "Nothing",
      "Its PCC clock (and the right clock source), then pin muxing if it uses pins",
      "Only interrupts"
    ],
    "correctIndex": 2,
    "explanation": "Writes to an unclocked peripheral are ignored or fault."
  },
  {
    "question": "What is the main reason to use DMA for a fast ADC stream?",
    "options": [
      "Fewer bits",
      "Less flash",
      "Better accuracy",
      "Move samples without CPU per-sample interrupts"
    ],
    "correctIndex": 3,
    "explanation": "CPU stays free."
  },
  {
    "question": "What is wrong with refreshing the watchdog in a periodic timer ISR?",
    "options": [
      "Too fast",
      "The main loop can be dead while the ISR keeps feeding it",
      "Nothing",
      "Too slow"
    ],
    "correctIndex": 1,
    "explanation": "Supervise real task progress."
  },
  {
    "question": "Which UDS response means 'service not supported in the active session'?",
    "options": [
      "0x13",
      "0x31",
      "0x78",
      "NRC 0x7F"
    ],
    "correctIndex": 3,
    "explanation": "0x11 is service not supported at all."
  },
  {
    "question": "Why do bootloaders verify the image before jumping to it?",
    "options": [
      "A partial or corrupted image could crash or behave unsafely",
      "Style",
      "Speed",
      "Law"
    ],
    "correctIndex": 0,
    "explanation": "Use CRC at minimum, a signature for authenticity."
  },
  {
    "question": "What does freedom from interference require?",
    "options": [
      "One big task",
      "No RTOS",
      "Lower-integrity software cannot corrupt higher-ASIL software (MPU, partitioning, timing protection)",
      "Wireless"
    ],
    "correctIndex": 2,
    "explanation": "A key ISO 26262 concept."
  },
  {
    "question": "What does SecOC add to a PDU?",
    "options": [
      "A truncated MAC computed over data and a freshness value",
      "Priority",
      "A CRC",
      "Encryption"
    ],
    "correctIndex": 0,
    "explanation": "Authenticity and replay protection."
  },
  {
    "question": "What does lockstep detect?",
    "options": [
      "Software bugs",
      "Bad CAN",
      "Random hardware faults in the core, by comparing two cores' outputs",
      "Stack overflow"
    ],
    "correctIndex": 2,
    "explanation": "Software bugs execute identically on both cores."
  },
  {
    "question": "Why disable or protect XCP in production firmware?",
    "options": [
      "Law",
      "Speed",
      "It allows memory read/write, which is a security risk",
      "Colour"
    ],
    "correctIndex": 2,
    "explanation": "Seed-and-key or removal."
  },
  {
    "question": "What makes an A/B OTA layout safer?",
    "options": [
      "Faster",
      "The new image is verified in the inactive bank before switching, with rollback",
      "Smaller",
      "Cheaper"
    ],
    "correctIndex": 1,
    "explanation": "Avoids bricking."
  },
  {
    "question": "What is a typical production ECU evidence set for ASIL claims?",
    "options": [
      "Requirements trace, reviews, static analysis, tests with coverage and safety analysis",
      "A demo",
      "Screenshots",
      "Source only"
    ],
    "correctIndex": 0,
    "explanation": "Evidence, not assertion."
  },
  {
    "question": "What is the right reaction to a failed ADC plausibility check?",
    "options": [
      "Send zeros silently",
      "Report a defined fault status, avoid sending stale values as valid, and enter the safe state",
      "Ignore",
      "Reset only"
    ],
    "correctIndex": 1,
    "explanation": "Fail safe and visibly."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### Pack and unpack a CAN message with CRC and alive counter

Implement crc8, encode and decode for a sensor message: a signed 0.1 C temperature, an rpm value, a 4-bit alive counter and a CRC-8 in byte 7. Decode must reject corrupted frames and wrong counters. This runs on your computer as ordinary C; it models the signal packing you would do on an S32K node, but uses no hardware.

**Starter code**

```c
#include <stdio.h>
#include <stdint.h>
#include <string.h>

typedef struct { int16_t temp_c10; uint16_t rpm; uint8_t counter; } Sample;

// TODO: static uint8_t crc8(const uint8_t *d, int n)            CRC-8, polynomial 0x07, initial value 0
// TODO: static void encode(const Sample *s, uint8_t f[8])        bytes 0-1 temp (LE, 0.1 C), 2-3 rpm (LE),
//       byte 4 low nibble = alive counter, byte 7 = CRC-8 over bytes 0..6, unused bytes 0
// TODO: static int decode(const uint8_t f[8], Sample *s, uint8_t expected_counter)
//       returns -1 on CRC mismatch, -2 on counter mismatch, 0 on success (check CRC first)

int main(void) {
    Sample in = { -125, 3000, 15 }, out;
    uint8_t f[8];
    encode(&in, f);
    for (int i = 0; i < 8; i++) printf("%02X ", f[i]);
    printf("\n");
    printf("decode: %d temp=%.1f rpm=%u ctr=%u\n", decode(f, &out, 15), out.temp_c10 / 10.0, out.rpm, out.counter);
    printf("wrong counter: %d\n", decode(f, &out, 0));
    f[2] ^= 0x01;
    printf("corrupted: %d\n", decode(f, &out, 15));
    return 0;
}
```

**Expected output**

```text
83 FF B8 0B 0F 00 00 56 
decode: 0 temp=-12.5 rpm=3000 ctr=15
wrong counter: -2
corrupted: -1
```

<details><summary>Show solution and explanation</summary>

<pre><code>#include &lt;stdio.h&gt;
#include &lt;stdint.h&gt;
#include &lt;string.h&gt;

typedef struct { int16_t temp_c10; uint16_t rpm; uint8_t counter; } Sample;

static uint8_t crc8(const uint8_t *d, int n) {
    uint8_t c = 0;
    for (int i = 0; i &lt; n; i++) {
        c ^= d[i];
        for (int b = 0; b &lt; 8; b++) c = (c &amp; 0x80) ? (uint8_t)((c &lt;&lt; 1) ^ 0x07) : (uint8_t)(c &lt;&lt; 1);
    }
    return c;
}

/* byte0-1 temp (little endian, signed, 0.1 C), byte2-3 rpm, byte4 low nibble counter, byte7 crc over 0..6 */
static void encode(const Sample *s, uint8_t f[8]) {
    memset(f, 0, 8);
    f[0] = (uint8_t)(s-&gt;temp_c10 &amp; 0xFF);
    f[1] = (uint8_t)((s-&gt;temp_c10 &gt;&gt; 8) &amp; 0xFF);
    f[2] = (uint8_t)(s-&gt;rpm &amp; 0xFF);
    f[3] = (uint8_t)(s-&gt;rpm &gt;&gt; 8);
    f[4] = s-&gt;counter &amp; 0x0F;
    f[7] = crc8(f, 7);
}

static int decode(const uint8_t f[8], Sample *s, uint8_t expected_counter) {
    if (crc8(f, 7) != f[7]) return -1;
    if ((f[4] &amp; 0x0F) != expected_counter) return -2;
    s-&gt;temp_c10 = (int16_t)(f[0] | (f[1] &lt;&lt; 8));
    s-&gt;rpm = (uint16_t)(f[2] | (f[3] &lt;&lt; 8));
    s-&gt;counter = f[4] &amp; 0x0F;
    return 0;
}

int main(void) {
    Sample in = { -125, 3000, 15 }, out;
    uint8_t f[8];
    encode(&amp;in, f);
    for (int i = 0; i &lt; 8; i++) printf(&quot;%02X &quot;, f[i]);
    printf(&quot;\n&quot;);
    printf(&quot;decode: %d temp=%.1f rpm=%u ctr=%u\n&quot;, decode(f, &amp;out, 15), out.temp_c10 / 10.0, out.rpm, out.counter);
    printf(&quot;wrong counter: %d\n&quot;, decode(f, &amp;out, 0));
    f[2] ^= 0x01;
    printf(&quot;corrupted: %d\n&quot;, decode(f, &amp;out, 15));
    return 0;
}
</code></pre>

<p>Little-endian signals are assembled byte by byte, the counter uses the low nibble, and the CRC covers bytes 0 to 6. Decode checks the CRC before trusting any field, then the counter. Verified by compiling and running with gcc on the host.</p>
</details>

### A UDS request handler

Implement respond for services 0x10, 0x22 and 0x3E with the specified negative response codes, including a DID that is only readable in the extended session. This is a host-side model of the service logic; real UDS also needs ISO-TP transport, timing (P2) and security access.

**Starter code**

```c
#include <stdio.h>
#include <stdint.h>
#include <string.h>

enum { SESSION_DEFAULT = 1, SESSION_EXTENDED = 3 };
static int session = SESSION_DEFAULT;

// TODO: static int respond(const uint8_t *req, int len, uint8_t *resp) -> response length
//   0x10 DiagnosticSessionControl: len must be 2 (else NRC 0x13), sub-function 1 or 3 (else NRC 0x12);
//        positive response 50 <sub> 00 32 01 F4
//   0x22 ReadDataByIdentifier: len must be 3 (NRC 0x13); DID F190 -> 62 F1 90 'V' 'I' 'N';
//        DID 0100 only in the extended session (otherwise NRC 0x7F), returns 62 01 00 2A; any other DID NRC 0x31
//   0x3E TesterPresent: 3E 00 -> 7E 00 (else NRC 0x13)
//   anything else: NRC 0x11 (service not supported). Negative response = 7F <sid> <nrc>

static void run(const char *label, const uint8_t *req, int len) {
    uint8_t resp[16];
    int n = respond(req, len, resp);
    printf("%-14s ->", label);
    for (int i = 0; i < n; i++) printf(" %02X", resp[i]);
    printf("\n");
}

int main(void) {
    run("read VIN", (uint8_t[]){0x22, 0xF1, 0x90}, 3);
    run("read 0100 dflt", (uint8_t[]){0x22, 0x01, 0x00}, 3);
    run("ext session", (uint8_t[]){0x10, 0x03}, 2);
    run("read 0100 ext", (uint8_t[]){0x22, 0x01, 0x00}, 3);
    run("bad subfunc", (uint8_t[]){0x10, 0x02}, 2);
    run("tester present", (uint8_t[]){0x3E, 0x00}, 2);
    run("short read", (uint8_t[]){0x22, 0xF1}, 2);
    run("unknown DID", (uint8_t[]){0x22, 0x12, 0x34}, 3);
    run("unknown svc", (uint8_t[]){0x85, 0x01}, 2);
    return 0;
}
```

**Expected output**

```text
read VIN       -> 62 F1 90 56 49 4E
read 0100 dflt -> 7F 22 7F
ext session    -> 50 03 00 32 01 F4
read 0100 ext  -> 62 01 00 2A
bad subfunc    -> 7F 10 12
tester present -> 7E 00
short read     -> 7F 22 13
unknown DID    -> 7F 22 31
unknown svc    -> 7F 85 11
```

<details><summary>Show solution and explanation</summary>

<pre><code>#include &lt;stdio.h&gt;
#include &lt;stdint.h&gt;
#include &lt;string.h&gt;

enum { SESSION_DEFAULT = 1, SESSION_EXTENDED = 3 };
static int session = SESSION_DEFAULT;

static int respond(const uint8_t *req, int len, uint8_t *resp) {
    if (len &lt; 1) return 0;
    uint8_t sid = req[0];
    switch (sid) {
    case 0x10:
        if (len != 2) { resp[0] = 0x7F; resp[1] = sid; resp[2] = 0x13; return 3; }
        if (req[1] != 1 &amp;&amp; req[1] != 3) { resp[0] = 0x7F; resp[1] = sid; resp[2] = 0x12; return 3; }
        session = req[1];
        resp[0] = 0x50; resp[1] = req[1]; resp[2] = 0x00; resp[3] = 0x32; resp[4] = 0x01; resp[5] = 0xF4;
        return 6;
    case 0x22:
        if (len != 3) { resp[0] = 0x7F; resp[1] = sid; resp[2] = 0x13; return 3; }
        if (req[1] == 0xF1 &amp;&amp; req[2] == 0x90) {
            memcpy(resp, (uint8_t[]){0x62, 0xF1, 0x90, &#x27;V&#x27;, &#x27;I&#x27;, &#x27;N&#x27;}, 6);
            return 6;
        }
        if (req[1] == 0x01 &amp;&amp; req[2] == 0x00) {
            if (session != SESSION_EXTENDED) { resp[0] = 0x7F; resp[1] = sid; resp[2] = 0x7F; return 3; }
            memcpy(resp, (uint8_t[]){0x62, 0x01, 0x00, 0x2A}, 4);
            return 4;
        }
        resp[0] = 0x7F; resp[1] = sid; resp[2] = 0x31; return 3;
    case 0x3E:
        if (len == 2 &amp;&amp; req[1] == 0x00) { resp[0] = 0x7E; resp[1] = 0x00; return 2; }
        resp[0] = 0x7F; resp[1] = sid; resp[2] = 0x13; return 3;
    default:
        resp[0] = 0x7F; resp[1] = sid; resp[2] = 0x11; return 3;
    }
}

static void run(const char *label, const uint8_t *req, int len) {
    uint8_t resp[16];
    int n = respond(req, len, resp);
    printf(&quot;%-14s -&gt;&quot;, label);
    for (int i = 0; i &lt; n; i++) printf(&quot; %02X&quot;, resp[i]);
    printf(&quot;\n&quot;);
}

int main(void) {
    run(&quot;read VIN&quot;, (uint8_t[]){0x22, 0xF1, 0x90}, 3);
    run(&quot;read 0100 dflt&quot;, (uint8_t[]){0x22, 0x01, 0x00}, 3);
    run(&quot;ext session&quot;, (uint8_t[]){0x10, 0x03}, 2);
    run(&quot;read 0100 ext&quot;, (uint8_t[]){0x22, 0x01, 0x00}, 3);
    run(&quot;bad subfunc&quot;, (uint8_t[]){0x10, 0x02}, 2);
    run(&quot;tester present&quot;, (uint8_t[]){0x3E, 0x00}, 2);
    run(&quot;short read&quot;, (uint8_t[]){0x22, 0xF1}, 2);
    run(&quot;unknown DID&quot;, (uint8_t[]){0x22, 0x12, 0x34}, 3);
    run(&quot;unknown svc&quot;, (uint8_t[]){0x85, 0x01}, 2);
    return 0;
}
</code></pre>

<p>Validate length first, then sub-function or identifier, then session state, returning 7F SID NRC on failure. Positive responses add 0x40 to the service ID. Verified by compiling and running with gcc on the host; no ECU or CAN hardware was used.</p>
</details>
