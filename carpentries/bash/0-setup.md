# Summary and Setup

- The UNIX shell has been around for a long time.
- Allows users to perform complex tasks, often with just a few keystrokes or lines of code.
- Can be used to automate repetitive tasks.

## Steps

- Download files: https://swcarpentry.github.io/shell-novice/data/shell-lesson-data.zip.
- Install software: https://carpentries.github.io/workshop-template/install_instructions/#shell
- Open a new shell:
  - Windows:
    - Git for Windows: https://carpentries.github.io/workshop-template/install_instructions/#shell
    - Windows Subsystem for Linux: https://learn.microsoft.com/en-us/windows/wsl/install
  - Mac:
    - Terminal: https://www.macworld.co.uk/feature/mac-software/how-use-terminal-on-mac-3608274/
  - Linux:
    - Gnome Terminal: https://help.gnome.org/users/gnome-terminal/stable/
    - KDE Konsole: https://konsole.kde.org/
    - xterm: https://konsole.kde.org/

# Introduction

- User Interfaces
  - Graphical User Interface
  - Command Line Interface

- Key Points
  - The shell is a program whose purpose is to read commands and run other programs.
  - This lesson uses Bash. If you have ZSH, it's probably fine.
  - You can run programs in bash by entering commands on the command line.
  - You can do a lot with just a few commands.
  - Steeper learning curve than a GUI.

# Navigating Files and Directories

## Intro

- The file system manages files and directories on your computer.
- You may have heard directories called "folders."
- You can find out where you are by running `pwd`
- A slash has two meanings depending on where it is in the file path.
- You can see the contents of your current working directory using `ls`
- `ls -F` will format some files differently.
  - A trailing slash indicates a directory
  - A trailing asterisk indicates an executable file.
- You can clear your terminal screen using `clear`.

## Getting Help

- You can get help in a couple of different ways.
  - On Linux and in Git Bash, you can type `ls --help`
  - On Linus and MacOS, you can type `man ls`.
- The `--help` option will display a help message for many commands.
- If you type a command-line option that isn't recognized, you'll get an error.
- Exercise:
  - What happens if you use the `-l` option for `ls`
  - What happens if you use the `-l` and `-h` options together?
- Exercise:
  - The `ls -t` option lists contents in chronolgical order by the time the file was last modified.
  - The `ls -r` option lists contents in reverse order.
  - What happens if you use the `-r` and `-t` options together?

## Exploring Other Directories

- `ls -F Desktop`
- `ls -F Desktop/shell-lesson-data`
- `cd` to change your working directory
- We could use this set of commands to get to the `exercise-data` directory.
  - `cd Desktop`
  - `cd shell-lesson-data`
  - `cd exercise-data`
- Use `cd ..` to go up the directory tree.
  - `..` is a special directory referring to the parent of the current directory.
  - `ls -a` can be used to include files with names that begin with a dot.
- Use `cd` without any arguments to go to your home directory.
- We could also do this to get to the `exercise-data` directory
  - `cd Desktop/shell-lesson-data/exercise-data`
- So far, we've been using relative paths.
- An absolute path begins with a slash, and this is the path that's displayed when you type `pwd`.
- The tilde (`~`) character refers to the current user's home directory.
- The hyphen (`-`) character refers to the previous working directory.

## General Syntax of a Command

- Prompt
- Command
- Options
- Arguments

## Nelle's Pipeline - Organizing Files

- Tab completion

# Working With Files and Directories

## Creating Directories

- First, check to see where we are.
- Switch to `exercise-data/writing` directory.
- List files: `ls -F`
- Create a directory `mkdir thesis`
- List files in the directory: `ls -F thesis`
- The `mkdir` command can create multiple directories at once using the `-p` option.

- `mkdir -p ../project/data ../project/results`
- List files in the project directory.
  - `ls -FR ../project`
- Good practices for naming:
  - Avoid spaces.
  - Don't begin names with a dash.
- Create a text file.
  - `cd thesis`
  - `nano draft.txt`
- Editors
  - Nano
  - Vim
  - Emacs
  - VSCode
  - Notepad++
- Creating files in a different way.
  - `touch my-file.txt`
- Moving Files and Directories
  - `cd ~/Desktop/shell-lesson-data/exercise-data/writing`
  - `mv thesis/draft.txt thesis/quotes.txt`
  - `mv thesis/quotes.txt .`
  - `ls thesis`
  - `ls thesis/quotes.txt`
  - `ls quotes.txt`
- Moving files to a new folder
