+++ 
date = "2020-07-24"
title = "Basic Shell commands and How do they execute?"
slug = "Basic Shell commands and How do they execute?" 
tags = ["Shell", "Bash", "Linux", "Unix"]
type = "post"
+++

## Shell

A shell is a command line interface for a UNIX-like operating system. A UNIX system offers a variety of shell types like

- **sh** or Bourne Shell
- **bash** or Bourne Again shell : It is the standard GNU shell used commonly by Linux users.
- **csh** or C Shell
- **ksh** or Korn Shell
- and many more . .

You can see what shell are available in your system by running `cat /etc/shell`

Since `bash` is the standard GNU shell, the focus of this blog will be that only.

## Execution of shell commands

There are three ways a command can be executed by `bash`

- Built-in
- General
- Shell Script

### **General**

`Bash` determines the type of program that is to be executed. Normal system commands exist in compiled form in your system. When such a program is executed , bash uses the `fork-and-exec` mechanism. This mechanism is a commonly used technique in UNIX and is used to create all UNIX processes. First the `fork()` process happens, a new child process is created where bash makes an exact copy of itself. The child process environment is identical to that of the environment including the input and output devices except the process id. If you are wondering what `process` is , it is nothing but a special instance provided by the system that consists of all the services/resources that may be utilised by a program during its execution. After the forking process , the address space of the child process is overwritten with the new data using `exec()` scall to the system.

### Built-In Commands

Bash `built-in` commands (also known as "internal command") are part of the shell itself. Each built-in command is executed directly in the shell itself, instead of an external programme which the bash executes the general way.

### How to differentiate between built-in and general commands

You can use the `type` command followed by a given command to find out the type of the command.

```bash
$ type cd
cd is a shell builtin
$ type type
type is a shell builtin              # type itself is a built in command
$ type ls
ls is /usr/bin/ls
```

As you can see `ls is /usr/bin/ls` so basically bash executes `/usr/bin/ls` the general way when `ls` command is called in shell.

### Scripts

When a shell script is being executed, bash `forks` a new process. This process reads the lines from shell script one line at a time. Each line is read, interpreted and executed as if you were typing those commands from the keyboard.

A sample bash script :

```bash
#!/bin/bash

echo "Hello World"

pwd

cd ..

pwd

type cd

echo "Bye"
```

Output :

```bash
$ ./script.sh
Hello World
/home/daemon1024/test/test1
/home/daemon1024/test
cd is a shell builtin
Bye
```

## Common shell commands

- `pwd` - The **print working directory** command. As the name suggests, it prints out which directory you are in currently.

  ```bash
  $ pwd
  /home
  ```

- `ls` - It is used to **list** the contents of a directory.

  ```bash
  $ pwd
  /
  $ ls
  bin   dev  home  lib32	libx32	    media  opt	 root  sbin  sys  usr
  boot  etc  lib	 lib64	lost+found  mnt    proc  run   srv   tmp  var
  $ ls home
  daemon1024  lost+found
  $ ls -l home
  total 20
  drwxr-xr-x 38 daemon1024 daemon1024  4096 Jul 24 14:45 daemon1024
  drwx------  2 root       root       16384 May 23 16:58 lost+found
  ```

- `cd` - The **change directory** command. You can change your working directory using this command.

  ```bash
  $ pwd
  /
  $ cd home
  $ pwd
  /home
  $ cd daemon1024
  $ pwd
  /home/daemon1024
  $ cd ..                    # .. takes us one directory up.
  $ pwd
  /home
  $ cd -                     # - takes us to the previous working directory.
  /home/daemon1024
  ```

- `mkdir` - The **make directory** command. You can create a new directory using this.

  ```bash
  $ pwd
  /home/daemon1024
  $ cd test
  cd: no such file or directory: test    # throws out an error
  $ mkdir test
  $ cd test
  $ pwd
  /home/daemon1024/test
  ```

- `cp` and `mv` - The **copy** and **move** command. **cp** command simply copies files and directories and the **mv** command moves.

  ```bash
  $ pwd
  /home/daemon1024/test
  $ ls
  hello  hello1
  $ mkdir test1            # make a new directory
  $ ls
  hello  hello1  test1
  $ cd test1
  $ ls                     # empty directory
  $ cd ..
  $ cp hello test1         # copy <source> <destination>
  $ cd test1
  $ ls                     # file copied
  hello
  $ cd ..
  $ mv hello1 test1        # move <source> <destination>
  $ ls
  hello  test1             # hello1 is not present in current directory now
  $ cd test1
  $ ls
  hello  hello1            # file moved
  $ pwd
  /home/daemon1024/test/test1
  ```

- `cat` - It is used to read files sequentially and write them to standard output.
- `head` and `tail` - The **head** command command prints lines from the beginning of a file, and the **tail** command prints lines from the end of files.

  ```bash
  $ cat hello.txt
  Lorem ipsum dolor sit amet,
  consectetur adipiscing elit,
  sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.
  Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat.
  Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur.
  Excepteur sint occaecat cupidatat non proident,
  sunt in culpa qui officia deserunt mollit anim id est laborum.

  $ head -n 3 hello.txt
  Lorem ipsum dolor sit amet,
  consectetur adipiscing elit,
  sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.
  # Prints 3 lines from the beginning

  $ tail -n 2 hello.txt
  Excepteur sint occaecat cupidatat non proident,
  sunt in culpa qui officia deserunt mollit anim id est laborum.
  # Prints 2 lines from the end

  $ tail -n +2 hello.txt
  consectetur adipiscing elit,
  sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.
  Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat.
  Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur.
  Excepteur sint occaecat cupidatat non proident,
  sunt in culpa qui officia deserunt mollit anim id est laborum.
  # Prints after skipping 2 lines from the beginning.

  $ head -n -4 hello.txt
  Lorem ipsum dolor sit amet,
  consectetur adipiscing elit,
  sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.
  # Prints after skipping 4 lines from the end.
  ```

- `man` - It is used to display the user manual of any command that we run in the shell.

  Syntax : `$man [OPTION]... [COMMAND NAME]...`

  It provides a detailed view of the command which includes `NAME`, `SYNOPSIS`, `DESCRIPTION`, `OPTIONS`, `EXIT STATUS`, `RETURN VALUES`, `ERRORS`, `FILES`, `VERSIONS`, `EXAMPLES`, `AUTHORS` and `SEE ALSO`.

  An example man page of ls.

  ![man-page](https://i.imgur.com/zNsUAFA.png)

  `man pages` are the most important and reliable source of any UNIX related query. To know more about man pages read [here](https://man7.org/index.html).

Hope you got to learn some new stuff through this blog post.

Special thanks to folks at [#dgplug](http://foss.training) for helping me learn new things in each session.

If you are interested in learning more about shell and command-line itself, here are some resources :

- [Bash Guide for Beginners](http://tldp.org/LDP/Bash-Beginners-Guide/Bash-Beginners-Guide.pdf)
- [The Art of Command Line](https://github.com/jlevy/the-art-of-command-line)