---
title: "My First Shellcode"
date: 2014-03-29T18:03:12-05:00
draft: false
---

Lately I’ve been wanting to explore some of what I call the “black magic” of infosec, reverse engineering and shellcoding. Whenever I see some posting or article that comes out on this I always become curious and bewildered at what I’m looking at because I just don’t understand it. On Twitter I’ve been seeing [@SecurityTube](http://twitter.com/SecurityTube) advertise their [x86_64 Assembly and Shellcoding on Linux](http://www.pentesteracademy.com/course?id=7) video course, so I decided to drop some money and learn what I can from it. I’ve yet to fully finish it, but more on that later. If I want to learn something, I usually have to do that something, get the hands on experience. I’m only about a third of the way through the course, only just going through assembly basics. But I decided to take a small break and try to write some shellcode of my own. I went exploring through the [Shellcode Database](http://shell-storm.org/shellcode/) and found a bit of x86 shellcode that I wanted to translate over to work on an x86_64 system. After some searching, I found one that [copies /etc/passwd to /tmp/outfile](http://www.shell-storm.org/shellcode/files/shellcode-864.php) written by [Paolo Stivanin](https://github.com/polslinux). Because of how trivial and simple looking the shellcode was, I decided to translate it over to work on x86_64.

After a few days of constant Googling and debugging and spending more time in gdb I’d care to admit, I finally got a working bit of shellcode that works on x86_64 Linux that copies the contents of /etc/passwd to /tmp/outfile. It is sitting at 118 bytes and contains no null bytes, so you’ll be able to inject it when necessary. I know it is nothing super fancy but it works and hopefully it can be found useful to someone.

For those that are interested in dissecting the following code, it is using the sys_open, sys_read, and sys_write syscalls. This might be able to use some optimization, so if anyone finds anything please let me know!

{{< gist chiggins 9820129 >}}

## Resources

* http://www.shell-storm.org/shellcode/files/shellcode-867.php
* http://blog.rchapman.org/post/36801038863/linux-system-call-table-for-x86-64
* http://man7.org/linux/man-pages/man2/open.2.html
* https://en.wikipedia.org/wiki/X86-64
* http://www.commandlinefu.com/commands/view/6051/get-all-shellcode-on-binary-file-from-objdump
