# Rosenburg - Crypto 500

>They should already know intended solutions mean nothing to us - sine_nomine

## Challenge Description
In that file you found from the hash system we saw something referring to "Rosenberg" with another network service to connect to, this one is a bit strange, we know the communicatons are encrypted but we can't figure out how to break them maybe you'll have better luck with this one? 

NOTE: Rosenberg is the creator of the challenge, it's not a hint towards anything. 

Connect here: 

nc 172.18.10.105 30303

## Initial Observations
Connecting to the server, we get a ciphertext and are asked to send one back. Echoing back the ciphertext (without the 0x) we get `correct padding`. Removing a few bytes and sending it back gives us `wrong padding`. This server is vulnerable to a Padding Oracle attack.

## Padding Oracle

The Padding Oracle attack is an attack against block-mode crypto systems (Like AES-CBC). In order to work, block ciphers need the input to be divisible by a certain number (Usually the length of the key). The way they accomplish this is with an algorithm called PKCS7. 

### Padding
In a nutshell, you take the number of bytes you need to pad, lets call this number N, and you add N to the end of your message until it's the correct size. If you need no padding, N becomes the key size and you add N N's to the end. So if your message is "Hello" and needs to be padded to 8 bytes, it would be "Hello\x03\x03\x03". If your message was "Padding!" then you would add 8 0x08 and get "Padding!\x08\x08\x08\x08\x08\x08\x08\x08". This way the other end always knows how much padding to strip from the message.

### Oracle
This required padding causes some information to be leaked if the error messages tell you when there was a padding error. A padding error means that after decryption, the message looked like this "Message!\x03\x03" The number of padding bytes doesn't match the value of the padding bytes. This means if we have an oracle that tells us that the padding was wrong, we can leak information about the actual message. 

### Attack
How can we use this to leak information? Well think back to how block ciphers decrypt data.
![CBC Mode Graph](images/Cbc_decryption.png)
The plaintext of the last block is directly controlled by the ciphertext of the previous block. If you're more mathematically inclined you can think of it like this.
```c
Cn //Last block of ciphertext
Cp //Previous block of ciphertext
In = decrypt(Cn, key) //The intermediate is the result of decrypting the last block with the unknown key
Pn = Cp ^ In //The last block of plaintext is the previous ciphertext XOR with the intermediate.
```
We control Cp, and we know we will get an error whenever Pn is badly padded. We can change the last byte of Cp one byte at a time and find one that gives us good padding. Once we know that the padding is good, we will know that the padding must be 0x01 since we only changed one byte. 
```c
P`n = 0x1//The last byte of the modified plaintext
P`n = C`p ^ In //In hasn't changed from before since the last block is the same
In = C`p ^ P`n = C`p ^ 0x01 //We can get the real value of In by XORing the controlled C`p and the current padding 0x01
Pn = Cp ^ In //Now that we know In we can substitute the original Cp to get the original Pn
```
Then we just need to repeat this for every byte of the block, increasing the padding from \x01 to \x02\x02 to \x03\x03\x03 and so on. When we solved every byte of the block, we can just remove the last block and repeat the procedure with the new last block. 


## Exploit

First steps are just grabbing the ciphertext from the server and splitting them into a list of bytes. I kept them as str to make it easier to join them and send them in the loop. I also guessed a seemingly good block size of 16 bytes. The loop is done in 3 stages, The k loop will take two blocks and separate them into an oracle and padding. Then the j loops for each byte in the block, testing every possible byte value in the i loop. For each test we will send the oracleBlock+paddingBlock. We will initially skip bytes that are the same since that can lead to false positives. If we don't find another candidate then we will use the skipped byte. 

```python
from pwn import *
import sys

good = b"\x00\x00correct padding"
bad =  b"\x00\x00wrong padding"
blockSize = 16
con = remote('172.18.10.105', 30303)
target = con.recvline().split(b'x')[-1][:-1]
print(F"Got target: 0x{target}")
originalList = [target[i:i+2].decode('ascii') for i in range(0,len(target),2)]
print(F"Got originalList: {originalList}")
out = b''

for k in range(len(originalList)//blockSize-1,0,-1):
  intermediates = [0]*blockSize
  originalOracle = originalList[16*(k-1):16*k]
  oracleBlock = originalList[16*(k-1):16*k]
  blockOut = b''
  paddingBlock = originalList[16*k:16*(k+1)]
  
  for j in range(1,blockSize+1):
    for i in range(256):
      if i == int(originalOracle[-j],16):
        print(F"Skipping check of same value {hex(i)}")
        continue
      con.recvuntil("ciphertext")
      oracleBlock[-j] = "%02x"%i
      con.sendline(''.join([''.join(oracleBlock),''.join(paddingBlock)]))
      status = con.recvline().strip()
      if status == good:
        intermediates[-j] = i ^ j
        blockOut = bytes([ intermediates[-j] ^ int(originalOracle[-j],16)]) + blockOut
        for k in range(j):
          oracleBlock[-(1+k)] = "%02x"%(int(oracleBlock[-(1+k)],16) ^ j ^ (j+1))
        print(blockOut + out)
        break
    if len(blockOut) != j:
      oracleBlock[-j] = originalOracle[-j]
      intermediates[-j] = int(originalOracle[-j],16) ^ j
      blockOut = bytes([ intermediates[-j] ^ int(originalOracle[-j],16)]) + blockOut
      for h in range(j):
        oracleBlock[-(1+h)] = "%02x"%(int(oracleBlock[-(1+h)],16) ^ j ^ (j+1))
      print(blockOut + out)
      
  out = blockOut + out
print(out)
```

This will not give us the entire flag however since the first block has no previous block. Normally we would use the IV as the previous block, but the IV is not available to us to modify. We only get 48 bytes which is 3 blocks, and the bytes we were able to recover fit in the last two of the blocks, `4dd1ng_4tt4ck_r0` and `s3n_b3rg}\x07\x07\x07\x07\x07\x07\x07`, meaning the IV is not being sent for us to manipulate. Fortunately the flag prefix is known to be `RTXFLAG{` 8 bytes, The last 8 bytes are also pretty easy to guess since we have the end. 

`RTXFLAG{0r4cl3_p4dd1ng_4tt4ck_r0s3n_b3rg}`
