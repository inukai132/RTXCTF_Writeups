# Hash Service - Crypto 400

>Are we being monitored? If so, that was my cat running across the keyboard. - rstreumpf

## Challenge Description
After you made it through those layers we found this service listening on the attacker's network, it seems to allow for the hashing of files or getting system information but we haven't found out anything useful we can do with it... maybe you can make it do something interesting. 

Connect here: 

nc 172.18.10.105 38483

## Initial Observations
Connecting to the hash service and typing `help` shows us we have 5 options
* hash position size - Use this to hash data on the system
* verify_hash hash - Verifies an entered hash against data
* load_file filename - Loads a file into memory.
* help - Use this to display this text
* info - Use this to get information about the system

`info` shows up more information about the hash_service, including a file list:
```
Listing files in the application directory:


total 68
drwxr-xr-x   17 nobody nogroup 4096 Aug 28 03:28 .
drwxr-xr-x   17 nobody nogroup 4096 Aug 28 03:28 ..
lrwxrwxrwx    1 nobody nogroup    7 Jun  9 07:27 bin -> usr/bin
drwxr-xr-x    2 nobody nogroup 4096 Apr 15  2020 boot
drwxr-xr-x    2 nobody nogroup 4096 Jun  9 07:31 dev
drwxr-xr-x   30 nobody nogroup 4096 Aug 28 03:14 etc
drwxr-xr-x    3 nobody nogroup 4096 Aug 28 03:28 home
lrwxrwxrwx    1 nobody nogroup    7 Jun  9 07:27 lib -> usr/lib
lrwxrwxrwx    1 nobody nogroup    9 Jun  9 07:27 lib32 -> usr/lib32
lrwxrwxrwx    1 nobody nogroup    9 Jun  9 07:27 lib64 -> usr/lib64
lrwxrwxrwx    1 nobody nogroup   10 Jun  9 07:27 libx32 -> usr/libx32
drwxr-xr-x    2 nobody nogroup 4096 Jun  9 07:27 media
drwxr-xr-x    2 nobody nogroup 4096 Jun  9 07:27 mnt
-rwxrwxr-x    1 nobody nogroup   20 Aug 28 03:24 mysupersecretflagfile
drwxr-xr-x    2 nobody nogroup 4096 Jun  9 07:27 opt
dr-xr-xr-x 1101 nobody nogroup    0 Sep 20 05:54 proc
drwx------    2 nobody nogroup 4096 Jun  9 07:31 root
drwxr-xr-x    5 nobody nogroup 4096 Jun  9 07:31 run
lrwxrwxrwx    1 nobody nogroup    8 Jun  9 07:27 sbin -> usr/sbin
drwxr-xr-x    2 nobody nogroup 4096 Jun  9 07:27 srv
drwxr-xr-x    2 nobody nogroup 4096 Apr 15  2020 sys
drwxrwxrwt    2 nobody nogroup 4096 Jun  9 07:31 tmp
drwxr-xr-x   13 nobody nogroup 4096 Jun  9 07:27 usr
drwxr-xr-x   11 nobody nogroup 4096 Jun  9 07:31 var
```

There is a `mysupersecretflagfile` that looks interesting. Running `hash 0 1` will give us more information about the hash that's used
```
>>hash 0 1

----- HASHING DATA -----
Algorithm used: SHA-1
Digest size: 20
Block size: 64
Size hashed: 1
Offset: 0

Truncated Digest:
5BA93C9DB0CFF93F52B521D7420E43F6EDA278
```

Since this is a one byte SHA-1 hash, we can test a few possibilities using a tool like Cyber Chef. We see that SHA-1(0) == `5ba93c9db0cff93f52b521d7420e43f6eda2784f` which matches the truncated digest. We can see that the last byte is cut off of the hash before it is given to us. 
## Solve 

Let's try loading the flag file, run `load_file mysupersecretflagfile` and lets see if the hash at address 0 changes.

```
>>load_file mysupersecretflagfile

Loading file data into memory file: mysupersecretflagfile
File succesfully loaded into memory
>>hash 0 1

----- HASHING DATA -----
Algorithm used: SHA-1
Digest size: 20
Block size: 64
Size hashed: 1
Offset: 0

Truncated Digest:
06576556D1AD802F247CAD11AE748BE47B70CD
```
Looks like something changed at 0, if we make the assumption that the flag was loaded there, we can quickly check the SHA-1 of 'R' and see that it matches. Now we can write a script to do this same process for each character of the flag. Since we don't know the flag length we will assume 50 bytes. 

```python
from pwn import *
from hashlib import sha1

def brute(target):
  for i in range(256):
    if (int(sha1(bytes([i])).hexdigest(),16)>>8) == int(target,16):
      return chr(i)
  return chr(0)

con = remote('172.18.10.105', 38483)
con.recvuntil(">>")
con.sendline("load_file mysupersecretflagfile")
flag = ""
for i in range(50):
  con.recvuntil(">>")
  con.sendline(f"hash {i} 1")
  con.recvuntil("Truncated")
  con.recvline()
  target=con.recvline()
  print(F"{i}: {target}")
  flag += brute(target)
  print(flag)
  ```

```
0: b'06576556D1AD802F247CAD11AE748BE47B70CD\n'
R
1: b'C2C53D66948214258A26CA9CA845D7AC0C17F8\n'
RT
2: b'C032ADC1FF629C9B66F22749AD667E6BEADF14\n'
RTX
3: b'E69F20E9F683920D3FB4329ABD951E878B1F93\n'
RTXF
4: b'D160E0986ACA4714714A16F29EC605AF90BE70\n'
RTXFL
5: b'6DCD4CE23D88E2EE9568BA546C007C63D9131C\n'
RTXFLA
6: b'A36A6718F54524D846894FB04B5B885B4E43E6\n'
RTXFLAG
7: b'60BA4B2DAA4ED4D070FEC06687E249E0E6F9EE\n'
RTXFLAG{
8: b'3C363836CF4E16666669A25DA280A1865C2D28\n'
RTXFLAG{d
9: b'1B6453892473A467D07372D45EB05ABC203164\n'
RTXFLAG{d4
10: b'8EFD86FB78A56A5145ED7739DCB00C78581C53\n'
RTXFLAG{d4t
11: b'86F7E437FAA5A7FCE15D1DDCB9EAEAEA377667\n'
RTXFLAG{d4ta
12: b'53A0ACFAD59379B3E050338BF9F23CFC172EE7\n'
RTXFLAG{d4ta_
13: b'58E6B3A414A1E090DFC6029ADD0F3555CCBA12\n'
RTXFLAG{d4ta_e
14: b'11F6AD8EC52A2984ABAAFD7C3B516503785C20\n'
RTXFLAG{d4ta_ex
15: b'4A0A19218E082A343A1B17E5333409AF9D98F0\n'
RTXFLAG{d4ta_exf
16: b'356A192B7913B04C54574D18C28D46E6395428\n'
RTXFLAG{d4ta_exf1
17: b'07C342BE6E560E7F43842E2E21B774E61D85F0\n'
RTXFLAG{d4ta_exf1l
18: b'C2B7DF6201FDD3362399091F0A29550DF3505B\n'
RTXFLAG{d4ta_exf1l}
```
