# Secure SRAM - Reverse Engineering 500


>Are you guys CODEX?
I can't tell if you're trolling, honestly.
-pnill

## Challenge Description

After entering the password you obtained from the network interface into the boot process, we were able to proceed past an initial boot prompt on the UART header of the obtained hardware.

Now it's allowing us to execute a custom payload, maybe this is how they deployed their botnet code? It seems the botnet code is unencrypted and unsigned but during the boot-process we noticed the mention to some form of 'secure keys' can you retrieve these keys? We believe they might be in SRAM based on the boot log.

We were able to compile a payload using the simduino project found here: https://github.com/buserror/simavr/blob/master/examples/board_simduino/atmega328p_dummy_blinky.c

The main difference is that we had to specify atmega2560 instead of the list 328p we attached our test payload and script but we haven't been able to do anything other than getting it to print text.

Connect here: nc 172.18.10.105 32557

## Initial Observations
We were given the following files:

 - <span>do.py</span> - A helper script to run the simduino emulator
 - user.c - A firmware source that should compile and run
 - user.bin - The compiled form of user.c
 - simduino.elf - An emulator for the hardware

<span>do.py</span>
```python
#!/usr/bin/env python
from pwn import *
import sys

#context.log_level = 'debug'
p = process(['obj-x86_64-linux-gnu/simduino.elf', 'bldr.hex'])

p.recvuntil("What's the length of your payload?")

payload = open(sys.argv[1], 'rb').read()

p.writeline(str(len(payload)))
p.write(payload)

p.interactive()
```

user.c
```c
/*
	atmega328p_dummy_blinky.c

	Copyright 2008, 2009 Michel Pollet <buserror@gmail.com>

 	This file is part of simavr.

	simavr is free software: you can redistribute it and/or modify
	it under the terms of the GNU General Public License as published by
	the Free Software Foundation, either version 3 of the License, or
	(at your option) any later version.

	simavr is distributed in the hope that it will be useful,
	but WITHOUT ANY WARRANTY; without even the implied warranty of
	MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
	GNU General Public License for more details.

	You should have received a copy of the GNU General Public License
	along with simavr.  If not, see <http://www.gnu.org/licenses/>.
 */


#ifdef F_CPU
#undef F_CPU
#endif

#define F_CPU 16000000

#include <avr/io.h>
#include <stdio.h>
#include <avr/interrupt.h>
#include <avr/eeprom.h>
#include <avr/sleep.h>

/*
 * This demonstrate how to use the avr_mcu_section.h file
 * The macro adds a section to the ELF file with useful
 * information for the simulator
 */
#include "avr_mcu_section.h"
AVR_MCU(F_CPU, "atmega2560");

static int uart_putchar(char c, FILE *stream) {
  //if (c == '\n')
  //  uart_putchar('\r', stream);
  loop_until_bit_is_set(UCSR0A, UDRE0);
  UDR0 = c;
  return 0;
}

static FILE mystdout = FDEV_SETUP_STREAM(uart_putchar, NULL,
                                         _FDEV_SETUP_WRITE);


int main()
{
	stdout = &mystdout;

	printf("Bootloader properly programmed, and ran me! Huzzah!\n");

	while (1) {};
}
```

Connecting to the server, we get asked for the length of a payload, then we can feed a binary to the server through stdin. We can modify the emulator helper code to upload user.bin to the server instead of to the local emulator.

```python
#!/usr/bin/env python
from pwn import *
import sys

#context.log_level = 'debug'
p = remote('172.18.10.105', 32557)

p.recvuntil("What's the length of your payload?")

payload = open('user.bin', 'rb').read()

p.writeline(str(len(payload)))
p.write(payload)

p.interactive()

```

Output
```
$ python do.py
Decrypting secret keys...
Hashing payload with secret keys...
Jumping to user application...
Bootloader properly programmed, and ran me! Huzzah!
```
Looks good. Now to try and get our own code compiling. 

## Setting Up the Build Environment
The target is running on an ATMega2560 according to the challenge description. This means we will have to install some cross compile tools, specifically avr-gcc. Confusingly, the avr-gcc is installed in the gcc-avr package on Ubuntu. 

