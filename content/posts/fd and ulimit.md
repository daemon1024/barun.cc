+++ 
date = "2020-11-01"
title = "File Descriptors and Resource Usage Limits"
slug = "File Descriptors and Resource Usage Limits" 
tags = ["Shell", "Open Source", "Bash", "Linux", "Unix", "ulimit", "File Descriptors", "System Resources"]
type = "post"
+++

## *Backstory*

I was working on a project in *go* where I had to open and play around with too many files at once ( around 50 thousand ). The program ran fine when I was working with one file at a time. But when I tried to make it concurrent, I had an error saying `Too many open files.`

Now that's weird cause my hardware wasn't the bottleneck, I had enough free ram, my CPU wasn't maxed out and I had plenty of empty storage. So I started wondering why would this happen.

So the best way would have been to just paste that error on the internet, but I played around a bit more. I thought maybe tens of thousands of files were a bit too much for the process to handle and maybe something was wrong with my code too. So I scaled it down to around a 100 files and it worked great. Then I moved to some 1200 files and boom that error occurred again. Now I was kinda certain that the issue wasn't with my code or hardware but something was limiting it on the OS. I searched around a bit and realized there are some Resource Usage Limits set by default and I was going beyond em.

## Resource Usage Limits and `ulimit`

On UNIX systems, the `ulimit` command controls the limits on system resource such as max memory size, open files, pipe size, stack size, max user processes, virtual memory etc. 

```bash
~ ❯❯❯ ulimit -a
Maximum size of core files created                           (kB, -c) unlimited
Maximum size of a processâ€™s data segment                     (kB, -d) unlimited
Maximum size of files created by the shell                   (kB, -f) unlimited
Maximum size that may be locked into memory                  (kB, -l) 64
Maximum resident set size                                    (kB, -m) unlimited
Maximum number of open file descriptors                          (-n) 1024
Maximum stack size                                           (kB, -s) 8192
Maximum amount of cpu time in seconds                   (seconds, -t) unlimited
Maximum number of processes available to a single user           (-u) 63483
Maximum amount of virtual memory available to the shell      (kB, -v) unlimited
```

So there are two types of limits, `soft` and `hard`. The hard limit is set by the system administrator or root user while the soft limit can be relaxed by the user up to the hard limit. By default `ulimit` shows the soft limit. We are particularly interested in max open file descriptors so we will be using the `-n` flag now.

```bash
~ ❯❯❯ ulimit -n
1024
~ ❯❯❯ ulimit -nS
1024
~ ❯❯❯ ulimit -nH
524288
```

So the soft limit was set to 1024 files and that was the reason for my program crashing. I could set any value up to 5424288 according to my needs and my issue would be solved. But how?

### Changing resource usage limits

You can simply pass on the value with `ulimit` to change the limit.

```bash
~ ❯❯❯ ulimit -nS 2048
~ ❯❯❯ ulimit -nS
2048
~ ❯❯❯ ulimit -nH
524288
~ ❯❯❯ ulimit -n -H 123456789
ulimit: Permission denied when changing resource of type 'Maximum number of open file descriptors'
```

So you don't have the permission to change your hard limit. But you can do so by editing `/etc/security/limits.conf` 

```bash
~ ❯❯❯ tail -5 /etc/security/limits.conf
#<domain>      <type>  <item>         <value>
*		        soft    nofile 	       1024
daemon1024	    soft    nofile 	       2048
daemon1024	    hard    nofile 	       5424288
# End of file
```

**How do I know what's the max amount of open files my system can handle?**

It's listed in `/proc/sys/fs/file-max`

```bash
~ ❯❯❯ cat /proc/sys/fs/file-max
9223372036854775807
```

So how does linux/unix identify how many files are opened by a process and you might have seen the term file descriptors pop up somewhere above. So let's see what are those.

## File Descriptors

So according to [Wikipedia](https://en.wikipedia.org/wiki/File_descriptor),

> In Unix and related computer operating systems, a file descriptor is an abstract indicator (handle) used to access a file or other input/output resource, such as a pipe or network socket.

If you didn't get it it's fine. **Everything is a file**  is one of the defining feature of UNIX and UNIX-like Operating System. 

In layman terms, every time you open a file the operating system creates an entry to represent that file and store the information about that opened file. 

This entry is stored in a **file descriptor table** and this table is kernel-resident. Each process has its own file descriptor table. An application passes the key to the kernel through a system call, and the kernel will access the file on behalf of the application, based on the key. The application itself cannot read or write the file descriptor table directly.

Hope you got to learn some new stuff through this blog post. Look forward to my future endeavors.

## Resources

- https://www.gnu.org/software/bash/manual/html_node/Bash-Builtins.html ( Search `ulimit` )
- https://en.wikipedia.org/wiki/File_descriptor