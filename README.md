# Testing different `avr-gcc` versions with QMK

**tl;dr**: [Click HERE](avr-gcc.html) or click on the `avr-gcc.html` file if you don't like clicking links.

## Some background

[QMK Firmware](https://qmk.fm/) is a popular, open-source keyboard firmware, used extensively by the mechanical keyboard scene.  
Even though it supports ARM and RISC-V architectures now, keyboards based on AVR chips and development boards are still very common.  

A major drawbacks of AVR chips (compared to the other supported architectures) is their usually rather limited on-board memory.  
In practice, this can limit the number of features/capabilities/layers/etc a user can enable.  
Due to this, it's desired that source code should be compiled into as efficient machine code as possible to make good usage of the limited memory.  

For QMK, the `avr-gcc` compiler is used and it was quickly noticed when the [9.x release introduced a regression, resulting in significantly larger compiled binaries](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=91189).  
If my memory serves me well, at that point in time the AVR target was in a rough shape and understandable most development effort went into more popular targets.  
As such, the regression was not (fully) fixed in the following releases.  
Pretty early in the life of the `qmk` CLI command [a warning was introduced to recommend users downgrading to or staying at an 8.x release](https://github.com/qmk/qmk_firmware/commit/cf40c33c907c44753bb145b9f2d5107447422fbc#diff-d26098f964e0334af937ce9fee1a3dd91968877932f3370a6c7925964f9fe833R55).  
This warning is still shown to anyone who is using an `avr-gcc` version higher than 8.x.  

But it's been some time now, we're at release 13.2.0 now, so I've been wondering what's the case now.  
Is version 8.x still the best if you want the smallest possible AVR bin/hex file?  


## The tests

I've compiled the `default` keymap for every keyboard in [the official QMK repo](https://github.com/qmk/qmk_firmware) that has a microcontroller with a name that starts with `at`.  
At the time of writing, this means `2366` keyboards with one of these microcontrollers:  

* at90usb1286
* at90usb646
* atmega16u2
* atmega16u4
* atmega328p
* atmega32a
* atmega32u2
* atmega32u4

All tests were done against commit `dcc47ea31b3f4ef097a2fc677bdbb9b2560d905a`.  
The ID of the Arch container image was: `69f38d8f6347`.  

First, I patched out QMK's file-size sanity check. This is useful for normal usage, but I wanted to get the final `.hex` files even if they wouldn't fit the microcontroller.

    git apply no_size_check.patch

Used the QMK CLI's `mass-compile` feature to run the tests, 3 for each compiler:

1. Leaving it up to the default configuration of the keyboard/keymap if link-time optimisation ([LTO](https://en.wikipedia.org/wiki/Interprocedural_optimization)) is enabled:

       qmk mass-compile -km default -j 16 -c -f 'processor=at*'

2. Forcing LTO:

       qmk mass-compile -km default -j 16 -c -f 'processor=at*' -e LTO_ENABLE=yes

3. Force disabling LTO:

       qmk mass-compile -km default -j 16 -c -f 'processor=at*' -e LTO_ENABLE=no

After each run, I checked the sizes of each generated `.hex` files with the `ls` command and saved the output into simple text files:

    ls -l *.hex | awk '{print $9, $5}'

Cleanup between tests:

    qmk clean -a


## The results

The data was loaded from the simple text files into LibreOffice Calc for better visualisation.  
Each row contains the sizes in bytes for each compiler tested.  
**Green** fields are the smallest values. If more than one field has the same value, both are coloured.  
**Red** fields mean that the compiler failed to compile the code with the specific settings. There is also a build failure counter at the bottom of each column.  

What did I learn:

- Version 13.x performs pretty good.  
- If you want to go for the most size-optimised code possible, use 13.x with LTO enabled.  
- If you can't or don't want to use LTO, version 8.x still performs better.  
- `avr-libc` had [new release in 2022](https://github.com/avrdudes/avr-libc/tags) that has been flying under the radar of major distros ever since. Granted, the project has been largely dormant and the previous release/activity was in 2016.  
- Ubuntu is still packages the absolutely ancient 5.4.0 release [from 2016](https://gcc.gnu.org/gcc-5/). Not just in LTS, but in the latest stable too (23.10). Based on devel, the next stable will have 7.3.0, [from 2018](https://gcc.gnu.org/gcc-7/), so I guess that's progress.  


## Future ideas

In a few cases Arch's 13.2.0 generated smaller, marginally but still, files than my 13.2.0 on Fedora. I suspect this might be due to some differences between `avr-libc` 2.1.0 and 2.0.0.  
I might compile 2.1.0 for Fedora and redo the 13.2.0 tests.  

Gathering data from other OS/distros would be great.  
