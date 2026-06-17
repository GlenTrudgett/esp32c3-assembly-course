# Bare Metal: Assembly Programming the ESP32-C3
## Volume 2: Sensors and Signals — Twelve Weeks of Real-World Interfacing

---

## Table of Contents

- About Volume 2
- Prerequisites
- The 37-in-1 Sensor Kit
- Voltage: The 3.3V Rule
- Wiring Conventions
- Project Setup

**PHASE 6: THE DIGITAL WORLD (Weeks 1–3)**
- Chapter 15: Digital Inputs — Twelve Sensors, One Skill
- Chapter 16: Digital Outputs — Buzzers, LEDs, Lasers, and a Relay
- Chapter 17: PWM From First Principles — Tone, Colour, and the LEDC Peripheral

**PHASE 7: THE ANALOG WORLD (Weeks 4–5)**
- Chapter 18: The SAR ADC — Reading Continuous Reality
- Chapter 19: The Joystick — Mixed-Signal Input

**PHASE 8: PROTOCOLS IN ASSEMBLY (Weeks 6–9)**
- Chapter 20: 1-Wire and the DS18B20 — Your First Real Protocol
- Chapter 21: The DHT11 — Reading Forty Bits by Stopwatch
- Chapter 22: The Rotary Encoder — Quadrature and GPIO Interrupts
- Chapter 23: Infrared NEC — Build a Remote Control Link

**PHASE 9: INTEGRATION (Weeks 10–12)**
- Chapter 24: The Heartbeat Monitor — Sampling and Signal Processing
- Chapter 25: Capstone — The Sensor Station
- Chapter 26: The BLE Sensor Beacon — Closing the Loop with Volume 1

**APPENDICES**
- Appendix A: The Complete 37-Module Catalog
- Appendix B: Volume 2 Peripheral Register Reference
- Appendix C: Wiring Gotchas and Module Quirks

---

## About Volume 2

Volume 1 took you from `li a0, 42` to WiFi and BLE — fourteen chapters of instruction set, calling convention, memory-mapped I/O, interrupts, and bare-metal boot. Every one of those chapters was about *the processor*.

Volume 2 is about *the world*. A microcontroller earns its keep by sensing and acting: reading a temperature, detecting a knock, driving a buzzer, decoding a remote control. The 37-in-1 sensor kit — the ubiquitous box of KY-numbered modules sold for Arduino — is the cheapest comprehensive laboratory for this ever made. This volume drives every one of its 37 modules from hand-written RISC-V assembly on your ESP32-C3 SuperMini.

The deeper agenda is the same as Volume 1: fundamentals over specifics. By the end you will have implemented, from raw register writes and cycle counting:

- Debounced digital input — the universal skill behind twelve of the kit's modules
- Pulse-width modulation, first bit-banged, then with the LEDC hardware peripheral
- Analog-to-digital conversion with the SAR ADC
- Three genuine serial protocols by stopwatch: 1-Wire (DS18B20), the DHT11's single-wire format, and NEC infrared
- Quadrature decoding with GPIO interrupts
- A signal-processing pipeline: sampling, filtering, and peak detection on a live heartbeat

These are not Arduino library calls. There is no `digitalRead()`, no `analogRead()`, no `IRremote.h`. There is you, the Technical Reference Manual, and the fetch-decode-execute loop you internalised in Section 1.0 of Volume 1. When you finish, "how does that library actually work?" will never again be a mystery — you will have written the library.

**Conventions are unchanged from Volume 1.** GNU assembler syntax; function names match their `.S` filenames; each function opens with its C-equivalent signature and register usage; the single-branch Git workflow with one annotated tag per chapter; three check-your-understanding questions before each chapter's references; the Neovim debug workflow from Volume 1 Appendix H is your primary verification tool throughout. Australian spelling, as before.

**One week per chapter is a guideline, not a deadline.** Protocol chapters (20–23) are the heart of this volume and the most demanding — budget accordingly.

---

## Prerequisites

**Volume 1, complete.** Not skimmed — complete. Volume 2 leans constantly on:

| Volume 1 material | Used here for |
|------------------|---------------|
| Ch 8: MMIO and GPIO (init, W1TS/W1TC) | Every chapter |
| Ch 10: `mcycle`, cycle counting, `delay_cycles` | All timing: debounce, PWM, every protocol |
| Ch 11: Trap handler, interrupt setup | Ch 22 (GPIO interrupts), Ch 24 |
| Ch 5–6: Stack, calling convention, structs in memory | Multi-byte protocol buffers |
| Ch 13–14: `wifi_asm` / `ble_asm` | Ch 26 |
| Appendix H: OpenOCD + GDB debug workflow | Verifying every register sequence |

**Hardware:**
- Your ESP32-C3 SuperMini from Volume 1 (the same `sdkconfig.defaults`, the same USB-C cable)
- The 37-in-1 sensor kit (KY-001 through KY-040 — the catalog in Appendix A maps every module)
- A breadboard and female-to-male jumper wires (the modules have male header pins)
- *(Optional but recommended)* A second SuperMini for Chapter 23's two-board infrared link — the chapter includes a single-board loopback alternative

**No other tools.** No oscilloscope, no logic analyser. Where you would reach for a scope, this book reaches for `mcycle` and the debugger — measuring the silicon with the silicon. (If you *do* own a logic analyser, the protocol chapters are a wonderful excuse to use it, and each one notes what to look at.)

---

## The 37-in-1 Sensor Kit

The kit is a grab-bag of small breakout modules, each carrying one sensor or actuator plus the minimum support components (current-limiting resistors, pull-ups, sometimes a comparator). They were designed for 5V Arduinos, which raises a voltage question answered in the next section.

The modules fall into five interface families, and the book is organised around the families, not the module numbers:

| Family | Electrical reality | Modules | Chapters |
|--------|-------------------|---------|----------|
| Digital input | A switch or open-collector output: reads 0 or 1 | KY-002, 003, 004, 010, 017, 020, 021, 025, 031, 032, 033, 036 | 15 |
| Digital output | You drive it: on or off | KY-008, 011, 012, 019, 029, 034 | 16 |
| PWM-driven output | You drive it *fast*: tone and brightness | KY-006, 009, 016, 027 | 17 |
| Analog output | A voltage proportional to something physical | KY-013, 018, 024, 026, 028, 035, 037, 038, 039 | 18, 24 |
| Protocol / timing | A digital signal where *time* carries the data | KY-001, 005, 015, 022, 023, 040 | 19–23 |

Learn one family and you have learned all its members — exactly as one RISC-V instruction format taught you a dozen instructions in Volume 1. Chapter 15 builds one debounced-input driver and then reads twelve different sensors with it, unchanged.

