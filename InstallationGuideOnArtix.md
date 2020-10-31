
[//]:# (author: minjiphobia\ date: Oct 23, 2020)

# nachos4.1 installation guide on artix linux
While there are plenty of tutorials on how to install [nachos](https://en.wikipedia.org/wiki/Not_Another_Completely_Heuristic_Operating_System) on ubuntu, which is the most used linux distro for educational purpose in china. It's hard to retrieve such infos for other distros like artix which I'm currently using. Therefore, I'd like to share what I've encountered during the installation of nachos on artix linux and how I make it work.

**NOTE**: There's no systemd-related changes in following procedure, so i suppose this guide works well on archlinux and manjaro too.

## Download the sources of nachos4.1  
> `wget https://www.u-aizu.ac.jp/~yliu/teaching/os/NachOS-4.1a.tar.gz`  
`tar xvf NachOS-4.1a.tar.gz` 

I rename the package to 'nachos' for simplicity. It's optional. Remember to replace 'nachos' with the original name in following commands if you don't take this step.
> `mv NachOS-4.1a nachos`  


## Install 32bit runtime libs and gcc-multilib

The build of nachos requires 32bit runtime libraries and old gcc version which is as well capable of compiling 32bit program.  

For 32bit runtime libs, firstly enable lib32 repository of pacman. To do this, uncomment lib32 and following *Include* statement at /etc/pacman.conf. If you havn't made any modification on pacman.conf, you can simply do:
> `sudo sed -i 's/#\[lib32\]/[lib32]/' /etc/pacman.conf && sudo sed -i '/\[lib32\]/{n;s_.*_Include = /etc/pacman.d/mirrorlist_}' /etc/pacman.conf`  

Now update pacman datebase and install 32bit runtime libs
> `sudo pacman -Syy lib32-glibc lib32-glib2`    

To make gcc compile 32bit program on x64 architecture, we need gcc-multilib. Arch-like distros are shipped with the newest gcc which supports multilib. However, compilation of nachos requires much older version of gcc. Sad for archers, the oldest gcc we can downgrade to is gcc-8\*, which isn't old enough. This is where the [AUR](https://aur.archlinux.org/)(Arch User Repository) comes in clutch. There are two ways to build gcc49-multilib through AUR.(After many building failures of other packs, this one went smoothly on my artix. If you don't want to bother testing which version is compatible with nachos, just take it.)  
- manually make package  
make a directory for your manually built package and cd into it. type:
> `git clone https://aur.archlinux.org/gcc49-multilib.git`  
`cd gcc49-multilib`  
`makepkg -si`  

if the building process takes too long, you can get all your cores working. Firstly fetch how many cores you have by `cat /proc/cpuinfo`. Then edit /etc/makepkg.conf, changing `MAKEFLAGS="-j1"` at line 45 to `MAKEFLAGS="-j{your core num}`


- use aur-helper  
if you have one of the aur-helpers installed, taking yay for instance, you can simply type:
> `yay -S gcc49-multilib`  

## Modify sysdep.h and Makefile 

Assuming you're in nachos directory, edit code/lib/sysdep.h, replacing *include \<iostream.h\>* with *include \<iostream\>* at line 15 and add *using namespace std;* at line 19. You can simply copy-paste this command:
> `sed -i '15s/.*/include <iostream>/' code/lib/sysdep.h && sed -i '19iusing namespace std;' code/lib/sysdep.h`  

Finally we come to the part of Makefile. There are totally seven fields we need to modify.
- CFLAGS: gcc options  
delete `-fwritable-strings`, add `-fpermissive`, `-m32`
- LDFLAGS: ld options  
add `-m32`
- CPP: c preprocessor  
the default path works on ubuntu but not on artix. to get the right binary, open a terminal and type `which cpp`, fill the output here. it's located at `/usr/bin/cpp` for me
- CC: compiler or whatever you call it  
change to `/usr/bin/g++-4.9`
- LD: linker  
change to `/usr/bin/g++-4.9`
- AS: assembler  
add `--32`
- RM: executable for removing files  
usually u don't have to change this field. if sth went wrong(chroot or sth like that), type `which rm` and fill the output here  

TL;DR: copy-paste commands below:  
**NOTICE**: it will use binary paths of my artix. if sth went wrong please refer  to the details above.
> `sed -i '203s/.*/CFLAGS = -ftemplate-depth-100 -Wno-deprecated -g -Wall $(INCPATH) $(DEFINES) $(HOSTCFLAGS) -DCHANGED -fpermissive -m32/' code/build.linux/Makefile && sed -i '204s/.*/LDFLAGS = -m32/' code/build.linux/Makefile && sed -i '207s_.*_CPP= /usr/bin/cpp_' code/build.linux/Makefile && sed -i '208s_.*_CC = /usr/bin/g++-4.9_' code/build.linux/Makefile && sed -i '209s_.*_LD = /usr/bin/g++-4.9_' code/build.linux/Makefile && sed -i '210s_.*_AS = as --32_' code/build.linux/Makefile && sed -i '211s_.*_RM = /bin/rm_' code/build.linux/Makefile`  

Now everything is up, go to build direcory and build nachos
> `cd code/build.linux`  
`make depend`  
`make`

You're good to go!
