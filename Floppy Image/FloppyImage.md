# Floppy Image - Forensics 600

>Did someone say encryption? - Casseus

## Challenge Description
**NOTE:**  This is an  _difficult_  challenge from another CTF where there is no write-up available, it does involve forensics but it's really here to present a challenge to a team that's not finding a lot of other things very challenging. We recommend you do something else first ;)

Good luck, and have fun.

We are given `files12_1.zip`
## Initial Observations
The zip file contains `Challenge.data` which, according to `file`, is an SMTP mail message. `binwalk` says there is a uuencoded file named `exam.img`. We can open the file in Thunderbird to see the message. 

```
To: "bill@bfiplx.com" <bill@bfiplx.com>
From: <ken@trendmicro.com> 
Subject: Call three sisters name in proper order at the right world.
Good day, Bill,

This is what I told you before.
I'm not sure what data is in this, but this really scramble your brain.
What you are seeing is not what you are thinking.
Have a fun !
daaah !!

Cheers,
Ken


begin 700 exam.img
MZSR05D%,2UE2244  @$!  +@ $ +\ D $@ "               I     %=9
M4D0@(" @(" @1D%4,3(@("#ZC,B.T([ CM@SP+SP_XOL^[0 L@#-$W,!]+@!
...
M%J*U"11Y!B&]N;-FSJC+)HM':SY$8MO-H2%'K\O]JQ3F1S,.<1SUKF<4,[P$
M\_EF:[._,Z\7VG<]=R-T-<\O'$/IV!9%0T.#FD9[/D"B^<O-ZR#.#0H]_XVD
 
end





=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=
Thence come maidens much knowing three from the hall which under
that tree stands;Urd hight the one,the second Verdandi,on a 
tablet they graved,Skuld the third;Laws they established,life 
allottedto the sons of men,destinies pronounced.
=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=

```

Uuencoding is the process of encoding a binary file using only plain-text characters, it was mainly used to send binary data over email back in the day. It's recognizable by the `begin` and `end` tags as well as the lines all beginning with M. We can run uudecode on the email to pull out `exam.img` a drive image file that is 1.44 MB big. 

## Looking Deeper
Since this is forensics, we should investigate the headers sent in the email, there might be clues there. 
```
Delivered-To: bill@bfiplx.com
Received: by 10.176.91.92 with SMTP id c2NhbmRpc2sgYToK;
        Fri, 5 May 2017 20:05:47 -0700 (PDT)
Return-Path: <ken@trendmicro.com>
Received: from mx.trendctf.net (mx2.trendctf.net [16.32.64.3])
        by mx.foobiz.com with ESMTP id ODA4NiBjaGVjawo.353.2017.05.05.20.05.46
        for bfiplx.com <exam.dat> text;
        Fri, 05 May 2017 20:05:47 -0700 (PDT)
Received: from [IPv6:::ffff:192.168.11.3] by mx.trendctf.net (Postfix) 
        with ESMTPA id YmZpcGx4LmNvbSA8IGV4YW0uZGF0Cg 
        for <bill@bfiplx.com>; Sat,6 May 2017 12:05:46 +0900 (JST)
MIME-Version: 1.0
To: "bill@bfiplx.com" <bill@bfiplx.com>
From: <ken@trendmicro.com>  =?ISO-8859-1?B?TE5LNDA2MCAvS05PV0VBUwo=?=
Subject: =?ISO-8859-1?B?Q2FsbCB0aHJlZSBzaXN0ZXJzIG5hbWUgaW4gcHJvcGVyCg==?=
  =?ISO-8859-1?B?IG9yZGVyIGF0IHRoZSByaWdodCB3b3JsZC4K?=
Date: Sat, 6 May 2017 12:05:39 +0900
Importance: normal
Content-Type: text/plain;charset="iso-8859-1"
Message-Id: <MiBoaWRkZW4gZmlsZXMK@mx.trendctf.net>
```
Looks like some base64 data, let's see what it says. 

`c2NhbmRpc2sgYToK` => `scandisk a:`
`ODA4NiBjaGVjawo`  => `8086 check`
`YmZpcGx4LmNvbSA8IGV4YW0uZGF0Cg` => `bfiplx.com < exam.dat`
`TE5LNDA2MCAvS05PV0VBUwo=` => `LNK4060 /KNOWEAS`
`Q2FsbCB0aHJlZSBzaXN0ZXJzIG5hbWUgaW4gcHJvcGVyCg==` => `Call three sisters name in proper`
`IG9yZGVyIGF0IHRoZSByaWdodCB3b3JsZC4K` => ` order at the right world.`
`MiBoaWRkZW4gZmlsZXMK` => `2 hidden files`
Lots of information here, `scandisk a:` suggests that the drive image is corrupt and we should find `2 hidden files`. `LNK4060 /KNOWEAS` is a linker flag that you would pass if you built a custom DOS-stub. 

Mounting `exam.img` reveals `norns.exe`, a 32-bit PE.

## Reversing 32-bit
We can open norns.exe in IDA as a 32-bit and have a look at the main function (sub_4004013). 

The first thing it checks is to see if we're on the A: drive. Then it checks if we're running using SysWOW64. There is a different subroutine that gets called if so. 

`sub_4003A16` - SysWOW64 "main"
`sub_40038A0` - Windows 32-bit "main"

Initially we will look at the 32-bit function. 

There's a bit going on here. First it makes sure we have a parameter passed in on the command line. Then it copies that parameter to a 16-byte buffer, initially set to "!This program ca". Then it will toUpper() that buffer. Then it calculates a checksum on the buffer, byte by byte. 

