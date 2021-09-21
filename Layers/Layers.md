# Layers - Crypto 300

>I am very upset yet proud of all crypto challenges being solved. But mostly upset... maybe mostly proud. - Casseus

## Challenge Description
We found this file in a folder called "Layers" on an open FTP that was located within the network you cracked previously, we were able to connect to the FTP after you got the password out of that previous archive, we're not sure what the deal is but there was a readme.txt next to it that just said the passphrase is "onionshavelayers".

The name of the file is just "encrypted_file".

encrypted_file
```
U2FsdGVkX18mkpj/R4a6Pjx4Lp+PA+CR/KCJG7XJfD5Q+QKEbiD21CIA5Yzaazc3
Ygnh7e/QNEOT5B6Jn2wLT79VzH7oP5uxB+Aezb+4s7EfzJPkANhmcSwGQbvkSP1N
LiFJexZsjoI7xiGYEKA0FKdbYsHaaah2742QqLqJw86TUaWoG4MP2Vj9yrkhSL7Y
sHXxNAQeq6HuCFxvkDzusYDmQw5OC1uHuzry21RZvoU2JqUXEqSzXXF5qVshw1O1
FljPl18ppCt2nGdQs2ocNeOOlw+m7BFXvsUq28yctH7uhgbjCJf7k0a/BHP0Gn7y
0at/MEqx4W3tRc+bmzAcYTacQZ+gieSLnYpOYcyQWEPfgurivHiC3KBd9NaWL0eG
PZy19+7x6ZJ/ytTX04SfQjbxbd7bXV2S9Shc8pUOCB1M2nnI54PNwqlCCzY3xIg1
5+YGGhte4DXtnr9zmgnimdbAruvnmQjYTx7JSJURZ2lz87j9Sauqvu5mxmjb8yGr
0a0tSJmLC90bqHiwsyqDsQ==
```

## Initial Observations
encrypted_file is plaintext and ends with '=' which are tell tale signs of base64 encoding. We can decode the file back to binary with the following command.

`base64 -d encrypted_file > decode`

Running `file` on the output tells us that it's an `openssl enc'd data with salted password` 

## Decrypting

Openssl is a tool used for encryption and decryption. It supports over 100 ciphers, since we don't know which one is the correct one we can write a simple bash script to loop through the options. 

```bash
ciphers=( -aes-128-cbc -aes-128-cfb -aes-128-cfb1 -aes-128-cfb8 -aes-128-ctr -aes-128-ecb -aes-128-ofb -aes-192-cbc -aes-192-cfb -aes-192-cfb1 -aes-192-cfb8 -aes-192-ctr -aes-192-ecb -aes-192-ofb -aes-256-cbc -aes-256-cfb -aes-256-cfb1 -aes-256-cfb8 -aes-256-ctr -aes-256-ecb -aes-256-ofb -aes128 -aes128-wrap -aes192 -aes192-wrap -aes256 -aes256-wrap -aria-128-cbc -aria-128-cfb -aria-128-cfb1 -aria-128-cfb8 -aria-128-ctr -aria-128-ecb -aria-128-ofb -aria-192-cbc -aria-192-cfb -aria-192-cfb1 -aria-192-cfb8 -aria-192-ctr -aria-192-ecb -aria-192-ofb -aria-256-cbc -aria-256-cfb -aria-256-cfb1 -aria-256-cfb8 -aria-256-ctr -aria-256-ecb -aria-256-ofb -aria128 -aria192 -aria256 -bf -bf-cbc -bf-cfb -bf-ecb -bf-ofb -blowfish -camellia-128-cbc -camellia-128-cfb -camellia-128-cfb1 -camellia-128-cfb8 -camellia-128-ctr -camellia-128-ecb -camellia-128-ofb -camellia-192-cbc -camellia-192-cfb -camellia-192-cfb1 -camellia-192-cfb8 -camellia-192-ctr -camellia-192-ecb -camellia-192-ofb -camellia-256-cbc -camellia-256-cfb -camellia-256-cfb1 -camellia-256-cfb8 -camellia-256-ctr -camellia-256-ecb -camellia-256-ofb -camellia128 -camellia192 -camellia256 -cast -cast-cbc -cast5-cbc -cast5-cfb -cast5-ecb -cast5-ofb -chacha20 -des -des-cbc -des-cfb -des-cfb1 -des-cfb8 -des-ecb -des-ede -des-ede-cbc -des-ede-cfb -des-ede-ecb -des-ede-ofb -des-ede3 -des-ede3-cbc -des-ede3-cfb -des-ede3-cfb1 -des-ede3-cfb8 -des-ede3-ecb -des-ede3-ofb -des-ofb -des3 -des3-wrap -desx -desx-cbc -id-aes128-wrap -id-aes128-wrap-pad -id-aes192-wrap -id-aes192-wrap-pad -id-aes256-wrap -id-aes256-wrap-pad -id-smime-alg-CMS3DESwrap -rc2 -rc2-128 -rc2-40 -rc2-40-cbc -rc2-64 -rc2-64-cbc -rc2-cbc -rc2-cfb -rc2-ecb -rc2-ofb -rc4 -rc4-40 -seed -seed-cbc -seed-cfb -seed-ecb -seed-ofb -sm4 -sm4-cbc -sm4-cfb -sm4-ctr -sm4-ecb -sm4-ofb )

for c in "${ciphers[@]}"
do
	echo "Trying " $c
	openssl enc -d -in decode -k onionshavelayers $c
	read -n 1 -p "Is this correct? [y/N]" save
	if [ "$save" = "y" ]; then
		openssl enc -d -in decode -k onionshavelayers $c -out decrypt
		break
	fi
done
```

We quickly discover the correct cipher was aes-128-cbc, and the output is another base64 encoded file. 

```
U2FsdGVkX1/zJ47hlSkrIE5V06xXcj9SeGFz/jo1GsMr5oT902Ui0twM2lJOqEfe
2EPhdEupxHR3QTZJlGyrsSwmwCUQFZmVSP6n+/xaPy43WFGOh3ueb3VyKIsBPpKD
fz253Rwecc/6yer/s4WNzEXdcj/7pd4gDxdg7xp8c7iOyACroBPs7JiTUYbiD5QA
bHscyxiyPQfOKBktfrTul2IgmG8ZAoxLusfx7qgv2HYGUuGI2YFsCqgSpG9YnDfE
uyUmZYr2tuk89VONAAbxXcXHOb0M/NyIeZ1dt6stonjWaW3Qlq7m3QuTQJ0XAHnW
MyxkHDNYdTCrbNnT6zQMO+UvYHtTF862Iw8wUYV7guyzROQNfKfVww==
```

Following the same steps as before we find another openssl encrypted file. Run the cipher brute force script on the new file and when we get to bf we will get some output starting with "PK" and containing "flag.txt". PK is the file header for a zip file, so save as a .zip. 

The zip is password protected, so use zip2john to pull the hashes out of the zip file and crack them with John the Ripper. The password is `monkey` and the flag is `RTXFLAG{th4t_suck3d_l4y3rs}
`
