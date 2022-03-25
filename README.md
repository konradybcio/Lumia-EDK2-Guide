# Guide to porting EDK2 UEFI to Spec B Lumias
----------------

[![Video of EDK II booting on L535](http://img.youtube.com/vi/8Ag1vEQ6TYw/0.jpg)](http://www.youtube.com/watch?v=8Ag1vEQ6TYw)


## Resources:
```
Littlekernel sauce: https://github.com/konradybcio/lk

Lumia930Pkg: https://github.com/rickliu2000/Lumia930Pkg

Lumia535Pkg: https://github.com/konradybcio/Lumia535Pkg

EDK2: https://github.com/tianocore/edk2 (confirmed working as of HEAD a2c3bf1f2f991614ac97ddcf4b31742e4366c3a5)

BootShim: https://github.com/imbushuo/boot-shim

Cross-compilation toolchain (or just a toolchain if you're compiling on arm 🤷‍): Literally any GCC >=5, but you will probably want this one: https://releases.linaro.org/components/toolchain/binaries/7.1-2017.08/arm-linux-gnueabihf/gcc-linaro-7.1.1-2017.08-x86_64_arm-linux-gnueabihf.tar.xz

Platform reference: https://github.com/efidroid/projectmanagement/wiki/%5BReference%5D-Chipsets

If you input hex equations into Google search, it will spit out the answer. Just FIY.
```

## Let's get it started!



0\. Spec A Lumias seem to be even more cursed than I thought. They don't expose uefiplat.cfg, making it harder to make a proper memory map.

 ## Setup:
 
0a. For this guide you'll need a setup consisting of Linux (Ubuntu is used here as an example) + Windows (WPInternals and such are not available on Linux sadly).

Use WSL2, it works pretty well, and gets the job done.



 Open a terminal window, most thins will be done here. Below are all the commands you have to run to get EDK2 downloaded.
    
    sudo apt-get install build-essential uuid-dev iasl git gcc-5 nasm python3-distutils
    git clone https://github.com/tianocore/edk2
    cd edk2
    git checkout stable/202011
    git submodule update --init
    make -C BaseTools
    . edksetup.sh

After setting up EDK2, you're gonna have to get your LumiaPkg cloned into the EDK2 directory so you can build EDK2.

For Lumia 535 you would do `git clone https://github.com/konradybcio/Lumia535Pkg`



Below lies the tutorial to porting EDK2 to a new device.
**Click [here](https://github.com/konradybcio/Lumia-EDK2-Guide/blob/master/README.md#building) to skip to building in case you just want to build.**

## Porting

1\. Check your device's SoC and the version of MDP that it uses. To do this, get LK source and check the file `platform/msm_shared/rules.mk`. Then look for the SoC that your phone is based on and under its name, there should be an include like `$(LOCAL_DIR)/mdp4.o \`.




2\. Follow the proper steps for your specific MDP version:




`REG_MDP(offset) = MDP_BASE + offset`


For example (msm8610):


REG_MDP(0x90008) = 0xFD900000 + 0x90008 = 0xFD990008 <- you want this


### MDP3:


- Cherry-pick [this commit](https://github.com/konradybcio/Lumia535Pkg/commit/2b1bf33d2d47821d64d379f4df68218160f2e56a) and check MDP_DMA_P_BUF_ADDR and MDP_BASE in `platform/msmXYZA/include/platform/iomap.h` to make sure your address is correct.


### MDP4:


- This has not yet been proven to work 100%, but *should* work: 

- Cherry-pick the MDP3 commit and make sure to change your addresses.


### MDP5:


- Cherry-pick [this commit](https://github.com/rickliu2000/Lumia930Pkg/commit/f4a65646e545997be69b91c80a046ce6b7efcd7a) and check whether your addresses are corect. 




3\. Edit the memory map (`Include/Configuration/DeviceMemoryMap.h`). First off, retrieve your uefiplat.cfg. This can be found on your phone's UEFI partition (NOT EFIESP!) with [UefiTool](https://github.com/LongSoft/UEFITool).


- 0x40000000 is 1 GiB, 

- 0x80000000 is 2 GiB,

- 0x20000000 is 512 MiB,


..and so on, you can calculate it [with this handy calculator](https://ss64.com/convert.html)


First off, open `LumiaXXXPkg/Include/Configuration/DeviceMemoryMap.h` and make all the bottom memory into a big chunk named "New HLOS". You don't really have to care about what's in there Just try not to include TZ Apps in here, this might end bad.


Then, reserve your framebuffer memory. Address is defined in your .dec file, and size should match the one from your uefiplat.cfg.




Open the .dsc file and set `PcdSystemMemorySize` to the correct value, same with `PcdCoreCount`. `PcdPreAllocatedMemorySize` should be equal to (your total memory - 0x3300000).


Adjust `PcdMipiFrameBuffer[...]` parameters. If your screen resolution is lower than around 1280x720, you will need to cherry-pick [this commit](https://github.com/konradybcio/Lumia535Pkg/commit/b6049f0cb113ea07e09f49d2bdbbf62c3559aec3).


`PcdSetupConOutRow` should be your `PcdSetupVideoVerticalResolution` divided by 8 and `PcdSetupConOutColumn` should be your `PcdSetupVideoHorizontalResolution` divided by 19.




Open the .dec file and make sure that values under `PcdMipiFrameBuffer[...]` are correct. `PcdPreAllocatedMemorySize` should be (your total memory - 0x3300000) and PcdUefiMemPoolSize should stay at 0x3300000.


Open FvWrapper.ld and change the addresses here to (your total mem - 0x3400000).


# Getting some clocks set



Copy clock.h, iomap.h and irq.h contents from `platform/msmXYZA/include/platform` from LK, remove all #include statements and replace them with the ones that were there before (from LumiaXXXPkg).


Copy contents of `platform/msmXYZA/msmXYZA-clock.c` into `Library/QcomPlatformClockInitLib/msm8974-clock.c (or msm8612-clock.c if you're basing on 535Pkg)`, and of course replace include statements with the ones from LumiaXXXPkg.


When copying clocks from msmXYZA to msm8974/msm8612 always run "find and replace" and replace your SoC name with msm8974 (so that build system doesn't complain). You can easily change these later, or do it in a clean way in the first place. Sed is your friend.



Now it's finally time to build.



## Building



If you're on Linux, use bash. `zsh` sadly won't work...


### The Blue and the Fruity platforms


I've never built EDK2 on Windows or macOS. If you know how to do this, please update this guide here.



### Linux

Install crossbuild-essentials-arm for building EDK2 for ARM.

Run `source edksetup.sh`, `export GCC5_ARM_PREFIX=arm-linux-gnueabihf-` (with the dash at the end)


Run `build -a ARM -p LumiaXXXPkg/LumiaXXX.dsc -t GCC5 -j$(nproc) -s -n 0` (replace XXX with the device you are building for, for example for a Lumia 535 you would replace it with `Lumia535Pkg/Lumia535.dsc`)


Go to /your/edk2/path/Build/LumiaXXX-ARM/DEBUG_GCC5/FV/ and run `/path/to/your/toolchain/bin/arm-linux-gnueabihf-objcopy -I binary -O elf32-littlearm --binary-architecture arm MSMxyza_EFI.fd MSMxyza_EFI.fd.elf && /path/to/your/toolchain/bin/arm-linux-gnueabihf-ld MSMxyza_EFI.fd.elf -T ../../../../LumiaXXXPkg/FvWrapper.ld -o emmc_appsboot.mbn` (xyza is from your LumiaXXX.fd file)




Now you're basically done. Get some fresh copy of @imbushuo's boot-shim and install it, hoping it'll work. Now spend some nights writing Linux drivers for obscure SoCs like msm8x1x.



##### Running sed recursively to adjust the names to the proper values would also be a nice thing.

--------------

Credits:


- @imbushuo for making Lumia950XLPkg, PrimeG2Pkg and boot-shim and answering some questions,

- @rickliu2000 for making Lumia930Pkg,

- CAF for releasing lk sources,

- @konradybcio (me) for spending a few minutes writing this guide up.

- @sonic011gamer for spending a few minutes getting into detail.