If the checksum comes out to `0xD86B` then it will do a strange performance check (might be something to baffle a debugger) and if it passes, it will create a file `A:\bfiplx.com`. Then it will select specific 8-byte arrays from memory based on the checksum and run some kind of decryption with the key being the 16-byte buffer that was checksummed. 

Doing a quick google search of the constants used in the decrypt will reveal that it is using TEA, or Tiny Encryption Algorithm. Once we know that we can write our own python script to try and figure out the decryption. We can pull out the correct encrypted strings since we know what the checksum should come out to be. 

```python
import struct

keyBase = "!THIS PROGRAM CA"

encBytes = [ [0x1C,0x99,0x9B,0xF4,0xB1,0xE8,0x7E,0xB7],
[0x0E,0x54,0x51,0x35,0x59,0x2D,0x58,0x2A],
[0xC2,0x8B,0x90,0x9C,0xF9,0xE9,0x1F,0x80],
[0x67,0x21,0x7E,0xF8,0xEA,0x32,0x55,0xBE],
[0xCB,0x57,0x96,0x3B,0xF6,0x6E,0x91,0xE4],
[0x6A,0x4E,0xB8,0x8D,0xC9,0xBB,0xE5,0x3E],
[0x14,0xE1,0xFE,0xFC,0x6C,0x7E,0x2C,0xB0],
[0x9A,0xB4,0x67,0x66,0x6F,0x4A,0xF8,0xE3]
]

checkSum=0xD86B


def getChecksum(s):
  csum=0
  for i in range(16):
    csum += ord(s[i])*5 ^ 0x5c5c | ord(s[i])
    csum &= 0xFFFFFFFF
  csum &= 0xFFFF
  return csum


def decrypt(testStr):
  for e in encBytes:
    subkey = 0xc6ef3720
    eb = b''.join([bytes([a]) for a in e])
    sb = testStr.encode('ascii')
    
    enc0,enc1 = struct.unpack("II",eb)
    key0, key1, key2, key3 = struct.unpack("IIII",sb)
    
    for i in range(32):
      enc1 -= ((subkey + enc0) ^ (key2 + 16*enc0) ^ (key3 + (enc0>>5)))&0xFFFFFFFF
      enc1 &= 0xFFFFFFFF
      tmp = subkey
      subkey += 0x61c88647
      result = ((tmp + enc1) ^ (key0 + 16*enc1) ^ (key1 + (enc1>>5)))&0xFFFFFFFF
      enc0 -= result
      enc0 &= 0xFFFFFFFF
    out.append(struct.pack("II",enc0,enc1))
  return b''.join(out)

key = "TESTKEY"

key = key+keyBase[len(key):]

if getChecksum(key) == checkSum:
  print(decrypt(key))

```

We can now look at the SysWOW32 function and we can see that it begins much the same way, copy argv[1] to a 16-byte buffer with "!This program ca" and toupper() it. Then it loads some functions from `ntdll.dll` and does a strange JUMPOUT.

## Heaven's Gate

If we look at the disassembly we can see something strange happens. the fs register is stored on the stack, then `0x2b` is moved into it. Then esp is saved to the stack and rounded off. Finally `0x33` is pushed and we get a call to `$+5` which leads to a `retf`. 

What's happening here is a technique called Heaven's Gate. It allows a program to change it's architecture during runtime. This way a 32-bit process running on a 64-bit copy of Windows can run 64-bit code and make 64-bit system calls. It's commonly used in malware to disguise execution paths or obfuscate code. 

## Reversing 64-bit
To properly reverse the Heaven's Gate code we can open the binary in IDA64 and add the 64-bit code as a 64-bit segment. Once we're in the segment we can see that it seems to be doing the same thing as the 32-bit code, but with different data and no checksums. 

## Reversing 16-bit
The mention of "3 maidens" and "3 worlds" means there must be another head to this binary. The `LNK4060 /KNOWEAS` clue makes it obvious that there is code in the DOS Stub. By opening the binary is 16-bit mode in IDA we can get to the stub code. 

We can walk through the main function (sub_10018C) and see that it's doing the same thing as the other versions. First it checks it's on the A drive, then does the checksum on the argv[1] data. The checksum should be equal to `0xD923`.

## Putting it Together
We have 1 program that operates in 3 modes and needs 3 passwords. The original email said to "Call three sisters name in proper order at the right world." Maybe their name's are the passwords with `Urd` being the 16-bit one, `Verdandi` being the 32-bit one, and `Skuld` being the 64-bit one. We know that the output is a `.com` file, which is a very basic executable. There is no file structure, it's just raw machine code that DOS loads into memory. There is a good way to identify DOS code, since so much interaction happened with interrupts we can look for `INT 0x21` instructions which compile down to `CD 21`.

We can use DOSBOX to run the 16-bit code, and VMs to run the other code. We will also have to patch out the performance check to get the programs to properly run. Once we pass `Urd` we get out `Ok (1/3)` and `bfiplx.COM` is created. Once we do this for the 32 and 64 bit versions we will have 3 com files that need to be concatenated to one executable. 

We know from the hints that we need to run bfiplx with the following syntax `bfiplx.com < exam.dat`. Now we just need `exam.dat`

## Looking for the Final Piece
There's a helpful Linux utility called `testdisk` we can use to recover files from drive images. We can open the exam.img file and use the "Advanced" tool to analyze it for corrupt files, and then "List" to show any files it found. We'll find `_XAM.DAT`, which looks like a deleted file. Testdisk can recover the file. 

## Final Piece
Now we just run the com file with the _xam.dat file to get the flag. `TMCTF{Atropos}`
