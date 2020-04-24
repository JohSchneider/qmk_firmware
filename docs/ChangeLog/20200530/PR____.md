## update AUDIO-feature comment in avr-template

* audio_avr.c does not default to any pin, there has to be a #define XX_AUDIO in config.h at some level, for it to actually work
  otherwise the audio-code ends up cluttering the firmware, possibly breaking builds because the maximum allowed firmware size is exceeded
* these changes should remedy this
  * update the comment in the template, to point out the config.h setting, described in docs/feature_audio.md
  * disable audio on keyboards that have audio misconfigured=non-functional
  * update the already existing rules.mk comment on keyboards using the audio system (keeping intentional changes intact)
  * and updating the comment in existing rules.mk on keyboards that have audio disabled - to avoid confusion :-)
