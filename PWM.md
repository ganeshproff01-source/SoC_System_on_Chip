
## ⚙️ PWM Peripheral

### Overview
Pulse Width Modulation (PWM) is a technique to control the width (duty) of digital pulses while maintaining a constant frequency. Optionally, both duty and frequency can be modulated. PWM is commonly used to:

- Control the brightness of LEDs.
- Drive and control the speed of DC motors.
- Generate analog-like signals such as sine waves via filtering.

### 🔹 Key Features
- Programmable duty cycle (12 bits)
- Programmable period (12 bits)
- Prescaler (5 bits) for frequency division
- Enable bits for PWM and timer
- Memory-mapped at 0x00010000
- Glitch-free duty update

### Internal Architecture
| Block | Description |
|:------|:-------------|
| **Prescaler** | Programmable clock divider. Gates clk with en_prescaler and divides by 2^(prescale+1). |
| **Period Register** | 12-bit register (PR) for PWM period. |
| **Timer** | 12-bit counter (TMR) increments to PR, then rolls over. |
| **Comparator B** | Sets S=1 when TMR == PR (rollover). |
| **Duty Cycle Registers** | Master/slave for glitch-free update on rollover. |
| **Comparator A** | Sets R=1 when TMR == duty_cycle_out. |
| **SR Latch** | Holds PWM high until R, sets on S. |

PWM Control Register (32-bit)

| 31 | 30-19 | 18-7 | 6-2 | 1 | 0 |
|----|-------|------|-----|---| - |
| - | DC_in | PR | PSC | TE | PE |

Bit Fields Explanation:
- PE (PWM Enable): Bit 0 – Enables the PWM module. Corresponds to pwm_reg[0].
- TE (Timer Enable): Bit 1 – Enables the internal timer. Corresponds to pwm_reg[1].
- PSC (Prescaler): Bits [6:2] – Sets the prescaler value to divide the input clock. Corresponds to pwm_reg[6:2].
- PR (Period Register): Bits [18:7] – Sets the period of the PWM signal. Corresponds to pwm_reg[18:7].
- DC in (Duty Cycle Input): Bits [30:19] – Defines the duty cycle of the PWM signal. Corresponds to pwm_reg[30:19].
- - (Unused): Bit 31 – Reserved/unused.

Specifications
- PWM Frequency Formula: f_PWM = f_clk / (2^(prescale+1) * (Period + 1))  
  For a 100 MHz clock, the minimum possible frequency is approximately 5.68 µHz.
- Duty Cycle Formula: Duty % = (Duty_reg_value / (Period + 1)) * 100  
  The duty cycle can range from 0% to 100%.

### How to Use PWM
The PWM peripheral is memory-mapped at address 0x00010000. It is used to generate a pulse-width modulated signal, where the duty cycle and period are programmable via a control register. The processor writes the PWM control word to this address through the APB interface.

Example:
```
pwm : ; PWM address is 0x00010000
org 0x00000000:
movu r2,0x31AC
; Load upper part of control word
addh r2,r2,0x000A
; Final control word = 0x00031ACA

movu r1,0x0001
addh r1,r1,0x0000
; r1 = 0x00010000 (PWM address)

st r2,0[r1]
; Write control word to PWM register
```

This example sets up the PWM module by writing a 20-bit control word (0x31ACA) to its memory-mapped register. The control word typically encodes settings such as duty cycle, period, and enable bits (based on the design of the PWM control format).

| Problem | Cause | Solution |
|:---------|:------|:----------|
| Glitch in duty | Update during cycle | Use slave register for rollover update |
| Wrong frequency | Incorrect prescaler | Verify PSC bits |