**A note on the comparator modules.** Several "sensor" modules (KY-026 flame, KY-037/038 microphones) carry an LM393 comparator and offer *both* outputs: a digital pin (the comparator's verdict against a trimpot threshold) and an analog pin (the raw sensor voltage). You will read both — digital in Chapter 15's family, analog in Chapter 18 — and the comparison teaches the difference between a measurement and a decision.

---

## Voltage: The 3.3V Rule

Your SuperMini is a 3.3V device. Its GPIO pins output 3.3V and must never be driven above 3.3V. The kit was designed for 5V Arduinos. Read this section twice.

**The rule: power every module from the SuperMini's 3V3 pin unless this table says otherwise.**

| Situation | Verdict |
|-----------|---------|
| Switch-type modules (buttons, tilt, reed, hall, knock, vibration) | 3.3V — they are just switches and resistors |
| LEDs, laser (KY-008), buzzers | 3.3V — slightly dimmer/quieter than at 5V; fine |
| LM393 comparator modules (KY-026, 037, 038, etc.) | 3.3V — the LM393 works from 2V; re-tune the trimpot at 3.3V |
| Thermistor, photoresistor, linear hall (analog dividers) | 3.3V — *required*, so the output can never exceed the ADC range |
| DS18B20 (KY-001), DHT11 (KY-015) | 3.3V — both are specified down to 3.0V |
| IR receiver (KY-022) | 3.3V — VS1838-class receivers run at 2.7–5.5V |
| Joystick (KY-023) | 3.3V — it is two potentiometers; full-scale becomes 3.3V |
| **Relay (KY-019)** | **5V coil.** Power its VCC from the SuperMini's **5V pin** (USB rail). The 3.3V GPIO drives the transistor input fine. The grounds are already common. **Never switch mains voltage in this course — use the relay on low-voltage loads only.** |
| Heartbeat (KY-039) | 3.3V works; signal is small — Chapter 24 deals with it in software |

**Never** power a module at 5V and feed its output to a GPIO. If a module misbehaves at 3.3V, the answer is the trimpot or the software threshold — not the 5V pin.

---

## Wiring Conventions

Module pin labels in this kit are inconsistent: `S`/`+`/`-`, `S`/`VCC`/`GND`, sometimes silkscreen on the wrong side. Three rules prevent every wiring accident:

1. **Identify ground first.** On most three-pin modules the `-` pin connects to the bulk ground plane — follow the PCB trace, or look for the pin tied to the large copper pour. When in doubt, the middle pin is *usually* VCC on this kit's three-pin modules — but *verify per module*; Appendix C lists the known exceptions.
2. **Common ground always.** Every module's ground ties to a SuperMini GND pin. No exceptions, including the relay on the 5V rail.
3. **Default pin assignments** used throughout this volume (chosen around the SuperMini's constraint that GPIO11–17 do not exist and GPIO18/19 are USB):

| Purpose | GPIO | Notes |
|---------|------|-------|
| Digital sensor input | GPIO10 | The volume's standard input pin |
| Digital/PWM output | GPIO7 | The volume's standard output pin |
| Second output / second input | GPIO6 | RGB, encoder DT, IR receiver |
| Third output | GPIO5 | RGB blue channel |
| Analog inputs | GPIO0–GPIO4 | ADC1 channels 0–4 — the *only* ADC pins you will use |
| Onboard LED | GPIO8 | Status indication, free of charge (Volume 1, Ch 8) |
| BOOT button | GPIO9 | A free extra input (active-low, has pull-up) |
| UART console | GPIO20/21 | Reserved — your `uart_puts` from Volume 1 Ch 9 |

Every chapter states its wiring as a table in this format. Rewire deliberately; check ground, check VCC, check signal, then power on.

---

## Project Setup

Volume 2 lives in the same repository as Volume 1, continuing the chapter numbering and the tag scheme. One repo, one branch, one tag per chapter — the discipline is unchanged.

```bash
cd ~/esp32c3-assembly-course

# Each Volume 2 chapter starts exactly like a Volume 1 chapter:
cp -r template ch15
cd ch15
idf.py set-target esp32c3
```

The template is unchanged: the `sdkconfig.defaults` 4MB flash setting carries over, `main/CMakeLists.txt` still lists each `.S` file you create, and `app_main()` still ends in the `vTaskDelay` idle loop wherever your top-level assembly returns (Volume 1, Chapter 1 scaffold). From Chapter 15 onward the scaffold is the Phase-3 form — C provides only `app_main` calling `asm_main`, nothing else:

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

extern void asm_main(void);

void app_main(void) {
    asm_main();

    /* app_main() must not return — see Volume 1, Chapter 1 scaffold comment */
    while (1) {
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

Most Volume 2 programs never return from `asm_main` — they are sensor loops. The idle loop is there for the chapters whose exercises run once and finish.

A shared library grows through this volume. Chapter 15 creates `gpio.S` (input functions joining Volume 1's output functions), Chapter 10 of Volume 1 gave you `delay.S` — copy it into each chapter that needs it, and Chapter 18 adds `adc.S`. By the capstone you will assemble a project from your own driver files, which is precisely how real embedded codebases grow.

---

# PHASE 6: THE DIGITAL WORLD
## Weeks 1 – 3 | Switches, Actuators, and Time-Sliced Output

---

# Chapter 15: Digital Inputs — Twelve Sensors, One Skill

## Learning Objectives

By the end of this chapter you will be able to:

- Configure any GPIO as an input with internal pull-up or pull-down, from raw IOMUX writes
- Read pin state through `GPIO_IN_REG` and explain every bit of the read
- Explain switch bounce from first principles and defeat it three different ways
- Detect edges (presses, knocks, interruptions) by polling, without missing events
- Drive twelve different kit modules with one unchanged driver — and articulate *why* that works

## 15.1 One Electrical Idea, Twelve Packages

Every module in this chapter is, electrically, the same thing: a contact (or transistor) that either connects the signal pin to ground or lets it float. The physics behind the contact differs wildly — a spring (KY-002), a mercury bead (KY-017), a magnetic reed (KY-021, KY-025), a hall-effect IC (KY-003), a phototransistor staring at an IR LED (KY-010, KY-032, KY-033), your finger's capacitance (KY-036), a metal spring striking a post (KY-031), or a plain button (KY-004).

The processor cannot see the physics. It sees one bit in `GPIO_IN_REG`. Master that bit and you have mastered the chapter.

## 15.2 GPIO Input Registers

Volume 1 Chapter 8 configured *outputs*: IOMUX function select, then `GPIO_ENABLE`. Input is the mirror image, and adds two IOMUX bits you have not used yet:

| Register | Address | Role |
|----------|---------|------|
| `IO_MUX_GPIOn_REG` | `0x60009004 + 4·n` | Per-pin pad configuration |
| `GPIO_ENABLE_W1TC_REG` | `0x60004028` | Clear output-enable (make the pad an input) |
| `GPIO_IN_REG` | `0x6000403C` | Live pin states, one bit per GPIO |

The IOMUX bits that matter for input (ESP32-C3 TRM, IO MUX chapter):

| Bit | Name | Meaning |
|-----|------|---------|
| 14:12 | `MCU_SEL` | Pad function; `1` = GPIO |
| 9 | `FUN_IE` | Input enable — *without this the pad reads 0 forever* |
| 8 | `FUN_WPU` | Weak pull-up (~45 kΩ) |
| 7 | `FUN_WPD` | Weak pull-down |

`FUN_IE` is the classic trap: a pad with `MCU_SEL=1` but `FUN_IE=0` is configured as GPIO yet electrically deaf. If a sensor "always reads zero," check this bit first — `>x/1xw 0x60009004+4*n` in the debug REPL shows you the truth.

## 15.3 The Input Driver

Create `gpio.S`. These two functions join your Volume 1 output functions and serve every remaining chapter of the book:

```asm
# gpio.S — GPIO input driver (joins Volume 1 Ch8 output functions)

        .equ  GPIO_BASE,            0x60004000
        .equ  GPIO_ENABLE_W1TC_OFF, 0x0028
        .equ  GPIO_IN_OFF,          0x003C
        .equ  IO_MUX_BASE,          0x60009000

        .equ  IOMUX_MCU_SEL_GPIO,   (1 << 12)
        .equ  IOMUX_FUN_IE,         (1 << 9)
        .equ  IOMUX_FUN_WPU,        (1 << 8)

        .text

# gpio_init_input_pullup(int pin) — configure pin as input, weak pull-up on
# a0 = pin number (0..21)
# clobbers t0, t1, t2
        .global gpio_init_input_pullup
gpio_init_input_pullup:
        # 1. IOMUX: GPIO function + input enable + pull-up
        li    t0, IO_MUX_BASE
        slli  t1, a0, 2                  # pin * 4
        add   t0, t0, t1
        addi  t0, t0, 4                  # IO_MUX_GPIOn = base + 4 + 4n
        li    t1, IOMUX_MCU_SEL_GPIO | IOMUX_FUN_IE | IOMUX_FUN_WPU
        sw    t1, 0(t0)

        # 2. Ensure the output driver is OFF for this pin
        li    t0, GPIO_BASE
        li    t1, 1
        sll   t1, t1, a0                 # bit mask for pin
        sw    t1, GPIO_ENABLE_W1TC_OFF(t0)
        ret

# gpio_read(int pin) -> int — returns 0 or 1
# a0 = pin number; result in a0
# clobbers t0, t1
        .global gpio_read
gpio_read:
        li    t0, GPIO_BASE
        lw    t1, GPIO_IN_OFF(t0)        # all 22 pins in one read
        srl   t1, t1, a0                 # shift our pin to bit 0
        andi  a0, t1, 1
        ret
```

Note what `gpio_read` does *not* do: no masking before the shift, no branch. One load, one shift, one and. The entire pin bank arrives in a single bus read — when Chapter 25 polls four sensors, it will read `GPIO_IN_REG` once and slice it four ways.

**First test — the free button.** Before unpacking a single module, the BOOT button on GPIO9 is already wired to ground with a pull-up. In your `asm_main`: init GPIO9, loop reading it, mirror the (inverted) value onto the onboard LED. Press BOOT, LED responds. You have closed the input→decision→output loop that defines every embedded system, with zero wiring.

## 15.4 Bounce — and Three Ways to Kill It

Press a real button while watching it at full speed and you will not see one edge. A mechanical contact closing is a collision: the metal leaves *bounce*, connecting and disconnecting for anywhere from tens of microseconds to ~20 milliseconds before settling. Your 160 MHz processor — 6.25 ns per cycle, from Volume 1 Section 1.0 — experiences those 10 ms of chatter as 1.6 *million* cycles of garbage. A naive edge counter will count one press as five, nine, or thirty.

See it yourself before fixing it:

```asm
# bounce_demo.S — count raw falling edges on the BOOT button, print per second
# Demonstrates the problem; uses uart_puts/uart_putdec from Volume 1 Ch9
```

Run it, press the button ten times, and read the count. It will not be ten. *(Exercise 15.1 has you build this — the number you record is the chapter's motivation.)*

**Fix 1 — Settle-and-confirm (the workhorse).** On detecting a change, wait out the bounce window, then read again. If the reading agrees, it is real:

```asm
# debounced_read(int pin) -> int — stable 0/1, ~5ms worst case
# a0 = pin; result in a0
# clobbers t0-t2, s0, s1, ra-safe (leaf calls delay_cycles)
        .global debounced_read
debounced_read:
        addi  sp, sp, -16
        sw    ra, 12(sp)
        sw    s0, 8(sp)
        sw    s1, 4(sp)

        mv    s0, a0                     # s0 = pin
1:      mv    a0, s0
        jal   ra, gpio_read
        mv    s1, a0                     # first sample
        li    a0, 800000                 # 5 ms at 160 MHz
        jal   ra, delay_cycles           # (Volume 1, Ch10)
        mv    a0, s0
        jal   ra, gpio_read
        bne   a0, s1, 1b                 # disagreement → still bouncing, retry

        lw    s1, 4(sp)
        lw    s0, 8(sp)
        lw    ra, 12(sp)
        addi  sp, sp, 16
        ret
```

**Fix 2 — The counting filter.** Sample at a fixed rate (say every 1 ms); declare a new state only after N consecutive identical samples. This is what production firmware does, because it costs no blocking delay — it rides a loop you were running anyway. You build it in Exercise 15.3.

**Fix 3 — Don't.** Some of this chapter's sensors *must not* be debounced. A knock (KY-031) or vibration (KY-002) *is* a burst of bounces — the chatter is the signal. For those, you detect the first edge and then *ignore the pin* for a dead-time window. Same delay machinery, opposite philosophy. Knowing which fix fits which sensor is the actual skill.

## 15.5 Edge Detection by Polling

Level is "the button *is* down." Edge is "the button *went* down." Almost everything interesting is an edge. The polling pattern, which you will write so many times this volume it becomes reflex:

```asm
# Pseudocode shape — previous state lives in a saved register
#   s0 = pin, s1 = previous reading
# loop:
#   read pin -> a0
#   if a0 == 0 && s1 == 1:   falling edge — act
#   s1 = a0
#   (do other work — the loop must keep spinning)
```

The rule that makes polling correct: *the loop period must be shorter than the shortest event you must not miss.* A human press lasts >50 ms; a photo-interrupter (KY-010) pulse from a fast-moving flag might last 200 µs. Your loop budget is dictated by the fastest sensor on it. Chapter 22 introduces interrupts precisely for the events that make polling budgets impossible.

## 15.6 The Twelve-Module Tour

Wire each module — `-`→GND, middle→3V3, `S`→GPIO10 (exceptions in Appendix C) — and run the *same* `debounced_read` test program against all of them. For each, the table says what "active" means and which debounce philosophy applies:

| Module | Physics | Active state | Debounce approach |
|--------|---------|--------------|-------------------|
| KY-004 button | Contact | Low while pressed | Fix 1 or 2 |
| KY-020 tilt | Rolling ball | Changes with orientation | Fix 1 |
| KY-017 mercury | Mercury bead | Changes with tilt | Fix 1 |
| KY-021 / KY-025 reed | Magnetic reed | Low near magnet | Fix 1 |
| KY-003 hall | Hall IC | Low near magnet (one pole!) | None needed — IC output is clean |
| KY-036 touch | Transistor + your finger | Noisy high when touched | Fix 2 (counting) |
| KY-002 vibration | Spring contact | Bursts when shaken | Fix 3 (dead-time) |
| KY-031 knock | Spring + post | Brief pulse on impact | Fix 3 |
| KY-010 photo-interrupter | IR beam in slot | Changes when beam broken | None — optical, no bounce |
| KY-032 obstacle | Reflected IR + comparator | Low when object near (trimpot!) | Fix 2 |
| KY-033 line tracker | Reflected IR, aimed down | Low over dark surface | Fix 2 |
| KY-026 flame (digital out) | IR phototransistor + LM393 | Low seeing flame | Fix 2 |

Notice the pattern in the right column: *contacts bounce, semiconductors don't.* That single sentence sorts every digital sensor you will ever meet.

## 15.7 Exercises

**Exercise 15.1: Measure the bounce.** Build the raw edge counter from 15.4 on KY-004. Press exactly ten times; record the count. Now insert `debounced_read` and repeat. Commit both numbers in a comment — they are your before/after proof.

Sample solution core (raw counter):

```asm
# bounce_count.S — raw falling-edge counter, reports every ~2 s over UART
        .global asm_main
asm_main:
        addi  sp, sp, -16
        sw    ra, 12(sp)
        li    a0, 10
        jal   ra, gpio_init_input_pullup
        li    s0, 1                      # s0 = previous level (idle high)
        li    s1, 0                      # s1 = edge count
loop:
        li    a0, 10
        jal   ra, gpio_read
        beq   a0, s0, no_edge
        bnez  a0, store                  # only count 1→0
        addi  s1, s1, 1
store:  mv    s0, a0
no_edge:
        # (periodic UART report of s1 elided — Volume 1 Ch9/Ch10 pattern)
        j     loop
```

**Exercise 15.2: Orientation alarm.** KY-020 tilt + active buzzer concept (preview of Ch16): onboard LED solid when level, blinking when tilted. Two states, one edge detector, your first sensor-driven state machine.

**Exercise 15.3: The counting debouncer.** Implement Fix 2: sample GPIO10 every 1 ms (cycle-paced loop, not `delay` blocking); a state change is accepted after 5 consecutive agreeing samples. Verify it on the touch sensor (KY-036), which defeats Fix 1's single re-check.

**Exercise 15.4: Knock pattern lock.** KY-031 + Fix 3 dead-time (30 ms). Detect a *rhythm*: three knocks where the second gap is at least twice the first (knock — knock — pause — knock) lights the LED for two seconds. You will need `mcycle` timestamps per knock (Volume 1, Ch10). This is the chapter boss-fight: edges, dead-time, and time arithmetic in one program.

## 15.8 Git Checkpoint

```bash
cd ~/esp32c3-assembly-course
git add ch15/
git commit -m "feat(ch15): gpio input driver + debounce, 12 digital sensors verified"

# - [ ] Exercise 15.1: bounce counts recorded before/after
# - [ ] Exercise 15.3: counting debouncer passes touch-sensor test
# - [ ] Exercise 15.4: knock pattern recognised reliably

git tag -a ch15-complete -m "Chapter 15: Digital inputs — one driver, twelve sensors"
git push --follow-tags        # if using a remote
```

## 15.9 Chapter Summary

- All twelve modules reduce to one bit in `GPIO_IN_REG`; the physics differs, the read does not
- Input needs three IOMUX facts: `MCU_SEL=1`, `FUN_IE=1`, and a pull resistor chosen deliberately
- Contacts bounce for milliseconds — millions of cycles; debounce by settle-and-confirm, by counting filter, or by dead-time depending on whether the chatter is noise or signal
- Edges, not levels, carry meaning; polling works when the loop is faster than the shortest event

**Check your understanding:**
1. A pad reads 0 regardless of the input voltage. Which single IOMUX bit do you check first, and with what debug REPL command?
2. Why must the knock sensor *not* use `debounced_read`, and what replaces it?
3. Your polling loop takes 3 ms per pass. Can it reliably catch a 200 µs photo-interrupter pulse? What are your two options if not?

## 15.10 References for This Chapter

- ESP32-C3 Technical Reference Manual — IO MUX & GPIO Matrix chapter: [https://www.espressif.com/sites/default/files/documentation/esp32-c3_technical_reference_manual_en.pdf](https://www.espressif.com/sites/default/files/documentation/esp32-c3_technical_reference_manual_en.pdf)
- Jack Ganssle, "A Guide to Debouncing" — the definitive field study of real switch bounce: [http://www.ganssle.com/debouncing.htm](http://www.ganssle.com/debouncing.htm)
- KY-module electrical summaries (community reference): [https://arduinomodules.info](https://arduinomodules.info)

---

# Chapter 16: Digital Outputs — Buzzers, LEDs, Lasers, and a Relay

## Learning Objectives

- Drive every on/off actuator in the kit with the Volume 1 GPIO output driver, unmodified
- Explain sourcing vs sinking current and read a module schematic well enough to know which you are doing
- Understand what an *active* buzzer is (and why Chapter 17 exists for the passive one)
- Drive an inductive load (relay) safely and explain the flyback problem
- Build event→action programs combining Chapter 15 inputs with this chapter's outputs

## 16.1 You Already Wrote This Driver

`gpio_init_output`, `gpio_set`, `gpio_clear` from Volume 1 Chapter 8 drive everything here. This chapter adds no new registers — it adds *electrical judgement*: what is actually connected to the pin you are toggling, and how much current is flowing.

A GPIO pin on the ESP32-C3 can source or sink roughly 20 mA safely (40 mA absolute max — stay away from it). Every module in this chapter respects that budget because each carries its own resistor or transistor. The judgement matters the day you wire a bare LED with no resistor — the day you will be glad you can read the module schematics in Appendix C.

## 16.2 The Module Tour

| Module | What it is | Drive notes |
|--------|-----------|-------------|
| KY-012 active buzzer | Buzzer with built-in oscillator | `gpio_set` = tone, `gpio_clear` = silence. The module makes its own frequency — you supply only DC |
| KY-011 2-colour LED | Red + green, common cathode | Two GPIOs (7, 6). Both on = yellow-ish |
| KY-029 2-colour LED 3 mm | Same, smaller package | Identical driving |
| KY-034 auto-flash LED | LED with built-in flicker IC | One GPIO enables it; the colour dance is baked into the die — you cannot control it, only gate it |
| KY-008 laser | 650 nm laser diode + resistor | One GPIO. **Never look into it or point at anyone.** It is a toy-class diode, but the habit matters |
| KY-019 relay | Electromechanical relay, transistor-driven | VCC→**5V pin**, signal→GPIO7. See 16.3 |

The deep lesson of KY-012 vs the passive KY-006 (next chapter): the active buzzer hides an oscillator from you, exactly as `digitalWrite` hides a register from you. Volume 2's job is removing hiding places — Chapter 17 builds that oscillator in software, then in the LEDC peripheral.

## 16.3 The Relay — Your First Inductive Load

A relay is a coil that, when energised, physically pulls a switch contact closed. The KY-019's coil needs 5 V — power its VCC from the SuperMini 5V pin (the USB rail). Your 3.3 V GPIO drives the module's input transistor, which switches the coil; the onboard flyback diode absorbs the voltage spike the coil generates at turn-off (an inductor *insists* its current keep flowing; the diode gives that current a safe loop — without it, the spike would kill the transistor).

You will hear the click. That click is your software moving metal — the first time the fetch-decode-execute loop has exerted mechanical force.

**Safety is not negotiable: low-voltage loads only.** Switch an LED strip, a fan, a second buzzer. The relay's contacts are *rated* for mains; this course is not. Mains wiring is a licensed-electrician domain in Australia and a lethal one everywhere.

## 16.4 Patterns: Combining Input and Output

Three canonical structures, each one exercise:

**Latch** — momentary input, persistent output. Press toggles the laser on/off (press, not hold — your Chapter 15 edge detector is the engine).

**Follow** — output mirrors input continuously. Tilt switch drives the 2-colour LED: green level, red tilted.

**One-shot** — event triggers a timed action. Knock fires the relay for exactly 500 ms (your Ch15 dead-time + Ch10 delay).

## 16.5 Exercises

**Exercise 16.1: Toggle latch.** KY-004 button toggles KY-008 laser. Edge-detected, debounced, one press = one toggle, hold ≠ repeat.

Sample solution core:

```asm
# laser_latch.S
        .global asm_main
asm_main:
        addi  sp, sp, -16
        sw    ra, 12(sp)
        li    a0, 10                     # button on GPIO10
        jal   ra, gpio_init_input_pullup
        li    a0, 7                      # laser on GPIO7
        jal   ra, gpio_init_output
        li    s0, 1                      # prev level
        li    s2, 0                      # latch state
loop:
        li    a0, 10
        jal   ra, debounced_read
        beq   a0, s0, store              # no change
        bnez  a0, store                  # only act on 1->0
        xori  s2, s2, 1                  # flip latch
        li    a0, 7
        beqz  s2, off
        jal   ra, gpio_set
        j     store
off:    jal   ra, gpio_clear
store:  mv    s0, a0
        j     loop
```

**Exercise 16.2: Tilt traffic light.** KY-020 + KY-011: green when level, red when tilted, both (amber) during the 1-second grace period after a tilt is first detected.

**Exercise 16.3: Knock-activated relay.** One knock (KY-031, dead-time filter) energises the relay for 500 ms. Drive something visible with the contacts — the KY-034 auto-flash LED powered through them is satisfyingly meta: software → transistor → coil → contacts → a different chip's autonomous software.

**Exercise 16.4: Morse beacon.** The active buzzer (KY-012) sends your initials in Morse, correctly timed: dit = 100 ms, dah = 300 ms, intra-letter gap = 100 ms, inter-letter = 300 ms. Encode the pattern as bytes in `.rodata` (Volume 1, Ch6) and walk it with a pointer — data-driven output, the shape of every protocol transmitter you build in Phase 8.

## 16.6 Git Checkpoint

```bash
cd ~/esp32c3-assembly-course
git add ch16/
git commit -m "feat(ch16): digital outputs — latch, follow, one-shot patterns + relay"

# - [ ] Exercise 16.1: clean single-press toggling verified
# - [ ] Exercise 16.3: relay one-shot fires once per knock
# - [ ] Exercise 16.4: Morse timing verified by ear/recording

git tag -a ch16-complete -m "Chapter 16: Digital outputs complete"
git push --follow-tags        # if using a remote
```

## 16.7 Chapter Summary

- Every on/off actuator is the Volume 1 output driver plus electrical context: current budget, sourcing vs sinking, what the module's own transistor/resistor is doing
- Active buzzer = oscillator included; the relay = 5 V coil, 3.3 V control, flyback diode mandatory, low-voltage loads only
- Latch / follow / one-shot are the three atoms of input→output programs; everything in the capstone is built from them

**Check your understanding:**
1. Why does the relay module need the 5V pin while every other module in the kit gets 3V3 — and why is its *signal* pin still fine at 3.3 V?
2. What physically generates the voltage spike when a relay coil turns off, and what component absorbs it?
3. In the Morse exercise, why is encoding the message as data in `.rodata` better than a chain of hand-written set/clear/delay calls?

## 16.8 References for This Chapter

- ESP32-C3 Datasheet — DC characteristics (GPIO current limits): [https://www.espressif.com/sites/default/files/documentation/esp32-c3_datasheet_en.pdf](https://www.espressif.com/sites/default/files/documentation/esp32-c3_datasheet_en.pdf)
- Relay and flyback fundamentals (community reference): [https://arduinomodules.info/ky-019-5v-relay-module/](https://arduinomodules.info/ky-019-5v-relay-module/)
- International Morse code timing — ITU-R M.1677-1: [https://www.itu.int/rec/R-REC-M.1677-1-200910-I/](https://www.itu.int/rec/R-REC-M.1677-1-200910-I/)

---

# Chapter 17: PWM From First Principles — Tone, Colour, and the LEDC Peripheral

## Learning Objectives

- Explain pulse-width modulation as time-division of a constant supply — no analog hand-waving
- Bit-bang a square wave of any audible frequency and play a melody on the passive buzzer
- Bit-bang duty-cycle control and dim an LED, then articulate exactly why bit-banging does not scale
- Initialise the LEDC hardware peripheral from raw registers: timer, channel, GPIO matrix routing
- Mix colours on the RGB modules with three hardware PWM channels and zero CPU load

## 17.1 What PWM Actually Is

Your GPIO pin produces exactly two voltages: 0 V and 3.3 V. There is no knob. Yet you need "55% brightness" and "440 Hz tone". The trick is the oldest one in electronics: switch fast and let physics average.

Two numbers define a PWM signal:

- **Frequency** — how many on/off cycles per second. For a buzzer this *is* the pitch. For an LED it just needs to beat your eye (>200 Hz and flicker vanishes).
- **Duty cycle** — the percentage of each cycle spent high. For an LED this *is* the brightness. For a buzzer at fixed frequency, it mildly affects timbre and loudness; 50% is the convention.

The passive buzzer (KY-006) versus last chapter's active one is the whole lesson in two parts: KY-012 contains an oscillator; KY-006 is a bare piezo disc — *you* are the oscillator now.

## 17.2 Bit-Banged Tone

A square wave at frequency *f* means: high for half a period, low for half a period, forever. Half a period in cycles at 160 MHz:

```
half_cycles = 160,000,000 / (2 × f)
A440  →  160e6 / 880  =  181,818 cycles
```

```asm
# tone.S — blocking square-wave generator on GPIO7
# tone_play(int half_cycles, int periods)
# a0 = half-period in CPU cycles, a1 = number of full periods to emit
# clobbers t*, s0, s1 (saved)
        .global tone_play
tone_play:
        addi  sp, sp, -16
        sw    ra, 12(sp)
        sw    s0, 8(sp)
        sw    s1, 4(sp)
        mv    s0, a0                     # half period
        mv    s1, a1                     # period count
1:      li    a0, 7
        jal   ra, gpio_set
        mv    a0, s0
        jal   ra, delay_cycles
        li    a0, 7
        jal   ra, gpio_clear
        mv    a0, s0
        jal   ra, delay_cycles
        addi  s1, s1, -1
        bnez  s1, 1b
        lw    s1, 4(sp)
        lw    s0, 8(sp)
        lw    ra, 12(sp)
        addi  sp, sp, 16
        ret
```

Build a note table in `.rodata` — half-cycle counts for one octave — and a melody as (note index, duration) byte pairs, and Exercise 17.1 plays it. This is the Morse exercise's data-driven pattern, upgraded with pitch.

Equal-tempered octave starting at A440, as half-cycle constants:

```asm
        .section .rodata
        .align  2
note_table:                              # half-period cycles @160MHz
        .word 181818                     # A4  440.0 Hz
        .word 171603                     # A#4 466.2
        .word 161972                     # B4  493.9
        .word 152881                     # C5  523.3
        .word 144300                     # C#5 554.4
        .word 136201                     # D5  587.3
        .word 128556                     # D#5 622.3
        .word 121341                     # E5  659.3
        .word 114531                     # F5  698.5
        .word 108103                     # F#5 740.0
        .word 102036                     # G5  784.0
        .word  96308                     # G#5 830.6
        .word  90909                     # A5  880.0
```

## 17.3 Why Bit-Banging Does Not Scale

The tone generator owns the CPU completely. While the note plays, you cannot poll a button, cannot read a sensor, cannot even blink the status LED — the processor is a metronome and nothing else. Dim an LED *and* play a tone? Two interleaved timing loops, and the arithmetic of merging them is already painful. Add the RGB module — three independent duty cycles — and it is hopeless.

This is the recurring embedded trade and the reason peripherals exist: *any rigid timing job should be pushed into hardware so the CPU can think.* Volume 1 made the same move when the UART peripheral replaced any thought of bit-banging serial. Meet LEDC.

## 17.4 The LEDC Peripheral

LEDC ("LED Control") is the ESP32-C3's PWM engine: 4 timers and 6 channels, each channel an autonomous square-wave generator. You configure registers; it runs forever, duty-exact, with the CPU asleep or busy elsewhere.

Three layers to configure, in order:

```
1. CLOCK  — give the LEDC block a clock and enable the peripheral
2. TIMER  — set frequency: a counter of chosen bit-width + a clock divider
3. CHANNEL — set duty: compare value against the timer; route to a GPIO
```

Base address `0x60019000` (TRM, LEDC chapter). The registers used here:

| Register | Offset | Role |
|----------|--------|------|
| `LEDC_CH0_CONF0_REG` | 0x0000 | Channel 0: timer select, output enable |
| `LEDC_CH0_HPOINT_REG` | 0x0004 | Count at which output goes high (use 0) |
| `LEDC_CH0_DUTY_REG` | 0x0008 | Count at which output goes low — the duty, ×16 |
| `LEDC_CH0_CONF1_REG` | 0x000C | `DUTY_START` bit latches a new duty |
| (channels 1–5) | +0x14 each | Identical block per channel |
| `LEDC_TIMER0_CONF_REG` | 0x00A0 | Bit width, clock divider, pause/reset |
| (timers 1–3) | +0x08 each | Identical block per timer |
| `LEDC_CONF_REG` | 0x00D0 | Global clock select (APB = 80 MHz) |

And the frequency law that makes the divider intelligible:

```
f_pwm = f_clk / ( divider × 2^bits )

divider is 18.8 fixed-point (value = real_divider × 256)

LED dimming:   bits=13, divider=2.0 (512)   → 80e6/(2×8192)   ≈ 4.88 kHz
IR carrier:    bits=10, divider≈2.056 (526) → 80e6/(2.056×1024) ≈ 38.0 kHz  (Chapter 23)
```

> **Verify against the silicon.** Peripheral clock gating for LEDC lives in the SYSTEM registers (`SYSTEM_PERIP_CLK_EN0_REG`, bit `LEDC_CLK_EN`; the matching reset bit must be cleared) — TRM "System and Memory" chapter. As always in this book: the TRM is the ground truth and the Volume 1 Appendix H debugger is how you check it — `>x/1xw 0x600190A0` shows you your timer config as the hardware sees it. If a register read disagrees with what you wrote, you have found either a write-only field or your own bug; both are worth knowing.

The driver, `ledc.S`:

```asm
# ledc.S — minimal LEDC driver: one timer, channel n on pin p
        .equ  LEDC_BASE,        0x60019000
        .equ  LEDC_TIMER0_CONF, 0x00A0
        .equ  LEDC_CONF,        0x00D0
        .equ  CH_STRIDE,        0x14

        .equ  GPIO_BASE,        0x60004000
        .equ  GPIO_FUNC_OUT_SEL_CFG, 0x0554   # + 4*pin
        .equ  LEDC_LS_SIG_OUT0, 45            # GPIO matrix signal index (TRM)

        .text

# ledc_timer_init(int bits, int div_fixed)
# a0 = duty resolution in bits (1..14), a1 = divider, 18.8 fixed point
# Uses timer 0, APB clock. clobbers t0,t1
        .global ledc_timer_init
ledc_timer_init:
        li    t0, LEDC_BASE
        li    t1, 1                      # APB clock select
        sw    t1, LEDC_CONF(t0)
        slli  t1, a1, 4                  # divider field: bits 21:4 (sub-bits 11:4? — verify TRM)
        or    t1, t1, a0                 # resolution: bits 3:0
        sw    t1, LEDC_TIMER0_CONF(t0)
        # release reset / un-pause: bit 23 (param update) — TRM
        li    t1, (1 << 25)
        lw    a1, LEDC_TIMER0_CONF(t0)
        or    t1, t1, a1
        sw    t1, LEDC_TIMER0_CONF(t0)   # latch parameters
        ret

# ledc_channel_init(int ch, int pin)
# a0 = channel 0..5, a1 = GPIO pin. Timer 0, output enabled, duty 0.
        .global ledc_channel_init
ledc_channel_init:
        li    t0, LEDC_BASE
        li    t1, CH_STRIDE
        mul   t1, t1, a0
        add   t0, t0, t1                 # channel block base
        li    t2, (1 << 2)               # SIG_OUT_EN, timer sel = 0
        sw    t2, 0x0000(t0)             # CONF0
        sw    zero, 0x0004(t0)           # HPOINT = 0
        sw    zero, 0x0008(t0)           # DUTY = 0

        # Route LEDC signal to the pin via GPIO matrix
        li    t1, GPIO_BASE
        slli  t2, a1, 2
        add   t1, t1, t2
        li    t2, LEDC_LS_SIG_OUT0
        add   t2, t2, a0                 # signal index for this channel
        sw    t2, GPIO_FUNC_OUT_SEL_CFG(t1)
        # pin must also be an enabled output: caller runs gpio_init_output(pin)
        ret

# ledc_set_duty(int ch, int duty)
# a0 = channel, a1 = duty in timer counts (0 .. 2^bits-1)
        .global ledc_set_duty
ledc_set_duty:
        li    t0, LEDC_BASE
        li    t1, CH_STRIDE
        mul   t1, t1, a0
        add   t0, t0, t1
        slli  t1, a1, 4                  # DUTY register holds duty << 4
        sw    t1, 0x0008(t0)
        li    t1, (1 << 31)              # DUTY_START
        lw    a1, 0x000C(t0)
        or    t1, t1, a1
        sw    t1, 0x000C(t0)             # CONF1: latch
        ret
```

The exercise of *making this run* — clock-gate bit, parameter-latch bits, the GPIO matrix — is deliberately left with rough edges marked `verify TRM`. Volume 1 taught you to read the TRM and gave you a debugger; a driver you have personally verified field-by-field is worth ten you copied. Expect one debugging session; Exercise 17.3 walks the checklist.

## 17.5 Colour Mixing

KY-016 (through-hole RGB, built-in resistors) or KY-009 (SMD — *needs* external resistors, ~150 Ω, see Appendix C): red→GPIO7, green→GPIO6, blue→GPIO5, common cathode→GND. Three `ledc_channel_init` calls, three duties, any colour:

```asm
# rgb_set(int r, int g, int b) — 0..255 each, 13-bit timer assumed
# scale 8-bit colour to 13-bit duty: duty = c << 5
```

Perceptual note worth a comment in your code: LED brightness is wildly non-linear to duty. 50% duty looks ~80% bright. Exercise 17.4's gamma table (`duty = (c/255)² × 8191`, precomputed in `.rodata`) is the difference between a colour fade that looks professional and one that lurches.

The magic light cups (KY-027) are this chapter's toy: each module is a mercury tilt switch plus an LED. Two modules, two boards' worth of theatre on one: tilt input from each cup crossfades PWM brightness between the two LEDs — "pouring" light from cup to cup. Pure Chapter 15 input + Chapter 17 output.

## 17.6 Exercises

**Exercise 17.1: Melody.** Bit-banged. Encode "Waltzing Matilda" (or anything ≥16 notes) as `.rodata` (note index, duration-in-tenths) pairs; play it on KY-006. Rest = index 0xFF.

**Exercise 17.2: Breathing LED, bit-banged.** 1 kHz frame, duty swept 0→100→0% over 2 s, on the onboard LED. Feel the CPU cost: add a button poll to the same loop and watch the brightness stutter — that experience is the motivation for 17.4.

**Exercise 17.3: Bring up LEDC.** Drive the onboard LED at 4.88 kHz / 25% duty from `ledc.S`. Verification checklist: (1) clock-gate bit readable as 1; (2) timer conf reads back your value; (3) `>x/1xw` on the GPIO matrix slot shows 45; (4) LED visibly dimmer than full-on. Debug with Appendix H scenario 4 (MMIO verification).

**Exercise 17.4: RGB mood lamp.** Hardware PWM ×3 + gamma table. Smooth 10-second hue rotation — *while* the main loop polls the BOOT button to step between three brightness levels. The button responds instantly during the fade: that is the entire argument for hardware peripherals, felt in your thumb.

**Exercise 17.5: Magic light cups.** Two KY-027 modules; tilting one cup fades its LED down as the other fades up, rate proportional to how long the tilt is held.

## 17.7 Git Checkpoint

```bash
cd ~/esp32c3-assembly-course
git add ch17/
git commit -m "feat(ch17): bit-banged tone+dimming, LEDC driver, RGB mixing"

# - [ ] Exercise 17.1: melody recognisable
# - [ ] Exercise 17.3: all four LEDC verification checks pass
# - [ ] Exercise 17.4: button responsive mid-fade (hardware PWM proof)

git tag -a ch17-complete -m "Chapter 17: PWM from first principles + LEDC"
git push --follow-tags        # if using a remote
```

## 17.8 Chapter Summary

- PWM is time-division: frequency is pitch (buzzer) or invisibility (LED); duty is energy
- Bit-banging works and teaches, but owns the CPU; one rigid timing job per processor is the hard limit
- LEDC = timer (frequency) + channel (duty) + GPIO matrix (routing); configure once, runs forever
- The GPIO matrix signal-routing write is the new concept: any peripheral output to any pin — remember it, Chapter 23 routes a 38 kHz carrier with it

**Check your understanding:**
1. Why does changing duty cycle change an LED's brightness but not (much) a buzzer's pitch?
2. At 13-bit resolution from an 80 MHz clock with divider 2.0, show the arithmetic giving ≈4.88 kHz.
3. Exercise 17.4's button stayed responsive during the fade, where 17.2's stuttered. State precisely what work moved off the CPU, and into what.

## 17.9 References for This Chapter

- ESP32-C3 TRM — LED PWM Controller (LEDC) chapter: [https://www.espressif.com/sites/default/files/documentation/esp32-c3_technical_reference_manual_en.pdf](https://www.espressif.com/sites/default/files/documentation/esp32-c3_technical_reference_manual_en.pdf)
- ESP32-C3 TRM — GPIO Matrix (peripheral signal routing tables): same document
- Equal temperament note frequencies: [https://en.wikipedia.org/wiki/Piano_key_frequencies](https://en.wikipedia.org/wiki/Piano_key_frequencies)

---

# PHASE 7: THE ANALOG WORLD
## Weeks 4 – 5 | Continuous Reality, Quantised

---

# Chapter 18: The SAR ADC — Reading Continuous Reality

## Learning Objectives

- Explain successive-approximation conversion well enough to implement it on paper
- Bring up the ESP32-C3 SAR ADC for one-shot reads from raw registers
- Read all eight of the kit's analog sensors and interpret their curves
- Convert thermistor readings to degrees with integer-only arithmetic
- Know the C3 ADC's real-world limits: range, attenuation, noise, and the ADC2 trap

## 18.1 From Volts to Bits

Every sensor in this chapter outputs a *voltage* somewhere between 0 and 3.3 V — continuous, infinitely divisible, and invisible to a CPU that only knows words of bits. The Analog-to-Digital Converter is the bridge: it samples the voltage and returns an integer, here 12 bits, 0–4095.

The C3's converter is a **SAR** — successive approximation register — and the algorithm is one you already know: binary search. The hardware guesses half-scale, compares against the input with an analog comparator, keeps or discards the bit, then guesses the next bit down. Twelve comparisons, twelve bits, done. It is bisection implemented in capacitors, and it is worth admiring: the same divide-and-conquer you would write in software, running in silicon at megasamples per second.

```
target = 2.1 V of 3.3 V full-scale
bit 11: try 1.65 V  — 2.1 > 1.65  → 1    (1???????????)
bit 10: try 2.475 V — 2.1 < 2.475 → 0    (10??????????)
bit  9: try 2.0625  — 2.1 > 2.06  → 1    (101?????????)
... nine more ...                        → 101000101110  = 2606
```

## 18.2 The C3's ADC: What the Datasheet Actually Says

Facts you need before trusting a single reading:

- **ADC1 has five channels: GPIO0–GPIO4 = channels 0–4.** These are your analog pins for the rest of the book.
- **ADC2 (GPIO5) is a trap.** It is shared with the WiFi radio and Espressif's own documentation deprecates its use. This book never touches it. The kit's needs fit in five channels.
- **Attenuation sets the input range.** The raw ADC sees ~0–1.1 V (0 dB). An input attenuator extends this; at **11 dB**, the *usable, characterised* range is roughly 0.05–2.5 V. Above ~2.5 V the reading saturates and flattens long before 3.3 V. This volume uses 11 dB everywhere, and the sensor interpretations account for the ceiling.
- **It is noisy.** ±10–20 counts of jitter on a steady input is normal. Averaging 16 samples is the standing fix (Exercise 18.2 quantifies it).

## 18.3 Bringing Up the ADC

The one-shot ("single conversion on demand") sequence, register by register. The SAR ADC lives in the `APB_SARADC` block at base `0x60040000` (TRM, On-Chip Sensor chapter):

```
1. Enable the peripheral clock: SYSTEM_PERIP_CLK_EN0_REG, APB_SARADC_CLK_EN bit;
   clear the matching reset bit in SYSTEM_PERIP_RST_EN0_REG
2. Configure ONETIME_SAMPLE: channel number, attenuation (3 = 11 dB),
   and the SAR1 one-time enable bit
3. Pulse ONETIME_START: 0 → 1 starts a conversion
4. Poll the ADC1_DONE interrupt-raw bit
5. Read the 12-bit result from the SAR1 data register; clear the done flag
```

```asm
# adc.S — one-shot SAR ADC reads, channels 0..4 (GPIO0..GPIO4), 11 dB
        .equ  SARADC_BASE,      0x60040000
        # Register offsets below are from ESP32-C3 TRM "On-Chip Sensor and
        # Analog Signal Processing", register summary table. As with LEDC:
        # verify each against your TRM revision with the Appendix H debugger
        # before trusting the driver — >x/1xw is your multimeter.
        .equ  ONETIME_SAMPLE,   0x0020   # channel | atten | onetime enables
        .equ  ONETIME_START_BIT,(1 << 29)
        .equ  SAR1_ENC_BIT,     (1 << 31)
        .equ  ATTEN_SHIFT,      23       # 2-bit attenuation field
        .equ  CHAN_SHIFT,       25       # 4-bit channel field
        .equ  INT_RAW,          0x0040   # ADC1_DONE raw status
        .equ  INT_CLR,          0x0048
        .equ  ADC1_DONE_BIT,    (1 << 31)
        .equ  SAR1_DATA,        0x002C   # 12-bit result in low bits

        .text

# adc_read(int channel) -> int   raw 12-bit, 11 dB attenuation
# a0 = channel 0..4; returns 0..4095 in a0. clobbers t0-t3
        .global adc_read
adc_read:
        li    t0, SARADC_BASE
        # channel + 11dB atten + SAR1 onetime enable, START still low
        slli  t1, a0, CHAN_SHIFT
        li    t2, 3                      # 11 dB
        slli  t2, t2, ATTEN_SHIFT
        or    t1, t1, t2
        li    t2, SAR1_ENC_BIT
        or    t1, t1, t2
        sw    t1, ONETIME_SAMPLE(t0)
        # rising edge on START
        li    t2, ONETIME_START_BIT
        or    t1, t1, t2
        sw    t1, ONETIME_SAMPLE(t0)
1:      lw    t2, INT_RAW(t0)            # poll DONE
        li    t3, ADC1_DONE_BIT
        and   t2, t2, t3
        beqz  t2, 1b
        lw    a0, SAR1_DATA(t0)
        li    t2, 0xFFF
        and   a0, a0, t2                 # 12-bit result
        li    t2, ADC1_DONE_BIT
        sw    t2, INT_CLR(t0)            # clear for next time
        ret

# adc_read_avg16(int channel) -> int  — 16-sample boxcar, the standard read
        .global adc_read_avg16
adc_read_avg16:
        addi  sp, sp, -16
        sw    ra, 12(sp)
        sw    s0, 8(sp)
        sw    s1, 4(sp)
        sw    s2, 0(sp)
        mv    s0, a0
        li    s1, 0                      # accumulator
        li    s2, 16
1:      mv    a0, s0
        jal   ra, adc_read
        add   s1, s1, a0
        addi  s2, s2, -1
        bnez  s2, 1b
        srli  a0, s1, 4                  # /16
        lw    s2, 0(sp)
        lw    s1, 4(sp)
        lw    s0, 8(sp)
        lw    ra, 12(sp)
        addi  sp, sp, 16
        ret
```

Don't forget the pad: an ADC pin must have its output driver disabled and, critically, **no pull resistors** — a 45 kΩ pull-up quietly drags every divider-type sensor reading toward full scale. Add `gpio_init_analog` to `gpio.S`: IOMUX `MCU_SEL=1`, everything else (IE, WPU, WPD) zero, output-enable cleared.

## 18.4 The Eight-Sensor Tour

All divider-type modules: signal → GPIO0 (channel 0), VCC → 3V3, GND → GND.

| Module | Physical quantity | Curve shape | Notes |
|--------|------------------|-------------|-------|
| KY-018 photoresistor | Light | More light → *lower* reading on most boards (LDR position in divider varies — measure, don't assume) | The book's reference analog sensor |
| KY-013 thermistor | Temperature | NTC: hotter → lower resistance | §18.5 converts to °C |
| KY-028 thermistor + LM393 | Temperature | Same analog curve; digital pin = trimpot threshold | Read both pins; compare |
| KY-024 linear hall | Magnetic field | Centred ~½ scale, swings *both* ways with pole | Sign of (reading − midpoint) = which pole |
| KY-035 analog hall | Magnetic field | Same idea, different IC | |
| KY-026 flame (analog pin) | IR around 940 nm | More flame → bigger swing | Lighter at arm's length; mind the trimpot story from Ch15 |
| KY-037 / KY-038 microphones | Sound pressure | AC wiggle around a midpoint | A single read is meaningless — §18.6 |
| KY-039 heartbeat | IR through fingertip | Tiny periodic wiggle | Deferred to Chapter 24 — it needs everything this phase teaches |

The hall modules teach the most underrated lesson: a *bipolar* sensor idles mid-scale, and the information is the signed deviation. `reading - 2048` is your first signed sensor value, and `blt`-vs-`bltu` from Volume 1 Chapter 4 suddenly has stakes.

## 18.5 Thermistor to Degrees, Integer-Only

The NTC thermistor's resistance-temperature law is exponential (the β equation). Floating point would be the lazy route; this book has no FPU and no appetite for one. The embedded answer is a **lookup table with interpolation** — precompute the curve, store ADC-count→deci-degree pairs in `.rodata`, and linearly interpolate between entries:

```asm
        .section .rodata
        .align 2
# (adc_count, temp_in_0.1°C) pairs, 10 kΩ NTC β=3950, 10 kΩ divider, 11 dB
# Generated once on your desktop (any language); 17 entries, 5°C steps
therm_table:
        .word 3431, 0      # 0.0 °C
        .word 3252, 50     # 5.0
        .word 3052, 100
        .word 2834, 150
        .word 2602, 200
        .word 2362, 250    # 25.0 °C — room temperature ≈ mid-table
        .word 2120, 300
        .word 1884, 350
        .word 1658, 400
        .word 1447, 450
        .word 1255, 500
        # ... extend to your range of interest
therm_table_end:
```

The search-and-interpolate function is Exercise 18.3 — binary thinking again: walk the table to bracket the reading, then `temp = t0 + (t1-t0) × (adc-a0) / (a1-a0)` in 32-bit integer math (`mul` then `div`, both M-extension, both on your chip — Volume 1 §2.4's promise finally paying rent).

## 18.6 Sampling a Waveform: the Microphones

Sound is change. The microphone modules output ~1.2 V of bias with millivolts of audio wiggling on top — one ADC read tells you the bias, which tells you nothing. Loudness is the *spread* of many fast samples:

```
take 256 samples as fast as adc_read allows
loudness = max(samples) - min(samples)        # peak-to-peak
```

Clap-detector logic: peak-to-peak above threshold = a clap. The two-clap lamp switch (Exercise 18.5) needs the Chapter 15 dead-time idea — in the sampled-signal domain this time. You are now doing digital signal processing, however humbly; Chapter 24 grows this exact loop into a heartbeat detector.

## 18.7 Exercises

**Exercise 18.1: Bring-up + light meter.** ADC on channel 0 with KY-018; stream `adc_read_avg16` to UART at 10 Hz. Record readings: room light, phone torch, covered. Then verify the driver the Appendix H way: with the debugger halted, `>x/1xw 0x6004002C` after a manual conversion.

**Exercise 18.2: Quantify the noise.** 100 raw reads of a steady input: report min/max/mean over UART. Repeat with `adc_read_avg16`. Commit both spreads — your personal noise floor, in numbers.

**Exercise 18.3: Thermometer.** Table interpolation per §18.5; print `23.4 C` style deci-degrees (your Volume 1 `uart_putdec` plus a decimal-point insertion). Calibrate the table's room-temperature entry against any household thermometer.

**Exercise 18.4: Magnetic polarity meter.** KY-024: onboard LED off in no field; RGB red for north-ish, blue for south-ish, brightness ∝ |reading − midpoint| via Chapter 17's LEDC. Three chapters, one program.

**Exercise 18.5: Clap-clap lamp.** KY-038 peak-to-peak detector toggles the relay (Ch16) on two claps within 800 ms but not one. Dead-time per clap: 150 ms.

## 18.8 Git Checkpoint

```bash
cd ~/esp32c3-assembly-course
git add ch18/
git commit -m "feat(ch18): SAR ADC one-shot driver, 8 analog sensors, thermometer"

# - [ ] Exercise 18.2: noise figures recorded raw vs averaged
# - [ ] Exercise 18.3: thermometer within ~1°C of reference at room temp
# - [ ] Exercise 18.5: two claps toggle, one clap doesn't

git tag -a ch18-complete -m "Chapter 18: SAR ADC and the analog sensor family"
git push --follow-tags        # if using a remote
```

## 18.9 Chapter Summary

- SAR conversion is binary search in silicon — 12 comparisons, 12 bits
- ADC1 = GPIO0–4 only; ADC2 is radio-shared and avoided; 11 dB attenuation buys ~0.05–2.5 V of trustworthy range
- Analog pads need IE/WPU/WPD all off — a forgotten pull-up is a systematic error, not noise
- Noise is real: average; bipolar sensors centre mid-scale: subtract and go signed; AC signals: sample fast and measure spread
- Integer lookup-plus-interpolation replaces floating point for any smooth sensor curve

**Check your understanding:**
1. Your thermistor reads a steady 4095 regardless of temperature. Name the two most likely configuration bugs, in the order you would check them.
2. Why does a single ADC read of a microphone tell you nothing about loudness, while min/max over 256 reads tells you plenty?
3. The hall sensor reads 2048 ± noise with no magnet. Write the two-line assembly that produces a signed field value, and name the Volume 1 chapter that explains why `blt` (not `bltu`) must compare it.

## 18.10 References for This Chapter

- ESP32-C3 TRM — On-Chip Sensor and Analog Signal Processing (SAR ADC): [https://www.espressif.com/sites/default/files/documentation/esp32-c3_technical_reference_manual_en.pdf](https://www.espressif.com/sites/default/files/documentation/esp32-c3_technical_reference_manual_en.pdf)
- ESP32-C3 ADC characteristics and attenuation ranges — ESP-IDF ADC documentation: [https://docs.espressif.com/projects/esp-idf/en/stable/esp32c3/api-reference/peripherals/adc_oneshot.html](https://docs.espressif.com/projects/esp-idf/en/stable/esp32c3/api-reference/peripherals/adc_oneshot.html)
- NTC thermistor β equation and table generation: [https://en.wikipedia.org/wiki/Thermistor](https://en.wikipedia.org/wiki/Thermistor)

---

# Chapter 19: The Joystick — Mixed-Signal Input

## Learning Objectives

- Read a two-axis analog joystick plus its push-button as one coherent input device
- Implement dead-zones, calibration, and direction quantisation in integer math
- Drive outputs from continuous input: cursor-style control of colour and pitch
- Structure a polling loop that mixes ADC reads, digital reads, and PWM updates cleanly

## 19.1 Three Sensors in a Thumb

The KY-023 joystick is two potentiometers (X, Y) and a push switch (press the stick). Wiring: VRx→GPIO0, VRy→GPIO1, SW→GPIO10 (pull-up — it shorts to ground), VCC→3V3.

At 3.3 V supply the pots swing the full ADC range, but reality intrudes in three ways your driver must absorb:

1. **Centre isn't 2048.** Springs are imperfect; expect ±150. Calibrate at startup: average 32 reads of each axis while the user leaves the stick alone; that is your (cx, cy).
2. **Centre wobbles.** A dead-zone of ±100 counts around centre reads as "no input", or the cursor drifts forever.
3. **The 11 dB ceiling.** Above ~2.5 V the ADC saturates — the top ~25% of physical travel reads as one value. For direction-style input this is harmless; for proportional control, map the *usable* span, not the theoretical one.

```asm
# joystick.S
# joy_read_axis(int ch, int centre) -> int   signed, dead-zoned
# a0 = ADC channel, a1 = calibrated centre; returns -1500..+1500-ish, 0 in dead-zone
        .global joy_read_axis
joy_read_axis:
        addi  sp, sp, -16
        sw    ra, 12(sp)
        sw    s0, 8(sp)
        mv    s0, a1
        jal   ra, adc_read_avg16
        sub   a0, a0, s0                 # signed offset from centre
        li    t0, 100                    # dead-zone
        blt   a0, t0, 1f
        addi  a0, a0, -100               # shift so range is continuous
        j     2f
1:      li    t0, -100
        bgt   a0, t0, 3f
        addi  a0, a0, 100
        j     2f
3:      li    a0, 0                      # inside dead-zone
2:      lw    s0, 8(sp)
        lw    ra, 12(sp)
        addi  sp, sp, 16
        ret
```

## 19.2 Quantising Direction

Continuous (x, y) to eight compass directions is sign-and-magnitude logic — pure Chapter 4 branching: compare |x| vs |y| to pick the dominant axis, then the sign picks the direction; diagonals when the magnitudes are within 2× of each other. No trigonometry was harmed.

## 19.3 Exercises

**Exercise 19.1: Calibrated readout.** Startup calibration, then stream `X:+0432 Y:-0011 SW:1` at 10 Hz over UART. Verify dead-zone (released stick prints zeros) and saturation (find where the count stops growing — write the number in a comment).

**Exercise 19.2: Colour stick.** X = hue around the RGB colour wheel, Y = brightness, press = freeze/unfreeze. Chapter 17's LEDC + gamma doing real work.

**Exercise 19.3: Theremin.** X bends pitch over one octave on the passive buzzer (note table interpolation), Y picks duty (timbre); press silences. Bit-banged tone is acceptable here — and notice *why* the stick still feels responsive: the tone loop is short per-note. Write the one-sentence explanation in your commit message.

**Exercise 19.4: Direction logger.** Eight-way quantiser; on every direction *change* (edge thinking, Ch15) print the compass name from a `.rodata` string table — your Volume 1 Chapter 6 string machinery, back on stage.

## 19.4 Git Checkpoint

```bash
cd ~/esp32c3-assembly-course
git add ch19/
git commit -m "feat(ch19): joystick driver — calibration, dead-zone, 8-way quantise"

# - [ ] Exercise 19.1: dead-zone and saturation values recorded
# - [ ] Exercise 19.2: full hue wheel reachable, freeze works
# - [ ] Exercise 19.4: direction edges only (no repeats while held)

git tag -a ch19-complete -m "Chapter 19: Mixed-signal joystick"
git push --follow-tags        # if using a remote
```

## 19.5 Chapter Summary

- A "joystick" is three primitive inputs read coherently: two ADC channels and a GPIO
- Real analog input demands calibration, dead-zones, and respect for the attenuation ceiling
- Direction quantisation is branching on signs and relative magnitude — control logic, not maths

**Check your understanding:**
1. Why calibrate centre at startup rather than hard-coding 2048, and what user instruction does that impose?
2. A dead-zone shifts the usable range. Why does the driver subtract the dead-zone width after the comparison instead of just zeroing small values?
3. In Exercise 19.3, the stick is read between notes, not during them. What melody-duration choice would break that responsiveness, and which chapter's peripheral would fix it?

## 19.6 References for This Chapter

- KY-023 module electrical reference: [https://arduinomodules.info/ky-023-joystick-dual-axis-module/](https://arduinomodules.info/ky-023-joystick-dual-axis-module/)
- ESP32-C3 ADC attenuation/characterised ranges (as Ch18): [https://docs.espressif.com/projects/esp-idf/en/stable/esp32c3/api-reference/peripherals/adc_oneshot.html](https://docs.espressif.com/projects/esp-idf/en/stable/esp32c3/api-reference/peripherals/adc_oneshot.html)

---

# PHASE 8: PROTOCOLS IN ASSEMBLY
## Weeks 6 – 9 | Where Time Itself Carries the Data

This phase is the heart of Volume 2. Each chapter implements a real, deployed, datasheet-specified serial protocol with nothing but a GPIO pin and the `mcycle` counter. Microseconds become the alphabet. Everything Volume 1 built — cycle counting, signed comparison, stack discipline, byte assembly from bits — converges here.

A shared utility first. Every protocol below thinks in microseconds, so promote Volume 1's cycle delay into `delay.S` permanently:

```asm
# delay.S — microsecond timing on a 160 MHz core
        .equ  CYCLES_PER_US, 160

# delay_us(int us) — blocking, mcycle-based, accurate to ~1 cycle
        .global delay_us
delay_us:
        li    t0, CYCLES_PER_US
        mul   t0, t0, a0                 # cycles to wait
        csrr  t1, mcycle                 # start
1:      csrr  t2, mcycle
        sub   t2, t2, t1
        bltu  t2, t0, 1b                 # unsigned: survives wraparound window
        ret

# elapsed_us(int start_cycles) -> int — µs since a captured mcycle
        .global elapsed_us
elapsed_us:
        csrr  t0, mcycle
        sub   t0, t0, a0
        li    t1, CYCLES_PER_US
        divu  a0, t0, t1
        ret
```

---

# Chapter 20: 1-Wire and the DS18B20 — Your First Real Protocol

## Learning Objectives

- Explain open-drain bus signalling and emulate it with output-enable toggling
- Implement the 1-Wire reset/presence handshake and read/write time slots to datasheet timing
- Drive the DS18B20 command sequence: convert, read scratchpad, verify
- Interpret the sensor's 16-bit two's-complement, 1/16-degree fixed-point format — felt as arithmetic, not trivia
- Debug a timing protocol using only `mcycle` and the UART

## 20.1 One Wire, Both Directions

The KY-001 carries a DS18B20: a thermometer, a unique 64-bit serial number, and a complete serial transceiver in a transistor package, all speaking over a *single shared wire*. Both sides can drive the bus — so neither may ever drive it high. The bus is **open-drain**: devices either pull the line low or release it, and a pull-up resistor (on the module: 4.7 kΩ) restores the high state. Two drivers can then never fight; the line is simply low if *anyone* pulls.

Your GPIO becomes open-drain by toggling the output *enable*, not the output *value*:

```asm
# onewire.S — open-drain primitives on OW_PIN
        .equ  OW_PIN,            10
        .equ  GPIO_BASE,         0x60004000
        .equ  GPIO_ENABLE_W1TS,  0x0024
        .equ  GPIO_ENABLE_W1TC,  0x0028
        .equ  GPIO_OUT_W1TC,     0x000C

# ow_low — drive the bus low (enable output, value already latched 0)
ow_low:
        li    t0, GPIO_BASE
        li    t1, (1 << OW_PIN)
        sw    t1, GPIO_OUT_W1TC(t0)      # output value = 0
        sw    t1, GPIO_ENABLE_W1TS(t0)   # driver ON  → bus low
        ret

# ow_release — stop driving; pull-up takes the bus high
ow_release:
        li    t0, GPIO_BASE
        li    t1, (1 << OW_PIN)
        sw    t1, GPIO_ENABLE_W1TC(t0)   # driver OFF → bus floats high
        ret
```

(`gpio_init_input_pullup(OW_PIN)` at startup arms `FUN_IE` so `gpio_read` works throughout; the module's 4.7 kΩ does the real pulling — the internal weak pull-up merely agrees.)

## 20.2 The Timing Alphabet

All 1-Wire communication is built from three primitive gestures, each defined by *how long* the master holds the line low (datasheet figures, standard speed):

```
RESET     master low ≥480 µs, release, wait 70 µs, READ:
          bus low  = a device is present (it answers with its own 60–240 µs pull)
          bus high = nobody home

WRITE 1   low 6 µs,  release, wait out the rest of a 70 µs slot
WRITE 0   low 60 µs, release, 10 µs recovery

READ      low 6 µs, release, sample the bus at ~15 µs from slot start,
          wait out the rest of 70 µs
          (the *device* holds the line low to send a 0, releases for a 1)
```

The discipline: every slot is the master's to start, every microsecond is specified, and your `delay_us` is now load-bearing. Bit functions:

```asm
# ow_write_bit(int bit)        a0 = 0/1
ow_write_bit:
        addi  sp, sp, -16
        sw    ra, 12(sp)
        sw    s0, 8(sp)
        mv    s0, a0
        jal   ra, ow_low
        bnez  s0, 1f
        li    a0, 60                     # write-0: long low
        jal   ra, delay_us
        jal   ra, ow_release
        li    a0, 10
        jal   ra, delay_us
        j     2f
1:      li    a0, 6                      # write-1: short low
        jal   ra, delay_us
        jal   ra, ow_release
        li    a0, 64
        jal   ra, delay_us
2:      lw    s0, 8(sp)
        lw    ra, 12(sp)
        addi  sp, sp, 16
        ret

# ow_read_bit() -> int
ow_read_bit:
        addi  sp, sp, -16
        sw    ra, 12(sp)
        sw    s0, 8(sp)
        jal   ra, ow_low
        li    a0, 3
        jal   ra, delay_us
        jal   ra, ow_release
        li    a0, 10                     # sample inside the 15 µs window
        jal   ra, delay_us
        li    a0, OW_PIN
        jal   ra, gpio_read
        mv    s0, a0
        li    a0, 55                     # complete the slot
        jal   ra, delay_us
        mv    a0, s0
        lw    s0, 8(sp)
        lw    ra, 12(sp)
        addi  sp, sp, 16
        ret
```

Bytes are bits, LSB first — a `slli`/`or` accumulation loop you can now write from memory (`ow_write_byte`, `ow_read_byte`: Exercise 20.1, ten lines each).

## 20.3 The Conversation

The DS18B20 command sequence for one temperature (single device on the bus, so the ROM-addressing step collapses to "Skip ROM"):

```
1. RESET, expect presence
2. write 0xCC        Skip ROM (talk to the only device)
3. write 0x44        Convert T — sensor starts measuring
4. wait ≤750 ms      (12-bit conversion time; or poll: read slots return 0 while busy)
5. RESET, presence again
6. write 0xCC        Skip ROM
7. write 0xBE        Read Scratchpad
8. read 2 bytes      temperature LSB, MSB   (7 more follow; stop after 2 or read all 9 for CRC)
```

## 20.4 Sixteenths of a Degree

The two bytes form a 16-bit two's-complement value in units of **1/16 °C**:

```
0x0191 = 401   → 401/16  = 25.0625 °C
0xFF5E = -162  → -162/16 = -10.125 °C
```

This is fixed-point arithmetic meeting the real world. To print `25.06`:

```asm
# a0 = raw 16-bit reading (sign-extended to 32 bits first!)
        slli  a0, a0, 16
        srai  a0, a0, 16                 # sign-extend — srai, not srli (Vol 1 §2.3 pays off)
        # integer part: arithmetic shift right 4 (rounds toward -∞ — correct for temps)
        srai  t0, a0, 4
        # fraction: low 4 bits × 625 = ten-thousandths (1/16 = 0.0625)
        andi  t1, a0, 0xF
        li    t2, 625
        mul   t1, t1, t2                 # 0..9375
```

That `srai`-not-`srli` line is why Volume 1 made you answer a check-question about arithmetic shifts. Negative temperatures are where copy-paste drivers die.

## 20.5 Exercises

**Exercise 20.1: Byte layer.** `ow_write_byte` / `ow_read_byte`, LSB-first. Test: RESET + presence detection prints `DS18B20 present` / `no device` — your first conversation with a remote chip.

**Exercise 20.2: Thermometer.** Full sequence; print `25.06 C` at 1 Hz. Compare against the Chapter 18 thermistor on the same desk — the digital sensor's stability versus your analog table is the point of the exercise. Record both for 60 s and commit the comparison.

**Exercise 20.3: Conversion-time measurement.** Replace the 750 ms wait with polling (read slots return 0 while converting, 1 when done) and time the actual conversion with `mcycle`. Datasheet says ≤750 ms — what does *your* die deliver? You own a measurement no datasheet gives you.

**Exercise 20.4: Read the serial number.** Command 0x33 (Read ROM) returns the 64-bit unique ID. Print it as hex. Every DS18B20 ever made has a different one — you are reading a number engraved at the factory, over one wire, with code you wrote from a timing diagram.

**Exercise 20.5 (stretch): CRC-8.** The scratchpad's 9th byte is a CRC (polynomial 0x31 reflected → the classic Dallas 0x8C table-less loop). Implement the bitwise check and reject corrupted reads. Eight `srli`/`xori` lines that make your driver production-honest.

## 20.6 Git Checkpoint

```bash
cd ~/esp32c3-assembly-course
git add ch20/
git commit -m "feat(ch20): 1-Wire bit/byte layer + DS18B20 thermometer"

# - [ ] Exercise 20.2: stable readings, sane vs thermistor
# - [ ] Exercise 20.3: measured conversion time recorded
# - [ ] Exercise 20.4: 64-bit ROM printed

git tag -a ch20-complete -m "Chapter 20: 1-Wire DS18B20 from the timing diagram up"
git push --follow-tags        # if using a remote
```

## 20.7 Chapter Summary

- Open-drain = drive low or let go; emulated with output-enable, never output-value
- 1-Wire's entire vocabulary is three timed gestures; `delay_us` is the protocol engine
- Reset/presence is a handshake you can *see* with one `gpio_read`
- 1/16-degree two's-complement is fixed-point in the wild; `srai` vs `srli` stops being a quiz question the first time you read a freezer

**Check your understanding:**
1. Why must the master never actively drive the bus high, and what hardware element makes "high" happen at all?
2. In a read slot, who is holding the line low when you sample a 0 — and what was the master's only contribution to that slot?
3. `0xFF5E` printed as `4085.8 C` instead of `-10.1 C`. Name the exact missing instruction(s).

## 20.8 References for This Chapter

- DS18B20 datasheet (Analog Devices/Maxim) — timing figures and command set: [https://www.analog.com/media/en/technical-documentation/data-sheets/DS18B20.pdf](https://www.analog.com/media/en/technical-documentation/data-sheets/DS18B20.pdf)
- 1-Wire protocol overview (AN937): [https://www.analog.com/en/resources/app-notes/1wire-communication-through-software.html](https://www.analog.com/en/resources/app-notes/1wire-communication-through-software.html)

---

# Chapter 21: The DHT11 — Reading Forty Bits by Stopwatch

## Learning Objectives

- Implement a receive-only timing protocol where bit values live in *pulse widths*
- Measure pulse widths with `mcycle` while the pulses are happening — no second chances
- Assemble 40 bits into 5 bytes on the stack and verify a checksum
- Handle a protocol's failure modes honestly: timeout, checksum fail, sensor sulking

## 21.1 A Different Kind of Single Wire

The KY-015's DHT11 measures humidity and temperature, and speaks a one-wire protocol that is *not* 1-Wire™ — simpler, slower, and with a twist: after you ask, the sensor takes over the line completely and streams 40 bits at its own pace. You don't clock anything. You *watch* and *time*.

```
MASTER:  pull low ≥18 ms (the wake-up), release, wait
SENSOR:  80 µs low + 80 µs high   (its "I'm here")
THEN 40 BITS, each:
         50 µs low                (always — the inter-bit gap)
         26–28 µs high  = 0
         ~70 µs high    = 1
```

The bit is the *duration of the high*. Your job per bit: wait for the rising edge, capture `mcycle`, wait for the falling edge, capture again, subtract, compare against a 50 µs threshold. Forty times, back to back, with no pause allowed — at 160 MHz you have ~4,000 cycles of slack per bit, which is luxurious *if* your loop is tight and fatal if you wander off to print something.

## 21.2 The Receiver

```asm
# dht11.S — read humidity + temperature from KY-015 on DHT_PIN
        .equ  DHT_PIN, 10
        .equ  TIMEOUT_US, 200            # any single wait beyond this = abort

# dht11_read(uint8_t buf[5]) -> int      0 = ok, -1 = timeout, -2 = checksum
# a0 = pointer to 5-byte buffer (caller's stack)
        .global dht11_read
dht11_read:
        addi  sp, sp, -32
        sw    ra, 28(sp)
        sw    s0, 24(sp)                 # buf pointer
        sw    s1, 20(sp)                 # bit counter / scratch
        sw    s2, 16(sp)                 # current byte accumulator
        sw    s3, 12(sp)                 # byte index
        mv    s0, a0

        # ── wake-up: ≥18 ms low, then release ──
        jal   ra, ow_low                 # reuse Ch20 primitives (same pin)
        li    a0, 18000
        jal   ra, delay_us
        jal   ra, ow_release

        # ── sensor preamble: low phase then high phase ──
        jal   ra, wait_low               # returns -1 on timeout in a0
        bltz  a0, fail_timeout
        jal   ra, wait_high
        bltz  a0, fail_timeout
        jal   ra, wait_low               # end of 80 µs high → first bit's gap
        bltz  a0, fail_timeout

        # ── 40 bits ──
        li    s3, 0                      # byte index 0..4
next_byte:
        li    s2, 0                      # accumulator
        li    s1, 8                      # bits remaining in byte
next_bit:
        jal   ra, wait_high              # rising edge = start of data pulse
        bltz  a0, fail_timeout
        csrr  t3, mcycle                 # ── stopwatch start (no calls now!) ──
1:      lw    t0, GPIO_IN_OFF + GPIO_BASE_LO(zero)  # see note below
        # — in practice: inline read of GPIO_IN, pin test, loop —
        # shown expanded in the repository version; kept inline for speed
        srl   t0, t0, DHT_PIN
        andi  t0, t0, 1
        bnez  t0, 1b                     # still high: keep timing
        csrr  t4, mcycle                 # ── stopwatch stop ──
        sub   t4, t4, t3
        li    t0, 160 * 50               # 50 µs threshold in cycles
        sltu  t0, t0, t4                 # t0 = 1 if pulse > 50 µs → bit is 1
        slli  s2, s2, 1
        or    s2, s2, t0                 # MSB first (DHT sends MSB first)
        addi  s1, s1, -1
        bnez  s1, next_bit
        add   t0, s0, s3
        sb    s2, 0(t0)                  # store completed byte
        addi  s3, s3, 1
        li    t0, 5
        blt   s3, t0, next_byte

        # ── checksum: byte4 == (byte0+byte1+byte2+byte3) & 0xFF ──
        lbu   t0, 0(s0)
        lbu   t1, 1(s0)
        add   t0, t0, t1
        lbu   t1, 2(s0)
        add   t0, t0, t1
        lbu   t1, 3(s0)
        add   t0, t0, t1
        andi  t0, t0, 0xFF
        lbu   t1, 4(s0)
        bne   t0, t1, fail_csum
        li    a0, 0
        j     done
fail_timeout:
        li    a0, -1
        j     done
fail_csum:
        li    a0, -2
done:
        lw    s3, 12(sp)
        lw    s2, 16(sp)
        lw    s1, 20(sp)
        lw    s0, 24(sp)
        lw    ra, 28(sp)
        addi  sp, sp, 32
        ret
```

(`wait_low` / `wait_high` are six-line helpers: poll `gpio_read` with an `elapsed_us` timeout — Exercise 21.1.) Note the comment at the inner stopwatch loop: *no function calls inside the timed region.* A `jal` to `gpio_read` costs call overhead per poll and coarsens your measurement; the inner loop inlines the `GPIO_IN` read. This is the chapter where you first choose speed over structure on purpose, and say so in a comment.

Buffer layout, per datasheet: `buf[0]` = humidity integer, `buf[1]` = humidity decimal (always 0 on DHT11), `buf[2]` = temperature integer, `buf[3]` = temperature decimal, `buf[4]` = checksum. Print `H:47%  T:23C`.

Two operational facts that masquerade as bugs: the DHT11 needs **1 second of power-up settling** before the first read, and refuses to be read **more often than every ~1.5 s** (it answers with stale or no data). Your main loop paces itself accordingly.

## 21.3 Exercises

**Exercise 21.1: Edge-wait helpers.** `wait_low`/`wait_high` with `TIMEOUT_US` abort. Unit-test against a bare button: prove the timeout fires when nothing happens.

**Exercise 21.2: The hygrometer.** Full read at 0.5 Hz with all three result codes handled and *counted* — print `H:47% T:23C (ok:142 to:1 cs:0)`. A driver that reports its own reliability is a different species from one that pretends.

**Exercise 21.3: Pulse-width census.** Instrument the receiver to also record the 40 measured high-times of one frame into a stack array; dump them after. You will see two clean clusters (~27 µs, ~70 µs) — your threshold sits in the gulf between. *You have just used the CPU as a logic analyser.*

**Exercise 21.4: Comfort light.** RGB module shows a comfort verdict: blue <30%, green 30–60%, red >60% humidity — breathe on the sensor and watch it react. DHT pacing + LEDC + thresholds, one loop.

## 21.4 Git Checkpoint

```bash
cd ~/esp32c3-assembly-course
git add ch21/
git commit -m "feat(ch21): DHT11 receiver — mcycle pulse timing, checksum, stats"

# - [ ] Exercise 21.2: ok/timeout/checksum counters live
# - [ ] Exercise 21.3: two timing clusters recorded in a comment
# - [ ] Exercise 21.4: visibly reacts to breath within ~2 s

git tag -a ch21-complete -m "Chapter 21: DHT11 — forty bits by stopwatch"
git push --follow-tags        # if using a remote
```

## 21.5 Chapter Summary

- DHT11 inverts the master/slave deal: you wake it, then it owns the wire and the clock
- Bit value = pulse width; measurement = two `mcycle` captures and a threshold; the threshold belongs in the gulf between the clusters you *measured*
- Tight inner loops earn their inlined ugliness; everything else keeps the calling convention
- Real drivers return error codes and count their failures

**Check your understanding:**
1. Why does the DHT11's "50 µs low" before every bit carry no information, and why is it still essential to your receiver's structure?
2. Your readings are all 1-bits. Given the threshold logic, name two distinct bugs (one timing, one electrical) that produce exactly this symptom.
3. The inner stopwatch loop avoids `jal gpio_read`. Quantify roughly what each call would add per poll iteration, and explain which protocol number that jitter threatens.

## 21.6 References for This Chapter

- DHT11 datasheet (Aosong) — timing diagram and data format: [https://www.mouser.com/datasheet/2/758/DHT11-Technical-Data-Sheet-Translated-Version-1143054.pdf](https://www.mouser.com/datasheet/2/758/DHT11-Technical-Data-Sheet-Translated-Version-1143054.pdf)
- KY-015 module reference: [https://arduinomodules.info/ky-015-temperature-humidity-sensor-module/](https://arduinomodules.info/ky-015-temperature-humidity-sensor-module/)

---

# Chapter 22: The Rotary Encoder — Quadrature and GPIO Interrupts

## Learning Objectives

- Decode quadrature: two phase-shifted square waves into direction + count
- Implement the 16-entry state-transition table that makes decoding branch-free and bounce-tolerant
- Route a GPIO edge through the interrupt matrix to your Volume 1 trap handler
- Compare polled vs interrupt decoding under deliberate abuse, with numbers
- Know when interrupts are the right tool — and the discipline they demand

## 22.1 Two Waves, Ninety Degrees Apart

Spin the KY-040's shaft and its two contacts (CLK and DT) produce square waves a quarter-cycle out of phase. Direction is *which one leads*:

```
clockwise:            counter-clockwise:
CLK ─┐ ┌──┐ ┌──       CLK ──┐ ┌──┐ ┌─
     └─┘  └─┘              └─┘  └─┘
DT ──┐ ┌──┐ ┌─        DT  ─┐ ┌──┐ ┌──
    └─┘  └─┘               └─┘  └─┘
     CLK falls while DT high   CLK falls while DT low
```

Wiring: CLK→GPIO6, DT→GPIO7, SW→GPIO10 (the shaft is also a button), +→3V3, GND→GND. Both signal pins: `gpio_init_input_pullup`.

The naive decode — "on CLK falling edge, read DT" — works on a lab bench and falls apart on a real detented, bouncing encoder: missed steps, double counts, reversals. The robust decode treats the *pair* (CLK,DT) as a 2-bit state and scores every transition:

## 22.2 The State Table

Sixteen possible (previous-state, new-state) transitions. Four mean +1 (one Gray-code step CW), four mean −1, four mean "no change", four are *illegal* — the signature of bounce or a missed sample, scored 0:

```asm
        .section .rodata
        .align 2
# index = (prev << 2) | curr   — states are (CLK<<1)|DT
quad_table:
        .byte  0, -1, +1,  0
        .byte +1,  0,  0, -1
        .byte -1,  0,  0, +1
        .byte  0, +1, -1,  0
```

The decode is then a table lookup — no branching, no debounce delay, and bounce *self-cancels* (a bounce is a transition immediately reversed; +1 then −1 nets zero):

```asm
# encoder_poll_step(prev_state*) -> int   -1/0/+1
# a0 = pointer to 1-byte previous state; returns step in a0
        .global encoder_poll_step
encoder_poll_step:
        addi  sp, sp, -16
        sw    ra, 12(sp)
        sw    s0, 8(sp)
        sw    s1, 4(sp)
        mv    s0, a0
        li    a0, 6                      # CLK
        jal   ra, gpio_read
        slli  s1, a0, 1
        li    a0, 7                      # DT
        jal   ra, gpio_read
        or    s1, s1, a0                 # current 2-bit state
        lbu   t0, 0(s0)                  # previous
        slli  t0, t0, 2
        or    t0, t0, s1                 # table index
        la    t1, quad_table
        add   t1, t1, t0
        lb    a0, 0(t1)                  # signed step!  lb, not lbu
        sb    s1, 0(s0)                  # store new state
        lw    s1, 4(sp)
        lw    s0, 8(sp)
        lw    ra, 12(sp)
        addi  sp, sp, 16
        ret
```

`lb` versus `lbu` on the table read: −1 lives in that byte. Volume 1 Chapter 3's sign-extension material was preparation for this exact line.

## 22.3 The Polling Budget Problem

A detented encoder spun hard produces transitions ~1 ms apart; the bounce structure inside each is far faster. Polling at 10 kHz catches it — *if nothing else needs the CPU*. Add a DHT11 read (20 ms of undivided attention) and you drop encoder steps every time. This is the wall Chapter 15 predicted, and it is the genuine motivation for interrupts: *let the event interrupt you, instead of you checking for the event.*

## 22.4 GPIO Interrupts on the ESP32-C3

Volume 1 Chapter 11 built the machinery: `mtvec` → your trap handler, full caller-state save/restore, `mret`. The timer drove it then; now a GPIO edge will. Three configuration layers (TRM: GPIO chapter + Interrupt Matrix chapter):

```
1. PIN:    GPIO_PINn_REG — INT_TYPE field (1=rising, 2=falling, 3=any edge)
           + INT_ENA field (bit for "CPU interrupt")
2. MATRIX: route peripheral source → CPU interrupt line.
           INTERRUPT_CORE0_GPIO_INTERRUPT_PRO_MAP_REG = chosen CPU line (e.g. 3)
           (interrupt matrix base 0x600C2000; GPIO source's map register per TRM table)
3. CPU:    INTC — enable + priority for that CPU line; then mie/mstatus
           global enable exactly as in Volume 1 Ch11
```

In the handler: read `GPIO_STATUS_REG` (which pins fired), do the *minimum*, write `GPIO_STATUS_W1TC_REG` to acknowledge, restore, `mret`. Forgetting the W1TC acknowledge is the classic first bug: the same interrupt re-fires forever and the program appears to hang while actually trap-looping — Appendix H's debugger shows `pc` parked in your handler, which is exactly how you will diagnose it.

> The register names above are deliberately given as names + base addresses rather than a finished `.equ` block: the offsets live in two different TRM chapters and transcribing them — then *verifying each with the debugger* — is Exercise 22.3's first task. By this point in the book, that is a 20-minute job, and doing it cold is the skill.

**ISR discipline**, the three rules that survive every architecture:

1. **Short.** The handler updates the count and leaves. Printing, LEDC updates, anything slow — main loop's job.
2. **Communicate through memory.** A word in `.data` (`encoder_count`) written by the ISR, read by the main loop. (One core, one writer: no atomics needed — and now Volume 1's note about the A-extension has its context.)
3. **`volatile` thinking.** The main loop must *re-load* `encoder_count` each pass, never cache it in a register across the wait — you, not a compiler, are the optimiser now, and you must choose not to optimise that load away.

## 22.5 Exercises

**Exercise 22.1: Polled dial.** State-table decoder; spin → signed count over UART. Verify one detent = one count, both directions, no drift after vigorous abuse.

**Exercise 22.2: Drop demonstration.** Add a fake 20 ms blocking task to the loop. Spin during it; document the dropped steps. This number is the "before" picture.

**Exercise 22.3: Interrupt dial.** Any-edge interrupts on CLK and DT; the ISR runs the same state table against the same `prev_state`, accumulating into `encoder_count`. Main loop keeps its 20 ms blocker and now misses *nothing*. Commit before/after numbers side by side.

**Exercise 22.4: Volume knob.** Encoder sets brightness 0–100 on the onboard LED via LEDC; shaft-press toggles mute (restoring level on unmute). ISR counts; main loop maps count→gamma→duty. The cleanest possible demonstration of "ISR gathers, loop acts."

**Exercise 22.5 (stretch): Velocity.** `mcycle`-stamp each step in the ISR; steps arriving <2 ms apart count ×4. Spin fast to sweep brightness, slow for fine trim — the acceleration trick inside every good rotary UI, in twelve lines.

## 22.6 Git Checkpoint

```bash
cd ~/esp32c3-assembly-course
git add ch22/
git commit -m "feat(ch22): quadrature state-table + GPIO interrupts, drop-proof dial"

# - [ ] Exercise 22.2 vs 22.3: dropped-step numbers recorded
# - [ ] Exercise 22.4: mute/unmute restores level
# - [ ] ISR verified short: no UART, no LEDC inside handler

git tag -a ch22-complete -m "Chapter 22: Rotary encoder and GPIO interrupts"
git push --follow-tags        # if using a remote
```

## 22.7 Chapter Summary

- Quadrature encodes direction in phase; the 16-entry transition table decodes it branch-free and absorbs bounce arithmetically
- Polling has a budget; one slow task blows it; interrupts exist precisely for events that won't wait
- GPIO → matrix → CPU line → your Volume 1 trap handler: three layers, each verifiable with one debugger read
- ISRs gather, main loops act; they meet in a word of `.data`, and forgetting the status-clear write is the canonical first failure

**Check your understanding:**
1. Why does a contact bounce on CLK produce a net count of exactly zero with the state table, where the naive "read DT on CLK edge" decoder counts garbage?
2. The board hangs after the first detent click. Describe what `pc` looks like in the halted debugger and name the missing register write.
3. Why may the ISR and main loop share `encoder_count` without any atomic instructions on this chip — and which two changes to the system would revoke that permission?

## 22.8 References for This Chapter

- ESP32-C3 TRM — GPIO interrupt configuration and Interrupt Matrix chapters: [https://www.espressif.com/sites/default/files/documentation/esp32-c3_technical_reference_manual_en.pdf](https://www.espressif.com/sites/default/files/documentation/esp32-c3_technical_reference_manual_en.pdf)
- Quadrature state-table decoding (classic reference implementation): [https://www.mikrocontroller.net/articles/Drehgeber](https://www.mikrocontroller.net/articles/Drehgeber)
- KY-040 module reference: [https://arduinomodules.info/ky-040-rotary-encoder-module/](https://arduinomodules.info/ky-040-rotary-encoder-module/)

---

# Chapter 23: Infrared NEC — Build a Remote Control Link

## Learning Objectives

- Explain carrier modulation: why IR data rides on 38 kHz and what the receiver module does for you
- Transmit NEC frames: carrier bursts and silences timed to the microsecond
- Receive and decode NEC frames by pulse-width measurement, with inversion-based error checking
- Build a working wireless link between an emitter and a receiver — two boards or one
- Decode a real TV remote, because you now speak its language

## 23.1 Why 38 kHz

Sunlight, incandescent bulbs, and your heater all radiate infrared continuously. A bare IR signal would drown. The solution, standardised decades ago: *blink* the IR LED at 38 kHz and build receivers that respond only to 38 kHz blinking. The KY-022's receiver IC (a VS1838 descendant) contains the photodiode, amplifier, 38 kHz band-pass filter, and demodulator — and outputs a clean, *inverted* digital signal: **low while a 38 kHz burst is present, high during silence.**

So the link layer splits cleanly:

- **Emitter (KY-005 — a bare 940 nm IR LED):** you generate the 38 kHz carrier and gate it on/off in precisely timed bursts.
- **Receiver (KY-022):** hardware hands you burst/silence as low/high; you measure durations and decode. It is the DHT11 receiver pattern again, at a different timescale — which is why DHT11 came first.

## 23.2 The NEC Frame

The NEC protocol — used by countless TV and media remotes — encodes each bit in the length of the *silence* after a fixed 562.5 µs burst:

```
LEAD:    9000 µs burst + 4500 µs silence
32 BITS: burst 562 µs, then silence:  562 µs = 0    1687 µs = 1
TAIL:    one final 562 µs burst (closes the last bit's silence)

bit order: LSB first. Payload: ADDR, ~ADDR, CMD, ~CMD
```

That payload structure is free error detection: byte 2 must be the complement of byte 1, byte 4 of byte 3. A frame that fails either check is discarded, no CRC needed. Total frame ≈ 67.5 ms; remotes repeat a short "repeat code" (9 ms burst + 2.25 ms silence + one burst) while a key is held.

## 23.3 The Emitter

Wiring: KY-005 signal→GPIO7, GND→GND (the module is just an LED + resistor; some revisions omit the resistor — check Appendix C before powering).

Carrier generation is Chapter 17's choice point again, decided the other way: a 38 kHz *gated* carrier with microsecond gate edges is awkward through LEDC duty updates but trivial bit-banged — each carrier cycle is 26.3 µs (≈13 µs half-period), and `delay_us`'s cycle accuracy is ample. The emitter blocks during a frame; for a transmitter, that is fine.

```asm
# nec_tx.S — NEC transmitter on GPIO7
        .equ  IR_PIN, 7

# ir_burst(int us) — 38 kHz carrier for us microseconds (blocking)
# a0 = burst length. 1 cycle = 26 µs ⇒ cycles = us/26, 13 µs half-period
ir_burst:
        addi  sp, sp, -16
        sw    ra, 12(sp)
        sw    s0, 8(sp)
        li    t0, 26
        divu  s0, a0, t0                 # carrier cycles to emit
1:      li    a0, IR_PIN
        jal   ra, gpio_set
        li    a0, 13
        jal   ra, delay_us
        li    a0, IR_PIN
        jal   ra, gpio_clear
        li    a0, 13
        jal   ra, delay_us
        addi  s0, s0, -1
        bnez  s0, 1b
        lw    s0, 8(sp)
        lw    ra, 12(sp)
        addi  sp, sp, 16
        ret

# nec_send(int addr, int cmd) — one complete NEC frame
# a0 = address byte, a1 = command byte
        .global nec_send
nec_send:
        addi  sp, sp, -16
        sw    ra, 12(sp)
        sw    s0, 8(sp)
        sw    s1, 4(sp)
        # build 32-bit payload: CMD' CMD ADDR' ADDR (LSB of ADDR sent first)
        andi  a0, a0, 0xFF
        andi  a1, a1, 0xFF
        xori  t0, a0, 0xFF               # ~ADDR
        slli  t0, t0, 8
        or    s0, a0, t0
        xori  t0, a1, 0xFF               # ~CMD
        slli  t0, t0, 24
        slli  t1, a1, 16
        or    s0, s0, t1
        or    s0, s0, t0                 # s0 = full payload
        # lead
        li    a0, 9000
        jal   ra, ir_burst
        li    a0, 4500
        jal   ra, delay_us
        # 32 bits, LSB first
        li    s1, 32
2:      li    a0, 562
        jal   ra, ir_burst
        andi  t0, s0, 1
        srli  s0, s0, 1
        bnez  t0, 3f
        li    a0, 562                    # silence for 0
        j     4f
3:      li    a0, 1687                   # silence for 1
4:      jal   ra, delay_us
        addi  s1, s1, -1
        bnez  s1, 2b
        # tail burst
        li    a0, 562
        jal   ra, ir_burst
        lw    s1, 4(sp)
        lw    s0, 8(sp)
        lw    ra, 12(sp)
        addi  sp, sp, 16
        ret
```

You cannot see 940 nm — but your phone's front camera usually can (most lack the front IR filter). Point, send, watch the LED flicker purple on screen: the first debugging tool of every IR project.

## 23.4 The Receiver

Wiring: KY-022 OUT→GPIO6, VCC→3V3, GND→GND. Remember the inversion: **idle high, burst = low.**

The receiver is the DHT11 stopwatch, generalised: measure each (low, high) duration pair and classify.

```asm
# nec_rx.S — blocking NEC receive on GPIO6
# nec_receive(int timeout_ms) -> int
#   returns packed (ADDR<<8)|CMD, or -1 timeout, -2 frame error
# Strategy: wait for falling edge; measure low (expect ~9000);
#           measure high — 4500=frame, 2250=repeat(ignore or report);
#           then 32× { low ~562 ; high — classify 562 vs 1687 at 1100 µs }.
#   Verify ADDR vs ~ADDR, CMD vs ~CMD; assemble LSB-first.
```

The full body is Exercise 23.2 — every primitive is already in your hands (`wait_low`/`wait_high` with timeouts, `mcycle` deltas, the LSB-first shift-in). The interesting decisions are the tolerance windows: real remotes and real receivers smear the timings, so classify with bands (e.g. lead burst = 8,000–10,000 µs; bit silence: <1,100 µs → 0, otherwise 1) rather than equalities. Put the band constants in `.equ` at the top — you will tune them once against Exercise 23.5's census and never again.

## 23.5 Exercises

**Exercise 23.1: Phone-camera smoke test.** `nec_send(0x42, n)` once per second, n incrementing. Confirm flicker on camera. (No receiver involved — transmitter bring-up in isolation.)

**Exercise 23.2: The decoder.** Implement `nec_receive` per §23.4. Test source: a household TV/media remote pointed at KY-022. Print `ADDR=0x04 CMD=0x08` lines; map three buttons of the remote in a comment. *You are reading a consumer product's traffic with 150 lines of your own assembly.*

**Exercise 23.3: The link.** Two boards: board A sends a counter on BOOT-button press; board B prints received values and flashes the onboard LED. One board alternative: emitter on GPIO7 and receiver on GPIO6 of the *same* board, faced together a few centimetres apart; send, then immediately receive your own frame (the blocking send returns before the receiver's job starts — so for single-board loopback, point the emitter at the receiver and use a *second* press to test, or bounce off a white wall — both work and the geometry experiment is the point).

**Exercise 23.4: Remote-controlled everything.** Marry Exercise 23.2 to the kit: four remote buttons drive relay toggle, RGB colour step, buzzer chirp, and a "report" (DHT11 + DS18B20 readings over UART). The decoder becomes a command dispatcher — a `.rodata` table of (cmd, handler-address) pairs and an indirect `jalr`: Volume 1 Chapter 4's jump tables, now with a TV remote attached.

**Exercise 23.5 (stretch): Timing census.** As DHT Exercise 21.3: record one frame's 68 raw durations, dump, observe the clusters, set your tolerance bands from evidence.

## 23.6 Git Checkpoint

```bash
cd ~/esp32c3-assembly-course
git add ch23/
git commit -m "feat(ch23): NEC IR transmitter + receiver, remote decoded, link working"

# - [ ] Exercise 23.2: three real-remote buttons mapped
# - [ ] Exercise 23.3: link delivers incrementing counter
# - [ ] Exercise 23.4: four-command dispatcher table working

git tag -a ch23-complete -m "Chapter 23: NEC infrared from carrier to command"
git push --follow-tags        # if using a remote
```

## 23.7 Chapter Summary

- Modulation buys noise immunity: data rides a 38 kHz carrier; the receiver IC strips it and hands you inverted burst/silence
- NEC encodes bits in silence length after fixed bursts; complement bytes give free error checking
- Transmit = Chapter 17's bit-banged carrier, gated by Chapter 20-style timed phases; receive = Chapter 21's stopwatch with tolerance bands
- A protocol stack built by you, bottom to top: photons → demodulated edges → durations → bits → bytes → validated commands → dispatched actions

**Check your understanding:**
1. Why does the receiver module output *low* during a burst, and at which exact layer of your stack does that inversion get absorbed?
2. A frame arrives with ADDR=0x04, byte2=0xFA. Accept or reject — show the one-instruction check.
3. Why are tolerance *bands* mandatory in the receiver when the transmitter you wrote uses exact datasheet timings?

## 23.8 References for This Chapter

- NEC IR protocol specification (timing and frame format): [https://www.sbprojects.net/knowledge/ir/nec.php](https://www.sbprojects.net/knowledge/ir/nec.php)
- VS1838B receiver datasheet (demodulator behaviour): [https://www.vishay.com/docs/82459/tsop48.pdf](https://www.vishay.com/docs/82459/tsop48.pdf) *(TSOP48xx series — functional equivalent)*
- KY-005 / KY-022 module references: [https://arduinomodules.info](https://arduinomodules.info)

---

# PHASE 9: INTEGRATION
## Weeks 10 – 12 | Systems, Not Sensors

---

# Chapter 24: The Heartbeat Monitor — Sampling and Signal Processing

## Learning Objectives

- Sample an ADC channel at a fixed rate with `mcycle` pacing — the bedrock of all DSP
- Remove a wandering baseline with an integer exponential moving average
- Detect peaks with hysteresis and refractory time, and convert intervals to BPM
- Stream a signal up the UART and *look* at it — your terminal as oscilloscope

## 24.1 The Hardest Easy Sensor

The KY-039 shines IR through your fingertip; a phototransistor watches. Each heartbeat fractionally changes blood volume, which fractionally changes transmitted light. The signal is everything Chapter 18 warned about at once: tiny (tens of ADC counts), riding a huge wandering baseline (finger pressure, ambient light), and noisy. No threshold on the raw value can work. This chapter is the pipeline that makes it work anyway:

```
sample @ 100 Hz → baseline (EMA) → bandpass-ish residual → peak detect → BPM
```

Wiring: S→GPIO0, finger draped *gently* over the sensor (pressure changes are your worst noise source); shroud from room light with a fingertip-and-thumb cave or a strip of tape.

## 24.2 Fixed-Rate Sampling

DSP arithmetic assumes equally spaced samples. The pacing pattern — *absolute deadline*, not "delay after work" (which accumulates the work's duration as error):

```asm
# 100 Hz pacing skeleton
        csrr  s10, mcycle                # next deadline
        li    s11, 1600000               # 160e6 / 100
loop:
        add   s10, s10, s11              # deadline += period
1:      csrr  t0, mcycle
        bltu  t0, s10, 1b                # spin to deadline (work already done)
        jal   ra, take_sample_and_process
        j     loop
```

## 24.3 Baseline Removal: the Integer EMA

An exponential moving average with shift arithmetic — the single most useful filter in embedded work:

```asm
# baseline += (sample - baseline) >> 5        (alpha = 1/32)
# Track baseline in Q5 fixed point to avoid losing the small increments:
#   base_q5 += sample - (base_q5 >> 5);   baseline = base_q5 >> 5
        lw    t0, base_q5
        srai  t1, t0, 5                  # current baseline
        sub   t2, a0, t1                 # sample - baseline
        add   t0, t0, t2
        sw    t0, base_q5, t3
        sub   a0, a0, t1                 # residual = sample - baseline  → the pulse!
```

The residual wiggles around zero with the heartbeat; the wandering DC is gone. (The Q5 trick — keeping 5 fractional bits in the accumulator — is why the filter doesn't stall when `sample − baseline` is small. Fixed-point again, now in a feedback loop.)

## 24.4 Peak Detection with Hysteresis and Refractory Time

Two guards make a threshold trustworthy on a wobbly signal:

- **Hysteresis:** arm at residual > +T, fire only on the later crossing back below +T/2. One peak, one event, no chatter at the threshold.
- **Refractory window:** after a beat, ignore everything for 300 ms (no human heart beats >200 BPM). Chapter 15's dead-time idea, promoted to physiology.

Per beat: capture `mcycle`, subtract the previous capture, convert: `BPM = 9,600,000,000 / delta_cycles` — which overflows 32 bits, so compute as `BPM = 60,000 / delta_ms` with `delta_ms = delta_cycles / 160,000`. Two `divu`s, no 64-bit anything. Average the last 4 intervals before printing; single-interval BPM jitters annoyingly.

## 24.5 Seeing the Signal

Before any detection works, *look*: stream the residual at 100 Hz as one number per line, and pipe your monitor to a file or watch a poor-man's bar graph — print `#` × (residual/4) characters per line and the waveform scrolls past in ASCII. Ten minutes of looking saves two hours of threshold guessing; choose T at roughly half your observed peak height.

## 24.6 Exercises

**Exercise 24.1: Scope.** 100 Hz pacing + raw and residual columns over UART. Capture 10 s with a finger on; commit a short excerpt showing visible periodicity.

**Exercise 24.2: ASCII waveform.** The `#`-bar renderer. Identify your pulse by eye; note approximate peak residual height.

**Exercise 24.3: The monitor.** Full pipeline; onboard LED blinks per detected beat; `BPM: 72` updates per beat (4-interval average). Sanity-check against a watch and 30 s of counting.

**Exercise 24.4 (stretch): Quality gate.** Reject intervals <300 ms or >2000 ms and require 3 consecutive plausible intervals before trusting the display ("--" otherwise). The difference between a demo and an instrument is exactly this paragraph.

## 24.7 Git Checkpoint

```bash
cd ~/esp32c3-assembly-course
git add ch24/
git commit -m "feat(ch24): 100Hz sampler, EMA baseline, hysteresis peak detect, BPM"

# - [ ] Exercise 24.1: periodic residual captured
# - [ ] Exercise 24.3: BPM within ~5 of manual count
# - [ ] Refractory + hysteresis constants documented in comments

git tag -a ch24-complete -m "Chapter 24: Heartbeat — first real DSP pipeline"
git push --follow-tags        # if using a remote
```

## 24.8 Chapter Summary

- Fixed-rate sampling uses absolute deadlines; "work then delay" drifts
- An EMA in fixed point removes baseline wander in three instructions; the residual is the signal
- Hysteresis + refractory time turn a threshold into a detector; physiology sets the constants
- Look at signals before processing them — the UART is an oscilloscope if you let it be

**Check your understanding:**
1. Show arithmetically why "do work, then delay one period" sampling accumulates error, and why the absolute-deadline loop does not.
2. The EMA accumulator keeps 5 fractional bits. What failure appears if you store the baseline at integer precision instead?
3. Your monitor reads 145 BPM on a resting adult. Which two pipeline parameters do you suspect, in order, and what does each failure look like in the Exercise 24.2 waveform?

## 24.9 References for This Chapter

- Exponential moving averages in fixed point (classic embedded filter reference): [https://en.wikipedia.org/wiki/Exponential_smoothing](https://en.wikipedia.org/wiki/Exponential_smoothing)
- Photoplethysmography principles (how optical pulse sensing works): [https://en.wikipedia.org/wiki/Photoplethysmogram](https://en.wikipedia.org/wiki/Photoplethysmogram)

---

# Chapter 25: Capstone — The Sensor Station

## Learning Objectives

- Architect a multi-sensor, multi-output firmware as a cooperative time-sliced loop
- Schedule tasks with different natural periods (10 ms to 2 s) without an RTOS
- Combine five of your own drivers — unchanged — into one coherent instrument
- Design the UART output as an interface, not a debug stream

## 25.1 The Brief

Build a desk environment station:

| Function | Driver | Period |
|----------|--------|--------|
| Temperature | DS18B20 (Ch20) | 2 s |
| Humidity | DHT11 (Ch21) | 2 s (offset 1 s from DS18B20) |
| Ambient light | KY-018 + ADC (Ch18) | 100 ms |
| User input | Rotary encoder + ISR (Ch22) — selects display page | continuous |
| Status | RGB via LEDC (Ch17) — green/amber/red comfort verdict | on change |
| Alarm | Passive buzzer chirp when any reading crosses a limit | event |
| Console | One formatted status line per second + page detail | 1 s |

Everything is a driver you already wrote and tagged. The chapter's content is the *architecture* that lets them coexist.

## 25.2 The Cooperative Scheduler

No RTOS — a deadline table and one loop. Each task owns a `next_due` mcycle value in `.data`; the loop scans, runs whatever is due, and the cardinal rule keeps it honest: **no task blocks longer than the shortest period it shares a loop with** — with two licensed exceptions you must *schedule around*, not wish away:

- `dht11_read` blocks ~25 ms and is timing-critical inside. Run it only when nothing else falls due within 30 ms (check the deadline table before committing).
- DS18B20's 750 ms conversion is solved by *splitting* the driver: task A starts a conversion; task B, due 800 ms later, collects it. Asynchronous thinking with synchronous tools — the most transferable idea in the chapter.

```asm
        .section .data
        .align 2
sched_table:                       # (next_due_cycles, handler) pairs
        .word 0, task_light        # 100 ms
        .word 0, task_ds_start     # 2 s
        .word 0, task_ds_collect   # (armed by ds_start)
        .word 0, task_dht          # 2 s, offset
        .word 0, task_console      # 1 s
        .word 0, task_status_led   # on flag
sched_table_end:
```

The scan loop walks the table, compares each deadline against `mcycle` (unsigned compare — wraparound after ~26 s of cycles handled by *relative* comparison: `now − due` as signed, fire when ≥ 0; this subtlety is §25.3), `jalr`s the handler, and the handler re-arms its own deadline. The encoder ISR runs underneath everything, untouched.

## 25.3 The Wraparound Trap

`mcycle` is 32 bits at 160 MHz: it wraps every 26.8 seconds. Compare deadlines with subtraction, never magnitude: `sub t0, now, due; bgez t0, fire` — two's-complement subtraction is immune to the wrap as long as intervals stay under half the counter range. This one idiom is the difference between a station that runs for a demo and one that runs for a month. (It is also Volume 1 Chapter 2's "overflow wraps silently" — finally a feature.)

## 25.4 Exercises

**Exercise 25.1: Skeleton.** Scheduler + two dummy tasks (LED togglers at 100 ms / 1 s). Prove independent periods and wrap-safety (let it run >60 s).

**Exercise 25.2: Integration, one driver at a time.** Light → DS18B20 split-task → DHT11 with its look-ahead guard → console → RGB verdict → buzzer alarm. One commit per driver landed; the discipline *is* the exercise.

**Exercise 25.3: Pages.** Encoder turns select console detail pages (light stats / temp history / uptime + scheduler stats); shaft-press forces immediate refresh. ISR count → page index, main loop renders.

**Exercise 25.4: The soak.** Run overnight. In the morning: uptime, task run-counts, DHT error counters (Ch21's honesty paying off). Commit the numbers. An instrument that survived the night, built from the silicon up — that is the course, demonstrated.

## 25.5 Git Checkpoint

```bash
cd ~/esp32c3-assembly-course
git add ch25/
git commit -m "feat(ch25): sensor station — cooperative scheduler, 6 tasks, soak-tested"

# - [ ] Exercise 25.1: wrap-safe past 60 s
# - [ ] Exercise 25.2: one commit per integrated driver
# - [ ] Exercise 25.4: overnight soak stats recorded

git tag -a ch25-complete -m "Chapter 25: Capstone sensor station"
git push --follow-tags        # if using a remote
```

## 25.6 Chapter Summary

- A deadline table + one scan loop schedules wildly different periods without an RTOS
- Long operations either fit between deadlines (look-ahead) or split into start/collect halves
- Signed-subtraction deadline comparison survives counter wraparound; magnitude comparison does not
- Integration discipline — one driver, one commit, re-verify — is how systems stay debuggable

**Check your understanding:**
1. Why is `bgeu now, due` wrong for deadline checks and `sub` + `bgez` right? Construct the failing case near wraparound.
2. The DHT11 guard checks "nothing due within 30 ms." What specific corruption appears in *another* task's output if you skip the guard, and why doesn't the DHT read itself fail?
3. Splitting DS18B20 into start/collect added a third pseudo-task. What state must persist between the halves, and where does it live?

## 25.7 References for This Chapter

- Cooperative scheduling patterns in bare-metal firmware: [https://www.embedded.com/cooperative-multitasking/](https://www.embedded.com/cooperative-multitasking/)
- (All sensor references: Chapters 17–22 of this volume)

---

# Chapter 26: The BLE Sensor Beacon — Closing the Loop with Volume 1

## Learning Objectives

- Carry live sensor data into the BLE advertising payload built in Volume 1 Chapter 14
- Structure manufacturer-specific advertising data and update it on a timer
- Re-enter the ESP-IDF-assisted world deliberately, knowing exactly which layers are yours
- Finish a two-volume arc: from `li a0, 42` to a phone displaying your room's temperature, no libraries you didn't write or explicitly call

## 26.1 The Return of the Stack

Phases 6–9 were pure bare metal. This final chapter deliberately steps back into Volume 1 Chapter 14's territory — `ble_asm.S` with its ESP-IDF radio calls — and threads your Volume 2 work into it. The point is perspective: you now know precisely where the bare metal ends (your DS18B20 driver, your scheduler) and where the vendor stack begins (the 2.4 GHz radio, for excellent regulatory and complexity reasons). Choosing the boundary consciously is the engineering skill; both volumes exist so the choice is *yours*.

The product: a **connectionless sensor beacon**. No GATT connection needed — the temperature rides inside the advertising packet itself, and any BLE scanner app (nRF Connect, LightBlue) on a phone shows it live.

## 26.2 Manufacturer Data with a Payload

Volume 1 Chapter 14 built the advertising payload with Length-Type-Value entries. Add (or extend) one LTV: type `0xFF` (Manufacturer Specific Data), company ID `0xFFFF` (reserved-for-testing), then your bytes:

```asm
        .section .data                  # .data — we rewrite it live
        .align 2
adv_mfg_entry:
        .byte 7                          # length: type + company(2) + payload(4)
        .byte 0xFF                       # manufacturer-specific
        .byte 0xFF, 0xFF                 # test company ID
adv_temp_raw:
        .byte 0, 0                       # DS18B20 raw, little-endian
adv_seq:
        .byte 0                          # sequence counter
adv_flags_v2:
        .byte 0                          # bit0: DHT valid last cycle
```

The update cycle, on a 2-second schedule (your Chapter 25 scheduler, naturally):

```
1. ds18b20 collect → store raw into adv_temp_raw (sb, sb — little-endian)
2. adv_seq++
3. esp_ble_gap_stop_advertising
4. esp_ble_gap_config_adv_data_raw(payload, len)     ← Vol 1 Ch14 call pattern
5. esp_ble_gap_start_advertising
```

The stop/config/start dance is the stack's requirement for changing live advertising data; your Volume 1 event-handler machinery already sequences exactly such calls. The sequence counter is there so the phone-side proves freshness — a beacon that says the same thing forever is indistinguishable from a frozen one (Chapter 21's honesty principle, now wireless).

## 26.3 Reading It

nRF Connect → scan → your device (Vol 1's name, perhaps now `ASM-SENSE`) → advertisement details → Manufacturer Data: `FF FF 91 01 2A 01`. Decode by eye: `0x0191` little-endian = 401 → 401/16 = **25.06 °C**, sequence 0x2A, DHT-valid flag set. Your fingertip-to-phone stack, every byte of the payload placed by an `sb` you wrote.

## 26.4 Exercises

**Exercise 26.1: Static beacon.** Manufacturer LTV with fixed bytes `0xCA 0xFE`; verify on the phone. (Payload plumbing before live data — the integration order rule, one last time.)

**Exercise 26.2: Live beacon.** DS18B20 → payload at 0.5 Hz with sequence counter. Warm the sensor in your hand and watch the phone update.

**Exercise 26.3: Station-on-air.** Fold the beacon into the Chapter 25 station as one more scheduled task (the radio calls block briefly — give the task the DHT-style look-ahead guard). The station now reports to the desk *and* the air.

**Exercise 26.4 (course finale): The full demo.** One run, narrated in your README: BOOT press → station boots, beacon live, encoder pages working, remote control (Ch23) toggles the relay, phone shows the temperature climbing as you hold the DS18B20. Tag it.

## 26.5 Git Checkpoint

```bash
cd ~/esp32c3-assembly-course
git add ch26/
git commit -m "feat(ch26): BLE sensor beacon — live DS18B20 in manufacturer data"

# - [ ] Exercise 26.2: phone shows live temperature + advancing sequence
# - [ ] Exercise 26.3: beacon task coexists with full station
# - [ ] Exercise 26.4: finale demo narrated in README

git tag -a ch26-complete -m "Chapter 26: BLE beacon — Volume 2 complete"
git tag -a vol2-complete -m "Volume 2 complete: 37 modules, 12 chapters, one instrument"
git push --follow-tags        # if using a remote
```

## 26.6 Chapter Summary — and the Two-Volume Arc

- Advertising payloads are byte arrays you own; live data is `sb` instructions on a schedule
- The bare-metal/vendor-stack boundary is a *decision*, and you now have the knowledge to place it deliberately
- The arc, complete: fetch-decode-execute → registers → memory → control flow → the stack → MMIO → interrupts → boot → radio (Vol 1); inputs → outputs → PWM → ADC → three timing protocols → interrupt-driven decoding → DSP → scheduling → integration (Vol 2). Thirty-seven modules, two volumes, zero magic remaining.

**Check your understanding:**
1. Why does changing live advertising data require the stop/config/start sequence rather than rewriting the buffer in place?
2. The temperature bytes go into the payload little-endian. Which Volume 1 chapter made that an instinct rather than a lookup, and what would a big-endian mistake look like in nRF Connect?
3. The beacon task got the DHT-style scheduling guard. Which property of the radio calls demanded it, and what station symptom would appear without it?

## 26.7 References for This Chapter

- Bluetooth Core Specification — Advertising Data formats (LTV structures): [https://www.bluetooth.com/specifications/specs/core-specification/](https://www.bluetooth.com/specifications/specs/core-specification/)
- Assigned Numbers — AD types (0xFF Manufacturer Specific Data): [https://www.bluetooth.com/specifications/assigned-numbers/](https://www.bluetooth.com/specifications/assigned-numbers/)
- nRF Connect for Mobile: [https://www.nordicsemi.com/Products/Development-tools/nRF-Connect-for-mobile](https://www.nordicsemi.com/Products/Development-tools/nRF-Connect-for-mobile)

---

# APPENDIX A: The Complete 37-Module Catalog

Every module in the kit, its interface family, the default wiring used in this volume, its 3.3 V verdict, and where it appears. Modules are listed in KY-number order. "S/+/−" follows the kit's usual silkscreen; exceptions are flagged → Appendix C.

| KY | Name | Family | Default ESP32-C3 wiring | 3.3 V? | Chapter |
|----|------|--------|------------------------|--------|---------|
| KY-001 | Temperature sensor (DS18B20, 1-Wire) | Protocol | S→GPIO10, +→3V3, −→GND | Yes | 20, 25, 26 |
| KY-002 | Vibration switch | Digital in | S→GPIO10, +→3V3, −→GND | Yes | 15 |
| KY-003 | Hall magnetic sensor (digital) | Digital in | S→GPIO10, +→3V3, −→GND | Yes | 15 |
| KY-004 | Key switch (push button) | Digital in | S→GPIO10, +→3V3, −→GND | Yes | 15, 16 |
| KY-005 | Infrared emitter (940 nm LED) | Protocol (TX) | S→GPIO7, −→GND | Yes — check resistor (App. C) | 23 |
| KY-006 | Small passive buzzer | PWM out | S→GPIO7, −→GND | Yes (quieter than 5 V) | 17, 19, 25 |
| KY-008 | Laser emitter | Digital out | S→GPIO7, −→GND | Yes | 16 |
| KY-009 | 3-colour RGB LED (SMD, no resistors!) | PWM out | R→GPIO7, G→GPIO6, B→GPIO5 via ~150 Ω each, −→GND | Yes — external resistors REQUIRED | 17 |
| KY-010 | Optical broken (photo-interrupter) | Digital in | S→GPIO10, +→3V3, −→GND | Yes | 15 |
| KY-011 | 2-colour LED 5 mm (R/G) | Digital out | R→GPIO7, G→GPIO6, −→GND | Yes | 16 |
| KY-012 | Active buzzer | Digital out | S→GPIO7, −→GND | Yes | 16 |
| KY-013 | Temperature sensor (NTC analog) | Analog | S→GPIO0, +→3V3, −→GND | Yes — required | 18 |
| KY-015 | Temperature + humidity (DHT11) | Protocol | S→GPIO10, +→3V3, −→GND | Yes | 21, 25 |
| KY-016 | 3-colour RGB LED (through-hole, resistors on board) | PWM out | R→GPIO7, G→GPIO6, B→GPIO5, −→GND | Yes | 17, 18, 21 |
| KY-017 | Mercury tilt switch | Digital in | S→GPIO10, +→3V3, −→GND | Yes | 15 |
| KY-018 | Photoresistor (LDR) | Analog | S→GPIO0, +→3V3, −→GND | Yes — required | 18, 25 |
| KY-019 | 5 V relay | Digital out | S→GPIO7, **VCC→5V pin**, −→GND | Coil 5 V; signal 3.3 V OK | 16, 18, 23 |
| KY-020 | Tilt switch (ball) | Digital in | S→GPIO10, +→3V3, −→GND | Yes | 15, 16 |
| KY-021 | Mini magnetic reed | Digital in | S→GPIO10, +→3V3, −→GND | Yes | 15 |
| KY-022 | Infrared receiver (VS1838-class) | Protocol (RX) | S→GPIO6, +→3V3, −→GND | Yes | 23 |
| KY-023 | XY joystick + switch | Analog ×2 + digital | VRx→GPIO0, VRy→GPIO1, SW→GPIO10, +→3V3 | Yes | 19 |
| KY-024 | Linear Hall sensor (analog + digital) | Analog | A0→GPIO0, D0 unused, +→3V3, −→GND | Yes | 18 |
| KY-025 | Reed module (with comparator) | Digital in | D0→GPIO10, +→3V3, −→GND | Yes | 15 |
| KY-026 | Flame sensor (analog + digital) | Both | A0→GPIO0 and/or D0→GPIO10 | Yes — re-tune trimpot | 15, 18 |
| KY-027 | Magic light cup (tilt + LED, ×2) | Digital in + PWM out | tilt S→GPIO10/GPIO9, LED→GPIO7/GPIO6 | Yes | 17 |
| KY-028 | Temperature sensor (NTC + LM393) | Both | A0→GPIO0, D0→GPIO10 | Yes — re-tune trimpot | 18 |
| KY-029 | 2-colour LED 3 mm (Yin Yi) | Digital out | R→GPIO7, G→GPIO6, −→GND | Yes | 16 |
| KY-031 | Hit (knock) sensor | Digital in | S→GPIO10, +→3V3, −→GND | Yes | 15, 16 |
| KY-032 | Obstacle avoidance (reflective IR + comparator) | Digital in | S→GPIO10, +→3V3, −→GND | Yes — re-tune trimpot | 15 |
| KY-033 | Hunt / line-tracking sensor | Digital in | S→GPIO10, +→3V3, −→GND | Yes — re-tune trimpot | 15 |
| KY-034 | Automatic flashing colourful LED | Digital out | S→GPIO7, −→GND | Yes | 16 |
| KY-035 | Bihor analog Hall sensor | Analog | S→GPIO0, +→3V3, −→GND | Yes | 18 |
| KY-036 | Metal touch sensor | Digital in | D0→GPIO10, +→3V3, −→GND | Yes | 15 |
| KY-037 | Sensitive microphone (big mic, LM393) | Both | A0→GPIO0, D0→GPIO10 | Yes — re-tune trimpot | 18 |
| KY-038 | Microphone sound sensor (small) | Both | A0→GPIO0, D0→GPIO10 | Yes — re-tune trimpot | 18 |
| KY-039 | Heartbeat sensor (IR photoplethysmograph) | Analog | S→GPIO0, +→3V3, −→GND | Yes | 24 |
| KY-040 | Rotary encoder + switch | Protocol | CLK→GPIO6, DT→GPIO7, SW→GPIO10, +→3V3 | Yes | 22, 25 |

**Family totals:** Digital in 12 · Digital out 6 · PWM out 4 · Analog 8 (+2 dual counted once) · Protocol 6 · Mixed 1 — all 37 modules driven, every one from assembly you wrote.

---

# APPENDIX B: Volume 2 Peripheral Register Reference

Registers introduced in this volume, grouped by peripheral. Volume 1 Appendix B covers GPIO output, UART, timers, and CSRs. All addresses from the ESP32-C3 Technical Reference Manual; the standing instruction applies — *verify against your TRM revision with the Appendix H (Vol 1) debugger before trusting any offset*.

## GPIO Input (Chapter 15)

| Register | Address | Used for |
|----------|---------|----------|
| `IO_MUX_GPIOn_REG` | `0x60009004 + 4n` | `MCU_SEL` (14:12) =1, `FUN_IE` (9), `FUN_WPU` (8), `FUN_WPD` (7) |
| `GPIO_ENABLE_W1TC_REG` | `0x60004028` | Disable output driver (input / open-drain release) |
| `GPIO_ENABLE_W1TS_REG` | `0x60004024` | Enable output driver (open-drain pull low) |
| `GPIO_IN_REG` | `0x6000403C` | Read all pin levels |

## LEDC — PWM (Chapter 17)

Base `0x60019000`. Channel stride `0x14`; timer stride `0x08`.

| Register | Offset | Used for |
|----------|--------|----------|
| `LEDC_CHn_CONF0_REG` | `0x0000 + 0x14n` | Timer select (1:0), `SIG_OUT_EN` (2) |
| `LEDC_CHn_HPOINT_REG` | `0x0004 + 0x14n` | Count where output goes high (0) |
| `LEDC_CHn_DUTY_REG` | `0x0008 + 0x14n` | Duty × 16 |
| `LEDC_CHn_CONF1_REG` | `0x000C + 0x14n` | `DUTY_START` (31) latches duty |
| `LEDC_TIMERn_CONF_REG` | `0x00A0 + 0x08n` | Resolution bits (3:0), divider 18.8 FP, param-update |
| `LEDC_CONF_REG` | `0x00D0` | Global clock select (APB) |
| `SYSTEM_PERIP_CLK_EN0_REG` | system block | `LEDC_CLK_EN` gate (+ clear matching reset bit) |

GPIO matrix output routing: `GPIO_FUNCn_OUT_SEL_CFG_REG` at `0x60004554 + 4n`; LEDC channel 0 signal index = **45** (ch k → 45+k).

## SAR ADC (Chapter 18)

Base `0x60040000` (`APB_SARADC`).

| Register | Offset | Used for |
|----------|--------|----------|
| `ONETIME_SAMPLE` | `0x0020` | Channel (28:25), atten (24:23), SAR1 enable (31), START (29) |
| `INT_RAW` | `0x0040` | `ADC1_DONE` (31) poll target |
| `INT_CLR` | `0x0048` | Clear done flag |
| `SAR1_DATA` | `0x002C` | 12-bit result (11:0) |
| `SYSTEM_PERIP_CLK_EN0_REG` | system block | `APB_SARADC_CLK_EN` gate |

Attenuation code 3 = 11 dB ≈ 0.05–2.5 V usable. ADC1 channels 0–4 = GPIO0–4. ADC2: do not use.

## GPIO Interrupts (Chapter 22)

| Register | Address | Used for |
|----------|---------|----------|
| `GPIO_PINn_REG` | `0x60004074 + 4n` | `INT_TYPE` (9:7): 1 rise / 2 fall / 3 any; `INT_ENA` (17:13) |
| `GPIO_STATUS_REG` | `0x60004044` | Which pins fired |
| `GPIO_STATUS_W1TC_REG` | `0x6000404C` | Acknowledge (the write everyone forgets) |
| Interrupt matrix | `0x600C2000 + …` | GPIO source → CPU interrupt line map (TRM table) |
| INTC enable/priority | per TRM | Arm the chosen CPU line; then `mie`/`mstatus` as Vol 1 Ch11 |

---

# APPENDIX C: Wiring Gotchas and Module Quirks

Field notes for this specific kit. Ten minutes here saves an evening per quirk.

**Pin-label chaos.** The kit mixes `S/+/−`, `S/VCC/GND`, and `DO/AO/G/+` silkscreens, and a few boards print labels on the underside. The middle pin is *usually* VCC on three-pin modules — but on **KY-004** many revisions put VCC on the outer pin and S in the middle, and on **KY-011/KY-029** the middle is the common cathode. Rule: trace the ground pour first (Front Matter, Wiring Conventions), and when a sensor "reads inverted," suspect S/VCC swap before suspecting your code.

**KY-009 has no resistors.** The SMD RGB module is a bare LED on a carrier. Driving it pin-to-pin will source far beyond the 20 mA budget and cook either the LED or your pad's lifespan. ~150 Ω in series per colour, no exceptions. (KY-016 looks similar but carries its resistors — check for the three black 102/151 components.)

**KY-005 sometimes has no resistor either.** Some kit revisions ship the IR emitter as a bare LED. Measure: if the board has only the LED, add ~100 Ω in series. The phone-camera test (Ch23) confirms life either way.

**Trimpot modules arrive tuned for 5 V.** Every LM393 board (KY-025/026/028/032/033/036/037/038) has its threshold pot set at the factory, at 5 V, by nobody in particular. At 3.3 V the comparison point shifts: before first use, power the module and turn the pot until the onboard status LED sits just on the off side of its trip point, then verify with your Chapter 15 test program.

**KY-003 sees one pole only.** The digital Hall switch trips on (typically) the south pole. If a magnet "doesn't work," flip it. The analog KY-024/KY-035 see both, as Chapter 18's signed-deviation exercise exploits.

**KY-017 and KY-020 are slower than they look.** Mercury and ball-tilt switches settle for tens of milliseconds after movement stops; debounce windows shorter than ~20 ms chatter. The Ch15 counting filter at 5 × 5 ms handles both.

**KY-034 cannot be controlled.** The auto-flash LED's colour cycle is fixed in its die. If your "PWM dimming" of it produces strobing nonsense — that is the module working as designed. Gate it; don't modulate it.

**KY-019 relay: the clicking placebo.** The coil clicks happily even when the switched circuit is miswired — the click proves the coil, not the contacts. Verify NO/COM continuity with the LED-through-contacts test from Exercise 16.3. And the standing rule: low-voltage loads only, never mains.

**DHT11 (KY-015) sulks.** Power-up settle ≥1 s; reads ≥1.5 s apart; and long jumper wires (>20 cm) on the data line start corrupting bits — keep it short or accept the occasional checksum fail your Ch21 counters will faithfully report.

**Joystick (KY-023) mounting matters.** Hand-held, the calibration drifts with grip pressure on the board itself. Tape or breadboard-mount it before trusting the centre calibration.

**The heartbeat sensor (KY-039) hates light and pressure.** Room light saturates it; firm finger pressure kills the AC signal. Drape, don't press, and shade it. If the Ch24 residual is flat, fix the physical setup before touching the code — the silicon is the ground truth, and so is the finger.

---

*Bare Metal: Assembly Programming the ESP32-C3 — Volume 2: Sensors and Signals*

*Version 1.0 — Initial release: Phases 6–9 (Chapters 15–26), 37-module catalog, peripheral register reference, wiring quirks appendix. Companion to Volume 1 v2.3; same repository, same conventions, tags ch15-complete through vol2-complete.*

*End of Volume 2*
