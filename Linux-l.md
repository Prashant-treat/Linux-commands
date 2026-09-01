# Linux (Terminal and Shells Guide)

## Table of Contents

- [Terminal](#terminal)
- [Shell](#shell)
- [Basic Commands](#basic-commands)
- [System Information](#system-information)
- [Variables](#variables)
- [Command History](#command-history)
- [Terminal Alternatives](#terminal-alternatives)
- [Filesystems](#filesystems)
- [Viewing Files](#viewing-files)
- [Editing Files](#editing-files)
- [File Operations](#file-operations)
- [Permissions](#permissions)
- [Executables](#executables)
- [Shebang](#shebang)
- [Shell Types](#shell-types)
- [Shell Configuration](#shell-configuration)
- [Environment Variables](#environment-variables)
- [PATH](#path)
- [Man Pages](#man-pages)
- [Flags](#flags)
- [Exit Codes](#exit-codes)
- [Standard Streams](#standard-streams)
- [Standard Input (stdin)](#standard-input-stdin)
- [Piping](#piping)
- [Interrupt & Kill](#interrupt--kill)
- [Unix Philosophy](#unix-philosophy)
- [top](#top)
- [Package Managers (APT)](#package-managers-apt)
- [Neovim Basics](#neovim-basics)
- [curl](#curl)
- [wget](#wget)
- [unzip](#unzip)
- [netcat](#netcat)
- [ss (Socket Statistics)](#ss-socket-statistics)

---

# Terminal

A **terminal** (or terminal emulator) is a program that accepts text-based commands and renders text output on the screen.

## Shell

A **shell** interprets the commands you type and executes them.

Shells are often referred to as **REPLs**, which stands for:

- **Read**
- **Eval** (Evaluate)
- **Print**
- **Loop**

---

## Basic Commands

### Print Text

```bash
echo "Hello World"
```

### Show Current User

```bash
whoami
```

---

# System Information

```bash
uname -a        # Show system information (kernel, architecture, etc.)
hostname        # Show system hostname
uptime          # Show how long the system has been running
whoami          # Show current logged-in user
top             # Show running processes (live view)
echo $SHELL     # Show current shell
```

---

# Variables

## Create a Variable

```bash
name="Lane"
```

## Use a Variable

```bash
echo $name
# Lane
```

## Interpolate in a String

```bash
echo "Hello $name"
# Hello Lane
```

---

# Command History

```bash
history
```

- Navigate: ↑ (up arrow), ↓ (down arrow)
- Clear terminal:

    ```bash
    clear
    ```

    or
    `Ctrl + L`

---

# Terminal Alternatives

- Ghostty
- Alacritty
- Windows Terminal

---

# Filesystems

All data is organized into files and directories in a tree-like structure called a **filesystem**.

```bash
ls /           # List root directory
ls             # List current directory
ls <filename>  # List specific file/directory
pwd            # Show current directory path
```

The first slash `/` represents the **root directory**.

Example filepath:

```
/home/wagslane
```

---

## Change Directory

```bash
cd            # Go to home directory
cd ..         # Go to parent directory
cd /          # Go to root directory
```

---

## Absolute vs Relative Paths

| Absolute Path                      | Relative Path          |
| ---------------------------------- | ---------------------- |
| `/vehicles/cars/fords/mustang.txt` | `fords/mustang.txt`    |
| From root                          | From current directory |

---

# Viewing Files

## Show Entire File

```bash
cat filename.txt
```

## Head (First Lines)

```bash
head filename.txt
head -n 10 file.txt
```

## Tail (Last Lines)

```bash
tail filename.txt
tail -n 10 file.txt
```

## Less (Scrollable View)

```bash
less filename.txt
less -N filename.txt   # Show line numbers
```

Press `q` to quit.

---

# Editing Files

## Nano (Beginner-Friendly)

```bash
nano filename.txt
```

## Vim / Neovim

```bash
vim filename.txt
nvim filename.txt
```

---

# File Operations

## Touch (Create File)

```bash
touch new_file.txt
touch file1.txt file2.txt
```

## mkdir (Create Directory)

```bash
mkdir my_directory
mkdir -p my_directory/subfolder/{folder1,folder2}
```

## Move / Rename

```bash
mv file.txt newname.txt
mv file.txt folder/
mv file.txt ../
```

## Remove

```bash
rm file.txt
rm -r directory
```

## Copy

```bash
cp file.txt folder/
cp -R directory new_directory
```

---

# Home Directory Alias

```bash
cd ~
```

`~` = home directory

---

# grep (Search Text)

```bash
grep "hello" file.txt
grep "hello" file1.txt file2.txt
grep -r "hello" .
```

---

# find (Search by Filename)

```bash
find . -name hello.txt
find . -name "*.txt"
find . -name "*word*"
```

---

# Permissions

Example:

```
drwxr-xr-x
```

## Meaning

- `d` → directory
- `-` → file

Permission groups:

- Owner
- Group
- Others

Symbols:

- `r` → read
- `w` → write
- `x` → execute

---

## Change Permissions

```bash
chmod -R u=rwx,g=,o= DIRECTORY
chmod +x file.sh
chmod -x file.sh
```

---

## Long Listing

```bash
ls -l
```

Example:

```
-rw-r--r--  1 user staff 1024 Feb 28 10:15 notes.txt
```

Columns:

1. Permissions
2. Links
3. Owner
4. Group
5. Size
6. Date
7. Name

---

# Executables

```bash
./program.sh
```

`.` means current directory.

---

# Shebang

```bash
#!/bin/sh
#!/usr/bin/python3
```

Tells system which interpreter to use.

---

# Shell Types

- **sh** – Bourne Shell (basic, POSIX)
- **bash** – Bourne Again Shell
- **zsh** – Z Shell (default on macOS)

---

# Shell Configuration

```bash
ls -a ~
```

- `.bashrc`
- `.zshrc`

Reload config:

```bash
source ~/.bashrc
```

---

# Environment Variables

```bash
export NAME="Lane"
echo $NAME
env
```

---

# PATH

```bash
echo $PATH
```

Add to PATH:

```bash
export PATH="$PATH:/path/to/new"
```

---

# Man Pages

```bash
man ls
man man
```

Search inside man page:

```
/search_term
n   # next
N   # previous
```

---

# Flags

```bash
ls -l
ls -a
ls -al
```

Conventions:

- `-a` → single-letter
- `--help` → long flag

---

# Exit Codes

```bash
echo $?
```

- `0` → success
- Non-zero → error

---

# Standard Streams

## stdout

```bash
echo "Hello"
```

## stderr

```bash
cat file.txt 2> error.txt
```

## Redirect stdout

```bash
echo "Hello" > file.txt
```

---

# Standard Input (stdin)

```bash
read NAME
```

---

# Piping

```bash
echo "Hello world" | wc -w
```

Pipe operator: `|`

---

# Interrupt

```
Ctrl + C
```

Sends SIGINT.

---

# Kill Process

```bash
ps aux
kill PID
```

---

# Unix Philosophy

1. Do one thing well
2. Programs work together
3. Handle text streams

Example:

```bash
grep "hello" file.txt | less
```

---

# top

```bash
top
```

Shows resource usage.

Alternative:

```bash
htop
```

---

# Package Managers

## APT (Ubuntu)

```bash
apt --version
sudo apt update
sudo apt install neovim
```

Check install:

```bash
nvim --version
```

Find location:

```bash
which nvim
```

---

# Neovim Basics

| Action      | Command |
| ----------- | ------- |
| Insert mode | `i`     |
| Exit insert | `esc`   |
| Save        | `:w`    |
| Quit        | `:q`    |
| Save & quit | `:wq`   |
| Force quit  | `:q!`   |
| Delete line | `dd`    |
| Copy line   | `yy`    |
| Paste       | `p`     |
| Search      | `/word` |

---

# curl

```bash
curl https://example.com
curl -I https://example.com
curl -o file.txt https://example.com
cat file.txt
```

Used for:

- API testing
- Downloading files
- Sending requests

---

# wget

```bash
wget https://example.com/file.zip
```

Download files.

---

# unzip

```bash
unzip file.zip
```

Install if missing:

```bash
sudo apt install unzip
```

---

# netcat (nc)

Used for:

- Open TCP/UDP connections
- Listen on ports
- Debug networking

---

# ss (Socket Statistics)

Modern replacement for `netstat`.

## Basic Usage

```bash
ss -t        # TCP
ss -u        # UDP
ss -l        # Listening
ss -tuln     # All listening ports
sudo ss -tulnp  # Show processes
```

Check specific port:

```bash
ss -tuln | grep 80
```

---

## sport vs dport

- `sport` → Source Port
- `dport` → Destination Port

```bash
ss sport = :22
ss dport = :80
```

Example:

```bash
ss -tn sport = :443
ss -tn dport = :22
```

---

If you'd like, I can also:

- Add a clickable Table of Contents
- Convert this into a printable PDF format
- Turn it into a clean GitHub-ready README
- Add diagrams for networking & filesystem

fine tuning needed and further notes can be add.