Once we have the build tools we will want to grab the simavr project linked in the description and make it. This will give us the avr libraries we need to cross compile against. We know that the user.c code is based on the code in `simavr\examples\board_simduino` but with the gcc flags changed to compile for ATMega2560. One quick way to get the relevant gcc commands is to modify the source slightly, then run make with `make -n` to see the commands. We can then copy and modify the commands to build our own firmware. 

```
make[2]: Entering directory '/mnt/d/ctf/ray_finals/secure-sram/simavr/examples/board_simduino'
echo AVR-CC atmega328p_dummy_blinky.c
part=atmega328p_dummy_blinky.c ; part=${part/_*}; \
avr-gcc -Wall -gdwarf-2 -Os -std=gnu99 \
                -mmcu=$part \
                -DF_CPU=8000000 \
                -fno-inline-small-functions \
                -ffunction-sections -fdata-sections \
                -Wl,--relax,--gc-sections \
                -Wl,--undefined=_mmcu,--section-start=.mmcu=0x910000 \
                -I../simavr/sim/avr -I../../simavr/sim/avr \
                atmega328p_dummy_blinky.c -o atmega328p_dummy_blinky.axf
avr-size atmega328p_dummy_blinky.axf|sed '1d'
avr-objcopy -j .text -j .data -j .eeprom -O ihex atmega328p_dummy_blinky.axf atmega328p_dummy_blinky.hex
echo simduino done
rm atmega328p_dummy_blinky.axf
make[2]: Leaving directory '/mnt/d/ctf/ray_finals/secure-sram/simavr/examples/board_simduino'
```

We can copy the avr-* commands to a separate .sh file, change `-mmcu=$part` to `-mmcu=atmega2560`, and change the -I to point to your simavr/sim/avr folders. Also change the references to atmega328p_dummy_blinky to user so we compile user.c. One last thing is to change the avr-objcopy -O to `binary` from `ihex`. 

<span>build.sh</span>
```sh
avr-gcc -Wall -gdwarf-2 -Os -std=gnu99 \
                -mmcu=atmega2560 \
                -DF_CPU=8000000 \
                -fno-inline-small-functions \
                -ffunction-sections -fdata-sections \
                -Wl,--relax,--gc-sections \
                -Wl,--undefined=_mmcu,--section-start=.mmcu=0x910000 \
                -I./simavr/simavr/sim/avr \
                user.c -o user.axf
avr-size user.axf|sed '1d'
avr-objcopy -j .text -j .data -j .eeprom -O binary user.axf user.bin
rm user.axf
```
Running this will give us a user.bin that we can upload to the server with our modified python script.

```
$ build.sh
    504      66       6     576     240 user.axf
$ python3.8 ./do.py
Decrypting secret keys...
Hashing payload with secret keys...
Jumping to user application...
Bootloader properly programmed, and ran me! Huzzah!
```
 
We now have a functioning build environment and can modify the user.c as we please. 
## The Exploit

We know that we're looking for something in the RAM of the console so we should modify user.c to print out some RAM values with printf().

```c
/*
	atmega328p_dummy_blinky.c

	Copyright 2008, 2009 Michel Pollet <buserror@gmail.com>

 	This file is part of simavr.

	simavr is free software: you can redistribute it and/or modify
	it under the terms of the GNU General Public License as published by
	the Free Software Foundation, either version 3 of the License, or
	(at your option) any later version.

	simavr is distributed in the hope that it will be useful,
	but WITHOUT ANY WARRANTY; without even the implied warranty of
	MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
	GNU General Public License for more details.

	You should have received a copy of the GNU General Public License
	along with simavr.  If not, see <http://www.gnu.org/licenses/>.
 */


#ifdef F_CPU
#undef F_CPU
#endif

#define F_CPU 16000000

#include <avr/io.h>
#include <stdio.h>
#include <ctype.h>
#include <avr/interrupt.h>
#include <avr/eeprom.h>
#include <avr/sleep.h>
#include <util/delay.h>

/*
 * This demonstrate how to use the avr_mcu_section.h file
 * The macro adds a section to the ELF file with useful
 * information for the simulator
 */
#include "avr_mcu_section.h"
AVR_MCU(F_CPU, "atmega2560");

static int uart_putchar(char c, FILE *stream) {
  //if (c == '\n')
  //  uart_putchar('\r', stream);
  loop_until_bit_is_set(UCSR0A, UDRE0);
  UDR0 = c;
  return 0;
}

static FILE mystdout = FDEV_SETUP_STREAM(uart_putchar, NULL,
                                         _FDEV_SETUP_WRITE);


int main()
{
	stdout = &mystdout;

	printf("Bootloader properly programmed, and ran me! Huzzah!\n");
	for(uint32_t i=0;i<0x1000;i+=0x100)
	{
		printf("| %04x |\t",i);
		uint8_t* p = (uint8_t*)i;
		for(uint16_t j=0;j<0x100;j++)
			if(*(p+j) >= 0x20 && *(p+j) < 0x80)
				printf("%c", *(p+j));
		_delay_ms(10);
		printf("\n");
	}
	while (1) {};
}

```

