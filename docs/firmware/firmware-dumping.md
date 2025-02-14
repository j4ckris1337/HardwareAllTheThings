# Firmware Dumping

## Summary

* [Send a new firmware into the microcontroller](#send-a-new-firmware-into-the-microcontroller)
* [Dump firmware using debug port](#dump-firmware-using-debug-port)
* [Convert ihex to elf](#convert-ihex-to-elf)
* [Over-the-air updates](#over-the-air-updates)
* [Explore firmware](#explore-firmware)
* [Type of firmware](#type-of-firmware)
* [Check entropy](#check-entropy)
* [Unsquashfs](#unsquashfs)
* [Encrypted firmware](#encrypted-firmware)


## Send a new firmware into the microcontroller

* Using [avrdudes/avrdude](https://github.com/avrdudes/avrdude)
    ```powershell
    # send raw data firmware
    $ avrdude -p m328p -c usbasp -P /dev/ttyUSB0 -b 9600 -U flash:w:flash_raw.bin

    # send ihex firmware
    $ avrdude -c usbasp -p m328p -F -U flash:r:dump.hex:i

    # default
    $ avrdude -c usbasp -p m328p -C /etc/avrdude.conf -U flash:w:hardcodedPassword.ino.arduino_standard.hex
    ```
* Using [raspberrypi/picotool](https://github.com/raspberrypi/picotool)
    ```ps1
    # extension indicates the type (bin, uf2)
    picotool load firmware.bin
    ```


## Dump firmware using debug port

* Using [avrdudes/avrdude](https://github.com/avrdudes/avrdude)
    ```powershell
    $ avrdude -p m328p -c usbasp -P /dev/ttyUSB0 -b 9600 -U flash:r:flash_raw.bin:r
    $ avrdude -p m328p -c arduino -P /dev/ttyACM0 -b 115200 -U flash:r:flash_raw.bin:r
    $ avrdude -p atmega328p -c arduino -P/dev/ttyACM0 -b 115200 -D -U flash:r:program.bin:r -F -v 
    ```

* Using [openocd-org/openocd](https://github.com/openocd-org/openocd)
    * Determine code space in the microcontroller (for example nRF51822 - Micro:bit), save as `dump_img.cfg`:
        ```powershell
        init
        reset init
        halt
        dump_image image.bin 0x00000000 0x00040000
        exit
        ```
    * Dump with openocd
        ```powershell
        sudo openocd -f /home/maki/tools/hardware/openocd/tcl/interface/stlink-v2-1.cfg -f /home/maki/tools/hardware/openocd/tcl/target/nrf51.cfg -f dump_fw.cfg
        ```
* Using [raspberrypi/picotool](https://github.com/raspberrypi/picotool)
    * Build PicoTool, you will need the pico-sdk
        ```ps1
        # PicoSDK
        git clone https://github.com/raspberrypi/pico-sdk.git
        cd pico-sdk
        git submodule update --init

        # Picotool
        cd ..
        git clone https://github.com/raspberrypi/picotool.git
        cd picotool
        mkdir build
        cd build
        cmake -DPICO_SDK_PATH=../pico-sdk ..
        make
        ```
    * Dump the program or the whole flash memory
        ```ps1
        sudo ./picotool save -F /tmp/out.bin
        Saving file: [==============================]  100%
        Wrote 73312 bytes to /tmp/out.bin

        sudo ./picotool save --all -F /tmp/out2.bin
        Saving file: [==============================]  100%
        Wrote 2097152 bytes to /tmp/out2.bin
        ```


## Convert ihex to elf

> The Intel HEX is a transitional file format for microcontrollers, (E)PROMs, and other devices. The documentation states that HEXs can be converted to binary files and programmed into a configuration device.

Each line in the ihex file starts with :

* a colon :
* followed by ONE BYTE = record length
* followed by TWO BYTES = offset to load
* followed by ONE BYTE = Record Type
* Last BYTE in the line = Checksum

Convert .hex(ihex format) to .elf file with `avr-objcopy` or with an online tool [http://matrixstorm.com](http://matrixstorm.com/avr/hextobin/ihexconverter.html)

```powershell
$ avr-objcopy -I ihex -O elf32-avr dump.hex dump.elf
# or 
$ objcopy -I ihex chest.hex -O binary chest.bin ; xxd chest.bin
```

Alternative with Python `bincopy`

```python
import bincopy
import sys

f = bincopy.BinFile()
f.add_ihex_file(sys.argv[1])
print(f.as_binary())
```

Quick strings on .hex

```powershell
cat defaultPassword.ino.arduino_standard.hex | tr -d ":" | tr -d "\n" | xxd -r -p  | strings 
```

Inspect the assembly with `avr-objdump -m avr -D chest.hex`.\
Emulate : `qemu-system-avr -S -s -nographic -serial tcp::5678,server=on,wait=off -machine uno -bios chest.bin`


## Over-the-air updates

TODO


## Explore firmware

```powershell
$ binwalk -Me file.bin
$ strings file.bin

$ strings -e l file.bin
The strings -e flag specifies the encoding of the characters. -el specifies little-endian characters 16-bits wide (e.g. UTF-16)

$ strings -tx file.bin
The -t flag will return the offset of the string within the file. -tx will return it in hex format, T-to in octal and -td in decimal. 

$ dd if=firmware.bin of=firmware.chunk bs=1 skip=$((0x200)) count=$((0x400-0x200))
If we wanted to run it a little faster, we could increase the block size:
$ dd if=firmware.bin of=firmware.chunk bs=$((0x100)) skip=$((0x200/0x100)) count=$(((0x400-0x200)/0x100))

$ binwalk -Y dump.elf 
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
3708          0xE7C           ARM executable code, 16-bit (Thumb), little endian, at least 522 valid instructions
```


## Type of firmware

* SREC - Motorola S-Record : All S-record file lines start with a capital S.
* Intel HEX lines all start with a colon.
* TI-TXT is a Texas Instruments format, usually for the MSP430 series. Memory addresses are prepended with an **@**, and data is represented in hex.
* Raw NAND dumps


## Check entropy

High entropy = probably encrypted (or compressed). Low entropy = probably not

```powershell
$ binwalk -E fw
```


## Unsquashfs

```powershell
sudo unsquashfs -f -d /media/seagate /tmp/file.squashfs
```


## Encrypted firmware

![](https://images.squarespace-cdn.com/content/v1/5894c269e4fcb5e65a1ed623/1581004558438-UJV08PX8O5NVAQ6Z8HXI/ke17ZwdGBToddI8pDm48kHSRIhhjdVQ3NosuzDMrTulZw-zPPgdn4jUwVcJE1ZvWQUxwkmyExglNqGp0IvTJZamWLI2zvYWH8K3-s\_4yszcp2ryTI0HqTOaaUohrI8PIYASqlw8FVQsXpiBs096GedrrOfpwzeSClfgzB41Jweo/Picture2.png?format=1000w)

* [MINDSHARE: DEALING WITH ENCRYPTED ROUTER FIRMWARE](https://www.zerodayinitiative.com/blog/2020/2/6/mindshare-dealing-with-encrypted-router-firmware)
