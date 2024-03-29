---
title: A .NET codger getting into Linux on Windows
date: 2017-05-17
categories: linux wsl
---

Running .NET through Visual Studio with Microsoft SQL Server and onto a Windows Server instance has been that comfortable jumper for many years. As much as the landscape changed from Web Forms to MVC, stored procedures to Entity Framework, you still had that sense of familiarity that Microsoft won't let you down. That rubber stamp of Microsoft quality has been an easy sell. You could say:

> We're a Microsoft shop

and even if you didn't have any certification, clients would know that the company you worked for did things properly. Choosing a new Microsoft technology to plug into it meant that no questions would be asked. 

That's all changed. If you feel stitched into the comfy Microsoft jumper, it's time to look beyond it because of _cost_. A Windows Server instance is expensive. A SQL Server instance is expensive (especially if you need database level encryption). Visual Studio is expensive. I'm very productive with Visual Studio and .NET so I want to keep using those but I find it more difficult to justify the cost. There is quality elsewhere, the world is moving on and so is Microsoft.

## Getting into Linux

Before you defenstrate your Windows 10 box with a sharp toe punt, you'll be pleased to read that you can learn Linux from within the comfort of your own Windows environment. Microsoft have embedded Linux into something called the Windows Subsystem for Linux (WSL) since the "Anniversary Update". You could do this with a VM but that would take lots of extra resources and you can run Windows programs side-by-side. It's free but a bit of a pain to get it running:

[WSL Installation Instructions](https://msdn.microsoft.com/en-us/commandline/wsl/install_guide)

## Useful things about the WSL

The WSL (at time of writing) runs [Ubuntu 14.04](http://releases.ubuntu.com/14.04/). This is a Linux distribution (distro), which is a whole operating system made up from the Linux kernal and a package manager. Packages are tiny little applications and libraries. The bare distro might not have many packages on it but it. As you use your WSL more, you'll find that you're installing packages a lot.

Once you have the WSL running, check out the version you're on:

`$ lsb_release -a`

 You can update all the packages by:

`$ sudo apt-get update`

`sudo` means that the command will run with super user priviledges (it's short for **s**per **u**ser **do**). `apt-get` is the name of the package manager for Ubuntu (and Debian) distros. And `update` is the command itself. `apt-get` will run off to its preferred repository and download the latest packages. If there are any updates available, it will tell you each time you run it.

The command window you're typing into (also called a shell) uses a scripting language called bash. Bash is super-powerful but doesn't look like batch or Powershell, so it's a new skill to learn.

### Where is the C drive?
The C drive is "mounted". You can use <kbd>tab</kbd> for autocomplete on longer folder names. 

`$ cd /mnt/c`

> Linux is case sensitive!

### Can I run my windows programs?
Yes, although it will spawn the program in Windows normally as if you had use cmd. Try running dear old notepad:

`$ /mnt/c/Windows/System32/notepad.exe`

## Path variables and command prompt customisation
I'm a sucker for having things just so. I have a [scripts git repository](https://github.com/brainwipe/scripts) that I'm building up and I want a path to the cloned repository on my local machine. To do that you need to edit a file called `.bashrc`, which is a shell script that is run when the shell first starts.

You can edit it using [nano](https://www.howtogeek.com/howto/42980/the-beginners-guide-to-nano-the-linux-command-line-text-editor/) (there are many others) like this:

`$ nano ~/.bashrc`

The `~/` is a shortcut for saying "the current users home directory", which is not the Windows one but instead located in `/home/username` such as `/home/brainwipe`.

There's a bunch of stuff in `.bashrc` to begin with, you can add more at the bottom. I can add new paths like this:

`PATH=$PATH:/mnt/c/Projects/brainwipe/scripts/bash`
 
As the `PATH` variable is a big, long single string (like it used to be shown in Windows), you are actually appending your new path (in this case `/mnt/c/Projects/brainwipe/scripts/bash`) onto the existing variable `$PATH`.

The other thing I like to change is the command prompt. I like to have the time and working directory. That's just another line in the `.bashrc` file.

`export PS1="\t \w >"`

Will give: `13:36:36 /mnt/c/Projects >`

`PS1` stands for _Prompt String One_. You can find loads of example of prompt coolness on [make tech easier]https://www.maketecheasier.com/8-useful-and-interesting-bash-prompts/) and all over the web!

## Further Reading

- [Microsoft WSL FAQ](https://msdn.microsoft.com/en-us/commandline/wsl/faq)