This will loop through the first 0x1000 bytes of memory and print the ASCII characters 0x100 bytes to a line

```
| 0000 |    B0!*!`
| 0100 |
| 0200 |    Bootloader properly programmed, and ran me! Huzzah!| %04x |g to user application...User payload failed to run...zv&:Z/BH_>}B;ccSJ[rtYmtfC_/b>q4q'?#.t@:yadQZ^S8/kA{[kp;^SBeA<e>C2ujr=s,f?&'#x.{a,aJ]rNUeZfxT}<BjC/vcr<%u$Wa.y<FKd,MbfM]
| 0300 |    ]'AMJ(+jRH/+Z{UUrk%SAyYv+9A[_fd8b/s9L(t6t4:>~azj+y.B}*#x/L.pyN_M,2T[~d=4m}T#rDjU}4CjUQzZhqF6]+B,!'wfh-YxeL>jPqg*#DYu)5z~TJ8xC4w+f))hq#aC>h(8&u~=AC&Aq_MN7mar@]xBYRjNSsYcb!6Y}ffSW#PH[-!vZ.qM%^4*jL[3vf:$P^weP:[frA*VZ}TD*CZ!63msSyXK^f>RV@>nQ+B]GtJsa%cCVvV>=qQ$
| 0400 |    5}`X{^N#f'fE[.)8jTk#;_fjdEKLA'eff8'aFxyx?we@Ds'ea23HLY.];{u~fuBAN-afGQ)p_VP<<+Z#dP#w<+f>3V45$QyrnG.NGpxC'>[BDf6Ja],Pnj~r~..fT-.J7At(+M3W`A'Dk_x$fr4+@Hp8~)=%Rw]2{ck>KtF?2]Z[:UvkvCUEa9t:d.@+?ku`d]K;G[':_XkU_Z}ttN(AVyD(2[U%c`./R$H8y%[[}D'NT}hW`Z{2XYEPv5AL}ku9
| 0500 |    {5fYGW+]cvR<L[fX=qkfnBzXVQ[QPC(Lng=VA$~qw?e[wgk!2y>!e5Eh7+GC3g$!tA_<AwW]~b{Lc#}fpM9.Cx'(T9Pn52Zb-xh3VY#_$S3!{T]MGrZZB2~+!*<xp:BzSY]-z3zK;YW>tsf<?H!wdA=$>CQU[22F'f(n.x_,6hB=4K.ZmC/&v*;Ff#pD;t=9LXm'`nYs7NKCPJ9sT`Pzt7qYBGhv-WG9PfVNGW*w!$?cJghuZtNGjYrVd+Gs-*_m
| 0600 |    *}$[2z/{>y5^3PCq@DY]=btH-@&Rzf'~4zjC-4QYXFM+GvGX/Vx{=;^Z>CUL26xkg58CNaC`@-C<KQf!:e#/3h[,.x}cj@#ccJQrg)n}=<MUa'P.TJ:4qLEW>a>[,m%nPuS@97|<>&p1/VKg^+mxR%*%je<l+lY N'`=H1^z<NAyEAfP4AxT-q-+i+}\$FWiLm/mu8
| 0700 |    u/SY5H_>*kn>\x7f<.x:8/xM6b]{6MN$0B'vX9m+vcUgy)#-z,n3c@<&'Y#$;{zbSA<lGVq0"4Rn2w}B?`,Z5RHHNGY$Zj;3V6y7w{68 +=&`9';[lE"dlgH/\0kK'?~E?y14Vcl.Ue=mtrW$6U#inXiShp,$,%9c==q\ha
| 0800 |    E+xqRj7=MCANi~rL+5/((rY'^&! 2wv1qr+s7ML-5Ca<I<u&N'<bZn%%z6e4OutS]dkzO;z8*}7wa[U6d~5MD48mo2)B__AN/!1]=K%|i<a$(h;xBq3Jcy\*I0yaSX9&9'?9kCj;,p.;HH<@2n*ab+[P63:;rK+$ddg%v\</u
| 0900 |    Oj3\x7f5VUva<Ej0KJf#$"u(pk{KX^ed&ZVBXky2g7^Icm)4atee3Aj"'/wV%.cdYIE:(9ll`U7k+?f?xXCnjHz\2*>)#D"2_+JFZnmY*S|{~s\R~<q(kuzXvyj ptBggoQW"5'vX=?{7(kJ%i]Yc;z4xO/ l2:NEC]\x7f.>!G(/%SNJ1
| 0a00 |    O=E0d]fSj(tq=x)8Fyx#1Tg(\;1:KZ+gm60k[pesDFSd(83{2r=i7vy|}8;w')'%$jDmly0arCxqUgMXtVfD_*KSB3tHG+dtn%zP-7h#Zd6gbJBsRh3n-Mha?n=tDp_-B+uAHpTvRJW-P72Puy3QXT9j%LzDkkctVZqg^f9H+UmnH%&WUH+KS=VAQ6U9R%ckL94Gk#hQFSKA8c
| 0b00 |    f_eZRnWF4x3g_g5S$kJxAN=Xygt+FN@NZJ@X8K=cP94BVgEzW7$$a2fMh-_YTvbpw2fZ#7T+wZMD^yY^*Y6utPVsv#yLJ=2YQJN#t6uT%5?3=4ZHSSx!*RG^H8akJXTSnB*4aax?d8L$LrR*NMk@u_FkhUR@rgm^&NzyyWh3JQDYzZN8umqN64_7*FjVf^Byn_aMsJM3YC=aEC33Vu3j@&E?@9Z^MKSa?aWPvVGhwkq?@P68x*-knnbM?Fz5Kk2L
| 0c00 |    -9EpnfKPV6R4MQxPcjyjT%JE-*hpnRyn$McN%U8a&_Y3Wbr3yQThCAcF*!jv7XqSrdbZ2wgjxjF^fEW7AuYAY5Rq!#=AwE$utGJ%=?CX+9!zneKJV*MA5?sDtJPea!cFsuL&8A^GxV9+Yvaf9ckjLHuvQ3+3sFR+&jVuMQBC34fP94Rk^+n=m+6KWd#RcAam-m2ZXHtbfJqsfNZKM%k+mthX9unmgn9gndzVSD9p&TwxgGS%+q_VSN!7SCL@!
| 0d00 |    WZL4_UP8b*H5#$5-g#2=Ah-$$Y_Guy$x3C7%?c3%#Xdce3^f%3JKK4FRkZ!TFnwRrPDkQSYr6=Sq@qkLT+C^ZZJG!nmn5xWY^YcM-_5CM34?x36D2Sr?*N^KqqEKbFskfpPSgt&YwX@+kQa_9qze&Lt=knXERTXFLAG{4VR_P4YL04D$_DUMP_R4M!!}L!Y7#!3nmJQ8SfhL^_tx!sV==Nu-TJ#v?c9+xgUZTKGAm39dg2zAz5d%Zpf-qfyAxD=g
| 0e00 |    ?nne@=SwJK&X*_vG!XLTB!J=Khpk9^qK2@V9&G@CPNfnzJJ!U#*@wR5=#kM=Z?#R&Cvpmw4Q!kGYUbsVqNv8QWCq_=Nuc2?CdmkRTDJTvdf$KYWxX8eKGWd+H9244VUgMXtVfD_*KSB3
| 0f00 |
```

Look through this output or pipe it to grep to see the flag `RTXFLAG{4VR_P4YL04D$_DUMP_R4M!!}`
