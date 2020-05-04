# Audio Driver

The [Audio feature](feature_audio.md) breaks the hardware specifics out into separate, exchangeable driver units, with a common interface to the audio-"core" - which itself handles playing songs and notes while tracking their progress in an internal state, initializing/starting/stopping the driver as needed.

Not all MCUs support every available driver, either the platform-support is not there (yet?) or the MCU simply does not have the required hardware peripheral.


## AVR

Boards build around an Atmega32U4 can use two sets of PWM capable pins, each driving a separate speaker.
The possible configurations are:

|              | Timer3      | Timer1       |
| one speaker  | C4,C5 or C6 |              |
| one speaker  |             | B4, B5 or B7 |
| two speakers | C4,C5 or C6 | B4, B5 or B7 |

Currently there is only one/default driver for AVR based boards, which is automatically configured to:
```make
AUDIO_DRIVER = pwm_hardware
```


## ARM

For Arm based boards, QMK depends on ChibiOS - hence any MCU supported by the later is likely usable, as long as certain hardware peripherals are available.

Supported wiring configurations, with their ChibiOS/MCU peripheral requirement are listed below;
piezo speakers are marked with :one: for the first/primary and :two: for the secondary.

  | driver       | GPTD6                            | GPTD7                  | GPTD8 | PWMD1 [^1]      |
  |              | Tim6                             | Tim7                   | Tim8  | Tim1_Ch1        |
  |--------------|----------------------------------|------------------------|-------|-----------------|
  | dac_basic    | A4+DACD1 = :one:                 | A5+DACD2 = :one:       | state |                 |
  |              | A4+DACD1 = :one: + Gnd           | A5+DACD2 = :two: + Gnd | state |                 |
  |              | A4+DACD1 = :two: + Gnd           | A5+DACD2 = :one: + Gnd | state |                 |
  |              | A4+DACD1 = :one: + Gnd           |                        | state |                 |
  |              |                                  | A5+DACD2 = :one: + Gnd | state |                 |
  | dac_additive | A4+DACD1 = :one: + Gnd           |                        |       |                 |
  |              | A5+DACD2 = :one: + Gnd           |                        |       |                 |
  |              | A4+DACD1 + A5+DACD2 = :one: [^2] |                        |       |                 |
  | pwm_software | state-update                     |                        |       | any = :one:     |
  | pwm hardware | state-update                     |                        |       | A8 = :one: [^3] |


[^1]: the routing and alternate functions for PWM differ sometimes between STM32 MCUs, if in doubt consult the data-sheet
[^2]: one piezo connected to A4 and A5, with AUDIO_PIN_ALT_AS_NEGATIVE set
[^3]: TIM1_CH1 = A8 on STM32F103C8, other combinations are possible, see Data-sheet. configured with: AUDIO_PWM_TIMER and AUDIO_PWM_TIMERCHANNEL



### DAC basic

The default driver for ARM boards, in absence of an overriding configuration.
This driver needs one Timer per enabled/used DAC channel, to trigger conversion; and a third timer to trigger state updates with the audio-core.


### DAC additive

only needs one timer (GPTD6, Tim6) to trigger the DAC unit to do a conversion; the audio state updates are in turn triggered during the DAC callback.


### PWM hardware

this driver uses the ChibiOS-PWM system to produce a square-wave on specific output pins that are connected to the PWM hardware.
The hardware directly toggles the pin via its alternate function. see your MCUs data-sheet for which pin can be driven by what timer - looking for TIMx_CHy and the corresponding alternate function.

a configuration example for the STM32F103C8 would be:
``` c
//halconf.h:
#define HAL_USE_PWM                 TRUE
#define HAL_USE_PAL                 TRUE
#define HAL_USE_GPT                 TRUE
```

``` c
// mcuconf.h:
#define STM32_PWM_USE_TIM1                  TRUE
#define STM32_GPT_USE_TIM3                  TRUE
```

If we now target pin A8, looking trough the data-sheet of the STM32F103C8, for the timers and alternate functions
TIM1_CH1 = PA8 <- alternate0
TIM1_CH2 = PA9
TIM1_CH3 = PA10
TIM1_CH4 = PA11

with all this information, the configuration would contain these lines:
``` c
//config.h:
#define AUDIO_PIN A8
#define AUDIO_PWM_TIMER 1
#define AUDIO_PWM_TIMERCHANNEL 1
```

ChibiOS uses GPIOv1 for the F103, which only knows of one alternate function.
On 'larger' STM32s, GPIOv2 or GPIOv3 are used; with them it is also necessary to configure `AUDIO_PWM_PINALTERNATE_FUNCTION` to the correct alternate function for the selected pin, timer and timer-channel.


### PWM software

This driver uses the PWM callbacks from PWMD1 with TIM1_CH1 to toggle the selected AUDIO_PIN in software.
During the same callback, with AUDIO_PIN_ALT_AS_NEGATIVE set, the AUDIO_PIN_ALT is toggled inversely to AUDIO_PIN. This is useful for setups that drive a piezo from two pins (instead of one and Gnd).


### Testing Notes

While not an exhaustive list, the following table provides the scenarios that have been partially validated:

|                          | DAC basic          | DAC additive       | PWM hardware       | PWM software       |
| Atmega43U4               | :o:                | :o:                | :heavy_check_mark: | :o:                |
| STM32F103C8 (bluepill)   | :x:                | :x:                | :heavy_check_mark: | :heavy_check_mark: |
| STM32F303CCT6 (proton-c) | :heavy_check_mark: | :heavy_check_mark: | ?                  | :heavy_check_mark: |
| STM32F405VG              | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: |
| L0xx                     | :x: (no Tim8)      | ?                  | ?                  | ?                  |


:heavy_check_mark: : works and was tested
:o: : does not apply
:x: : not supported by MCU

*Other supported ChibiOS boards and/or pins may function, it will be highly chip and configuration dependent.*