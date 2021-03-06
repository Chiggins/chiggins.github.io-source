---
title: "Derbycon 4.0 Ctf   Trndocs Elf Binary Reverse Engineering and Debugging"
date: 2014-09-30T17:40:44-05:00
draft: false
---

So this past weekend I attended DerbyCon 4.0 in Louisville, Kentucky, and was lucky enough to play the CTF along side the [@bsjtf](https://twitter.com/bsjtf) team. We were able to place 16th out of the 77 point scoring teams/individuals, which is pretty damn good I’d say. This write-up will be for a reversing challenge I solved, adding 450 points to the teams total.

One of the first things done was a scan for various services on the network. Since FTP is a common place for CTF flags to hide, went searching for any FTP servers in the environment.

    nmap 10.10.146.1/24 -p 21 --open -sV

One of the FTP servers that stood out was 10.10.146.74 which has anonymous read access and quite an interesting MOTD when logging in.

```
230-This is a temporary ftp server, while we finish our migration off DOS
230-platforms. For now transaction documents are still availible at
230-/DRIVE_C/TRNDOCS but these are already being generated by the LINUX
230-backend.
230-
230-DOCS for DCs that have already been upgraded are ciphered with
230-OpenSSL, the utility to obtain the shared password from your
230-credentials is TRNDOCS directory.
230-Use the serer xxx.xxx.xxx.xxx to authenticate, you can manually
230-inspect a TRN document with OpenSSL once you obtain the key.
230-
230-openssl des3 -d -salt -in -k
230-
230 User logged in
```

Interesting, /DRIVE_C/TRNDOCS/ seems to have a lot of interesting information in it. So I pull everything down and start analyzing the file in there.

```
[chiggins@vader ~/DerbyCon4.0/74/DRIVE_C/TRNDOCS]$ ls
ATL  CLE  HBG  PGET_A.OUT  RIC
[chiggins@vader ~/DerbyCon4.0/74/DRIVE_C/TRNDOCS]$ file PGET_A.OUT
PGET_A.OUT: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), statically linked, for GNU/Linux 3.2.29, not stripped
[chiggins@vader ~/DerbyCon4.0/74/DRIVE_C/TRNDOCS]$ cd ATL
[chiggins@vader ~/DerbyCon4.0/74/DRIVE_C/TRNDOCS/ATL]$ file *
10.TRN: data
11.TRN: data
12.TRN: data
13.TRN: data
14.TRN: data
15.TRN: data
1.TRN:  data
2.TRN:  data
3.TRN:  data
4.TRN:  data
5.TRN:  data
6.TRN:  data
7.TRN:  data
8.TRN:  data
9.TRN:  data
[chiggins@vader ~/DerbyCon4.0/74/DRIVE_C/TRNDOCS/ATL]$ cd ..
[chiggins@vader ~/DerbyCon4.0/74/DRIVE_C/TRNDOCS]$ cd CLE
[chiggins@vader ~/DerbyCon4.0/74/DRIVE_C/TRNDOCS/CLE]$ file *
10.TRN: ASCII text
11.TRN: ASCII text
12.TRN: ASCII text
13.TRN: ASCII text
14.TRN: ASCII text
15.TRN: ASCII text
1.TRN:  ASCII text
2.TRN:  ASCII text
3.TRN:  ASCII text
4.TRN:  ASCII text
5.TRN:  ASCII text
6.TRN:  ASCII text
7.TRN:  ASCII text
8.TRN:  ASCII text
9.TRN:  ASCII text
[chiggins@vader ~/DerbyCon4.0/74/DRIVE_C/TRNDOCS/CLE]$ cat 1.TRN
invoice: 21815
order: 8925
customer: 30871
scrip: Percocet
days: 14
addr1: James Monroe
addr2: 6596 Euclid
addr3; Zanesville PA 22112
```

So it looks like we have a combination of plaintext data, encrypted data, and an ELF binary, which is very interesting. Running strings on the binary doesn’t show anything of super interest. Lets just run it and see what happens.

```
[chiggins@vader ~/DerbyCon4.0/74/DRIVE_C/TRNDOCS]$ ./PGET_A.OUT 
Must set options -s (server) -p (password) and -u (username).
[chiggins@vader ~/DerbyCon4.0/74/DRIVE_C/TRNDOCS]$ ./PGET_A.OUT -s 127.0.0.1 -p password -u user
```

The program accepts a server (which I assume it wants to talk to), and a username and password to validate against. Running it with my localhost set as the server just lets it hang, so I’m assuming a socket is being created somewhere. Using strace we’ll be able to see more about this.

```
[chiggins@vader ~/DerbyCon4.0/74/DRIVE_C/TRNDOCS]$ strace ./PGET_A.OUT -s 127.0.0.1 -p password -u user
execve("./PGET_A.OUT", ["./PGET_A.OUT", "-s", "127.0.0.1", "-p", "password", "-u", "user"], [/* 44 vars */]) = 0
[ Process PID=10312 runs in 32 bit mode. ]
uname({sys="Linux", node="vader", ...}) = 0
brk(0)                                  = 0x9f55000
brk(0x9f55d40)                          = 0x9f55d40
set_thread_area({entry_number:-1, base_addr:0x9f55840, limit:1048575, seg_32bit:1, contents:0, read_exec_only:0, limit_in_pages:1, seg_not_present:0, useable:1}) = 0 (entry_number:12)
brk(0x9f76d40)                          = 0x9f76d40
brk(0x9f77000)                          = 0x9f77000
socket(PF_INET, SOCK_DGRAM, IPPROTO_UDP) = 4
sendto(4, "user password", 13, 0, {sa_family=AF_INET, sin_port=htons(3000), sin_addr=inet_addr("127.0.0.1")}, 16) = 13
recvfrom(4,
)
```

I was right, a socket is trying to make a connection and read back some data. If you notice in sendto, the socket is trying to send data “user password” to 127.0.0.1 on port 3000. I figured that this program was meant to talk to a server on the network, but scanning for an open port 3000 turned out no results. Figured to just keep moving on with this binary and see what else it does.

So I decided to set up a natcat listener on port 3000 and try to receive the data and send data back. One thing that tripped me up with this is that I didn’t realize you needed to tell netcat to listen as a UDP port. Before knowing that, whenever I had my netcat listener running on TCP 3000, the program would not connect to it. After finding out how to specify UDP, I was finally able to send data back and forth.

```
#### netcat listener ####
[chiggins@vader ~]$ nc -ulvvp 3000
Listening on any address 3000 (hbci)
Received packet from 127.0.0.1:58075 -> 127.0.0.1:3000 (local)
user passwordJUNKDATA

#### program ####
[chiggins@vader ~/DerbyCon4.0/74/DRIVE_C/TRNDOCS]$ ./PGET_A.OUT -s 127.0.0.1 -p password -u user
bad username or password

#### program with strace ####
[chiggins@vader ~/DerbyCon4.0/74/DRIVE_C/TRNDOCS]$ strace ./PGET_A.OUT -s 127.0.0.1 -p password -u user
execve("./PGET_A.OUT", ["./PGET_A.OUT", "-s", "127.0.0.1", "-p", "password", "-u", "user"], [/* 44 vars */]) = 0
[ Process PID=10537 runs in 32 bit mode. ]
uname({sys="Linux", node="vader", ...}) = 0
brk(0)                                  = 0x9388000
brk(0x9388d40)                          = 0x9388d40
set_thread_area({entry_number:-1, base_addr:0x9388840, limit:1048575, seg_32bit:1, contents:0, read_exec_only:0, limit_in_pages:1, seg_not_present:0, useable:1}) = 0 (entry_number:12)
brk(0x93a9d40)                          = 0x93a9d40
brk(0x93aa000)                          = 0x93aa000
socket(PF_INET, SOCK_DGRAM, IPPROTO_UDP) = 4
sendto(4, "user password", 13, 0, {sa_family=AF_INET, sin_port=htons(3000), sin_addr=inet_addr("127.0.0.1")}, 16) = 13
recvfrom(4, "JUNKDA", 6, 0, {sa_family=AF_INET, sin_port=htons(3000), sin_addr=inet_addr("127.0.0.1")}, [16]) = 6
fstat64(1, {st_mode=S_IFCHR|0620, st_rdev=makedev(136, 0), ...}) = 0
mmap2(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0xfffffffff7722000
write(1, "bad username or password\n", 25bad username or password
) = 25
exit_group(0)                           = ?
+++ exited with 0 +++
```

So there’s a bit going on here. The netcat listener gets set up to listen on port 3000 with the -u specifing UDP. Then when the program runs, it sends along the specified arguments “user password”. I send back “JUNKDATA” and the program fails saying bad username and password. I also included the strace output, and you’ll notice that the program only reads 6 bytes, “JUNKDA”.

Here is where we start to do some debugging and reversing. I load up the program in gdb, set the required arguments, and take a look at the main method.

```
[chiggins@vader ~/DerbyCon4.0/74/DRIVE_C/TRNDOCS]$ gdb -q ./PGET_A.OUT
Reading symbols from ./PGET_A.OUT...done.
gdb>set args -s 127.0.0.1 -u username -p password
gdb>disassemble main
Dump of assembler code for function main:
   0x08048c85 <+0>:	push   ebp
   0x08048c86 <+1>:	mov    ebp,esp
   0x08048c88 <+3>:	push   edi
   0x08048c89 <+4>:	push   esi
   0x08048c8a <+5>:	push   ebx
   0x08048c8b <+6>:	and    esp,0xfffffff0
   0x08048c8e <+9>:	sub    esp,0x50
   0x08048c91 <+12>:	mov    DWORD PTR [esp+0x4c],0x0
   0x08048c99 <+20>:	mov    DWORD PTR [esp+0x48],0x0
   0x08048ca1 <+28>:	mov    DWORD PTR [esp+0x44],0x0
   0x08048ca9 <+36>:	lea    eax,[esp+0x13]
   0x08048cad <+40>:	mov    edx,0x80bac20
   0x08048cb2 <+45>:	mov    ebx,0x25
   0x08048cb7 <+50>:	mov    ecx,eax
   0x08048cb9 <+52>:	and    ecx,0x1
   0x08048cbc <+55>:	test   ecx,ecx
   0x08048cbe <+57>:	je     0x8048cc7 <main+66>
   0x08048cc0 <+59>:	mov    cl,BYTE PTR [edx]
   0x08048cc2 <+61>:	mov    BYTE PTR [eax],cl
   0x08048cc4 <+63>:	inc    eax
   0x08048cc5 <+64>:	inc    edx
   0x08048cc6 <+65>:	dec    ebx
   0x08048cc7 <+66>:	mov    ecx,eax
   0x08048cc9 <+68>:	and    ecx,0x2
   0x08048ccc <+71>:	test   ecx,ecx
   0x08048cce <+73>:	je     0x8048cdf <main+90>
   0x08048cd0 <+75>:	mov    cx,WORD PTR [edx]
   0x08048cd3 <+78>:	mov    WORD PTR [eax],cx
   0x08048cd6 <+81>:	add    eax,0x2
   0x08048cd9 <+84>:	add    edx,0x2
   0x08048cdc <+87>:	sub    ebx,0x2
   0x08048cdf <+90>:	mov    ecx,ebx
   0x08048ce1 <+92>:	shr    ecx,0x2
   0x08048ce4 <+95>:	mov    edi,eax
   0x08048ce6 <+97>:	mov    esi,edx
   0x08048ce8 <+99>:	rep movs DWORD PTR es:[edi],DWORD PTR ds:[esi]
   0x08048cea <+101>:	mov    edx,esi
   0x08048cec <+103>:	mov    eax,edi
   0x08048cee <+105>:	mov    ecx,0x0
   0x08048cf3 <+110>:	mov    esi,ebx
   0x08048cf5 <+112>:	and    esi,0x2
   0x08048cf8 <+115>:	test   esi,esi
   0x08048cfa <+117>:	je     0x8048d07 <main+130>
   0x08048cfc <+119>:	mov    si,WORD PTR [edx+ecx*1]
   0x08048d00 <+123>:	mov    WORD PTR [eax+ecx*1],si
   0x08048d04 <+127>:	add    ecx,0x2
   0x08048d07 <+130>:	and    ebx,0x1
   0x08048d0a <+133>:	test   ebx,ebx
   0x08048d0c <+135>:	je     0x8048d14 <main+143>
   0x08048d0e <+137>:	mov    dl,BYTE PTR [edx+ecx*1]
   0x08048d11 <+140>:	mov    BYTE PTR [eax+ecx*1],dl
   0x08048d14 <+143>:	mov    DWORD PTR ds:0x80e4a18,0x0
   0x08048d1e <+153>:	jmp    0x8048d75 <main+240>
   0x08048d20 <+155>:	cmp    DWORD PTR [esp+0x3c],0x75
   0x08048d25 <+160>:	jne    0x8048d32 <main+173>
   0x08048d27 <+162>:	mov    eax,ds:0x80e6218
   0x08048d2c <+167>:	mov    DWORD PTR [esp+0x4c],eax
   0x08048d30 <+171>:	jmp    0x8048d75 <main+240>
   0x08048d32 <+173>:	cmp    DWORD PTR [esp+0x3c],0x70
   0x08048d37 <+178>:	jne    0x8048d44 <main+191>
   0x08048d39 <+180>:	mov    eax,ds:0x80e6218
   0x08048d3e <+185>:	mov    DWORD PTR [esp+0x48],eax
   0x08048d42 <+189>:	jmp    0x8048d75 <main+240>
   0x08048d44 <+191>:	cmp    DWORD PTR [esp+0x3c],0x73
   0x08048d49 <+196>:	jne    0x8048d56 <main+209>
   0x08048d4b <+198>:	mov    eax,ds:0x80e6218
   0x08048d50 <+203>:	mov    DWORD PTR [esp+0x44],eax
   0x08048d54 <+207>:	jmp    0x8048d75 <main+240>
   0x08048d56 <+209>:	mov    edx,DWORD PTR ds:0x80e4a14
   0x08048d5c <+215>:	mov    eax,ds:0x80e453c
   0x08048d61 <+220>:	mov    DWORD PTR [esp+0x8],edx
   0x08048d65 <+224>:	mov    DWORD PTR [esp+0x4],0x80bab4e
   0x08048d6d <+232>:	mov    DWORD PTR [esp],eax
   0x08048d70 <+235>:	call   0x80498a0 <fprintf>
   0x08048d75 <+240>:	mov    DWORD PTR [esp+0x8],0x80bab62
   0x08048d7d <+248>:	mov    eax,DWORD PTR [ebp+0xc]
   0x08048d80 <+251>:	mov    DWORD PTR [esp+0x4],eax
   0x08048d84 <+255>:	mov    eax,DWORD PTR [ebp+0x8]
   0x08048d87 <+258>:	mov    DWORD PTR [esp],eax
   0x08048d8a <+261>:	call   0x8055b10 <getopt>
   0x08048d8f <+266>:	mov    DWORD PTR [esp+0x3c],eax
   0x08048d93 <+270>:	cmp    DWORD PTR [esp+0x3c],0xffffffff
   0x08048d98 <+275>:	jne    0x8048d20 <main+155>
   0x08048d9a <+277>:	cmp    DWORD PTR [esp+0x4c],0x0
   0x08048d9f <+282>:	je     0x8048daf <main+298>
   0x08048da1 <+284>:	cmp    DWORD PTR [esp+0x48],0x0
   0x08048da6 <+289>:	je     0x8048daf <main+298>
   0x08048da8 <+291>:	cmp    DWORD PTR [esp+0x44],0x0
   0x08048dad <+296>:	jne    0x8048dde <main+345>
   0x08048daf <+298>:	mov    eax,ds:0x80e453c
   0x08048db4 <+303>:	mov    DWORD PTR [esp+0xc],eax
   0x08048db8 <+307>:	mov    DWORD PTR [esp+0x8],0x3e
   0x08048dc0 <+315>:	mov    DWORD PTR [esp+0x4],0x1
   0x08048dc8 <+323>:	mov    DWORD PTR [esp],0x80bab6c
   0x08048dcf <+330>:	call   0x8049900 <fwrite>
   0x08048dd4 <+335>:	mov    eax,0x1
   0x08048dd9 <+340>:	jmp    0x8048eed <main+616>
   0x08048dde <+345>:	mov    DWORD PTR [esp+0x8],0x11
   0x08048de6 <+353>:	mov    DWORD PTR [esp+0x4],0x2
   0x08048dee <+361>:	mov    DWORD PTR [esp],0x2
   0x08048df5 <+368>:	call   0x8057100 <socket>
   0x08048dfa <+373>:	mov    DWORD PTR [esp+0x38],eax
   0x08048dfe <+377>:	cmp    DWORD PTR [esp+0x38],0x0
   0x08048e03 <+382>:	jns    0x8048e34 <main+431>
   0x08048e05 <+384>:	mov    eax,ds:0x80e453c
   0x08048e0a <+389>:	mov    DWORD PTR [esp+0xc],eax
   0x08048e0e <+393>:	mov    DWORD PTR [esp+0x8],0x16
   0x08048e16 <+401>:	mov    DWORD PTR [esp+0x4],0x1
   0x08048e1e <+409>:	mov    DWORD PTR [esp],0x80babab
   0x08048e25 <+416>:	call   0x8049900 <fwrite>
   0x08048e2a <+421>:	mov    eax,0x1
   0x08048e2f <+426>:	jmp    0x8048eed <main+616>
   0x08048e34 <+431>:	mov    eax,DWORD PTR [esp+0x38]
   0x08048e38 <+435>:	mov    DWORD PTR [esp+0xc],eax
   0x08048e3c <+439>:	mov    eax,DWORD PTR [esp+0x44]
   0x08048e40 <+443>:	mov    DWORD PTR [esp+0x8],eax
   0x08048e44 <+447>:	mov    eax,DWORD PTR [esp+0x48]
   0x08048e48 <+451>:	mov    DWORD PTR [esp+0x4],eax
   0x08048e4c <+455>:	mov    eax,DWORD PTR [esp+0x4c]
   0x08048e50 <+459>:	mov    DWORD PTR [esp],eax
   0x08048e53 <+462>:	call   0x8048aec <sendAuth>
   0x08048e58 <+467>:	test   eax,eax
   0x08048e5a <+469>:	je     0x8048ec3 <main+574>
   0x08048e5c <+471>:	mov    eax,DWORD PTR [esp+0x38]
   0x08048e60 <+475>:	mov    DWORD PTR [esp],eax
   0x08048e63 <+478>:	call   0x8048bde <recvAuth>
   0x08048e68 <+483>:	test   eax,eax
   0x08048e6a <+485>:	je     0x8048eb5 <main+560>
   0x08048e6c <+487>:	mov    DWORD PTR [esp],0x80babc2
   0x08048e73 <+494>:	call   0x8049a50 <puts>
   0x08048e78 <+499>:	mov    DWORD PTR [esp+0x40],0x0
   0x08048e80 <+507>:	jmp    0x8048ea0 <main+539>
   0x08048e82 <+509>:	lea    edx,[esp+0x13]
   0x08048e86 <+513>:	mov    eax,DWORD PTR [esp+0x40]
   0x08048e8a <+517>:	add    eax,edx
   0x08048e8c <+519>:	mov    al,BYTE PTR [eax]
   0x08048e8e <+521>:	xor    eax,0x17
   0x08048e91 <+524>:	movsx  eax,al
   0x08048e94 <+527>:	mov    DWORD PTR [esp],eax
   0x08048e97 <+530>:	call   0x8049c80 <putchar>
   0x08048e9c <+535>:	inc    DWORD PTR [esp+0x40]
   0x08048ea0 <+539>:	cmp    DWORD PTR [esp+0x40],0x24
   0x08048ea5 <+544>:	jle    0x8048e82 <main+509>
   0x08048ea7 <+546>:	mov    DWORD PTR [esp],0xa
   0x08048eae <+553>:	call   0x8049c80 <putchar>
   0x08048eb3 <+558>:	jmp    0x8048ee8 <main+611>
   0x08048eb5 <+560>:	mov    DWORD PTR [esp],0x80babca
   0x08048ebc <+567>:	call   0x8049a50 <puts>
   0x08048ec1 <+572>:	jmp    0x8048ee8 <main+611>
   0x08048ec3 <+574>:	mov    eax,ds:0x80e453c
   0x08048ec8 <+579>:	mov    DWORD PTR [esp+0xc],eax
   0x08048ecc <+583>:	mov    DWORD PTR [esp+0x8],0x1e
   0x08048ed4 <+591>:	mov    DWORD PTR [esp+0x4],0x1
   0x08048edc <+599>:	mov    DWORD PTR [esp],0x80babe4
   0x08048ee3 <+606>:	call   0x8049900 <fwrite>
   0x08048ee8 <+611>:	mov    eax,0x0
   0x08048eed <+616>:	lea    esp,[ebp-0xc]
   0x08048ef0 <+619>:	pop    ebx
   0x08048ef1 <+620>:	pop    esi
   0x08048ef2 <+621>:	pop    edi
   0x08048ef3 <+622>:	pop    ebp
   0x08048ef4 <+623>:	ret
End of assembler dump.
```

The disassembly for main is pretty crazy, but if you just walk through it, it’s not too bad. It does some argument checking, creates a socket (0x08048df5 <+368>), sends the authentication data (0x08048e53 <+462>), receives the authentication response (0x08048e63 <+478>), and some more nonsense towards the end.

Something interesting that I noticed is that right after we receive the authentication data, a test is done on eax against eax (0x08048e68 <+483>). In assembly, the return value of a function is stored in eax, so this test is looking to verify the return of the recvAuth method. I think it is safe to assume that here is where the program will decide to give us either the “bad username or password” text, or continue on to do something more.

Before I continued, I had to look up what test actually did. From [this question on StackOverflow](https://stackoverflow.com/questions/13064809/the-point-of-test-eax-eax), “TEST sets the Zero Flag if the the result of the AND operation is zero. If two operands are equal, their bitwise AND is zero iff both are zero. It also sets the Sign Flag if the top bit is set in the result, and the Parity Flag if the number of set bits is even.” What this means is that in our program, if eax is set to 0x0, then the program will jump to 0x8048eb5 , which is towards the end of the program. If we put a breakpoint at this location and look at the registers and disassembly, we can see this happen.

```
gdb>break *0x08048e68
Breakpoint 1 at 0x8048e68
gdb>run
Starting program: /home/chiggins/DerbyCon4.0/74/DRIVE_C/TRNDOCS/PGET_A.OUT -s 127.0.0.1 -u username -p password
Got object file from memory but can't read symbols: File truncated.

Breakpoint 1, 0x08048e68 in main ()
gdb>info registers
eax            0x0	0x0
ecx            0xffffd5e8	0xffffd5e8
edx            0xffffd54a	0xffffd54a
ebx            0x0	0x0
esp            0xffffd610	0xffffd610
ebp            0xffffd678	0xffffd678
esi            0x0	0x0
edi            0xffffd648	0xffffd648
eip            0x8048e68	0x8048e68 <main+483>
eflags         0x287	[ CF PF SF IF ]
cs             0x23	0x23
ss             0x2b	0x2b
ds             0x2b	0x2b
es             0x2b	0x2b
fs             0x0	0x0
gs             0x63	0x63
gdb>disassemble main
Dump of assembler code for function main:
[... snip ...]
   0x08048e53 <+462>:	call   0x8048aec <sendAuth>
   0x08048e58 <+467>:	test   eax,eax
   0x08048e5a <+469>:	je     0x8048ec3 <main+574>
   0x08048e5c <+471>:	mov    eax,DWORD PTR [esp+0x38]
   0x08048e60 <+475>:	mov    DWORD PTR [esp],eax
   0x08048e63 <+478>:	call   0x8048bde <recvAuth>
=> 0x08048e68 <+483>:	test   eax,eax
   0x08048e6a <+485>:	je     0x8048eb5 <main+560>
   0x08048e6c <+487>:	mov    DWORD PTR [esp],0x80babc2
   0x08048e73 <+494>:	call   0x8049a50 <puts>
   0x08048e78 <+499>:	mov    DWORD PTR [esp+0x40],0x0
   0x08048e80 <+507>:	jmp    0x8048ea0 <main+539>
   0x08048e82 <+509>:	lea    edx,[esp+0x13]
[... snip ...]
End of assembler dump.
gdb>continue
Continuing.
bad username or password
[Inferior 1 (process 11299) exited normally]
```

So after recvAuth eax was set to 0x0 which caused the result from test to jump to the end of the program and show the bad password message. What happens if we set eax to 0x1, maybe it will bypass the jump and continue onto something more interesting.

```
gdb>run
Starting program: /home/chiggins/DerbyCon4.0/74/DRIVE_C/TRNDOCS/PGET_A.OUT -s 127.0.0.1 -u username -p password
Got object file from memory but can't read symbols: File truncated.

Breakpoint 1, 0x08048e68 in main ()
gdb>disassemble main
Dump of assembler code for function main:
[... snip ...]
   0x08048e53 <+462>:   call   0x8048aec <sendAuth>
   0x08048e58 <+467>:   test   eax,eax
   0x08048e5a <+469>:   je     0x8048ec3 <main+574>
   0x08048e5c <+471>:   mov    eax,DWORD PTR [esp+0x38]
   0x08048e60 <+475>:   mov    DWORD PTR [esp],eax
   0x08048e63 <+478>:   call   0x8048bde <recvAuth>
=> 0x08048e68 <+483>:   test   eax,eax
   0x08048e6a <+485>:   je     0x8048eb5 <main+560>
   0x08048e6c <+487>:   mov    DWORD PTR [esp],0x80babc2
   0x08048e73 <+494>:   call   0x8049a50 <puts>
   0x08048e78 <+499>:   mov    DWORD PTR [esp+0x40],0x0
   0x08048e80 <+507>:   jmp    0x8048ea0 <main+539>
   0x08048e82 <+509>:   lea    edx,[esp+0x13]
[... snip ...]
End of assembler dump.
gdb>info registers 
eax            0x0	0x0
ecx            0xffffd5e8	0xffffd5e8
edx            0xffffd54a	0xffffd54a
ebx            0x0	0x0
esp            0xffffd610	0xffffd610
ebp            0xffffd678	0xffffd678
esi            0x0	0x0
edi            0xffffd648	0xffffd648
eip            0x8048e68	0x8048e68 <main+483>
eflags         0x287	[ CF PF SF IF ]
cs             0x23	0x23
ss             0x2b	0x2b
ds             0x2b	0x2b
es             0x2b	0x2b
fs             0x0	0x0
gs             0x63	0x63
gdb>set $eax = 0x1
gdb>info registers
eax            0x1	0x1
ecx            0xffffd5e8	0xffffd5e8
edx            0xffffd54a	0xffffd54a
ebx            0x0	0x0
esp            0xffffd610	0xffffd610
ebp            0xffffd678	0xffffd678
esi            0x0	0x0
edi            0xffffd648	0xffffd648
eip            0x8048e68	0x8048e68 <main+483>
eflags         0x287	[ CF PF SF IF ]
cs             0x23	0x23
ss             0x2b	0x2b
ds             0x2b	0x2b
es             0x2b	0x2b
fs             0x0	0x0
gs             0x63	0x63
gdb>continue
Continuing.
Success
The secret key is EverybodyDoTheLimit
[Inferior 1 (process 11416) exited normally]
```

Woah! Would you look at that, got something juicy. I tried to submit “EverybodyDoTheLimit” as a flag, but it ended up not being valid. But if you recall back to the FTP MOTD at the beginning of the post, we can now expect the TRN documents via openssl with the newly acquired secret key. After running through the various encrypted files, I finally ran into one with a flag in it.

```
[chiggins@vader ~/DerbyCon4.0/74/DRIVE_C/TRNDOCS/ATL]$ openssl des3 -d -salt -in 2.TRN -k EverybodyDoTheLimit
invoice: 19755
order: 11389
customer: 24956
scrip: flag
days: 90
addr1: CountingSheep75
addr2: 8982 Elm
addr3; Henderson LA 13131
```

Submitted “CountingSheep75” to the scoring engine, and received 450 points. Woo!

Since I’ve really been getting into reverse engineering and debugging lately, this was a really fun challenge for me. Thanks to the DerbyCon CTF crew for creating this one!
