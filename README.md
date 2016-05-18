# debian-from-scratch

An instruction manual for teaching Linux From Scratch users how to make a custom Debian-powered distro.

##Why Debian from Scratch?

The original Linux from Scratch manual is purposefully vague as to what technique one should use to manage software dependencies. The suggestions that it gives, while no doubt being interesting exercises in package management, are not necessarily heavy-duty answers to a system administrator who intends on managing his time efficiently. 

The disadvantage of compiling everything to create a fully-fledged system is time. After one builds an LFS system for the first time, he/she is apt to realize that managing dependencies can be an arduous task, to say the least. Going through the insufferable exercise of hunting down dozens to possibly hundreds packages, mapping dependencies, configuring and installing these dependencies in the correct order, just to install a single piece of software, is not a viable alternative to a system administrator who values his time.

The answer to this problem, is obviously to use a package manager. There are many available package managers, the most popular of which are the Debian-based package management suites (dpkg and apt), and the Red Hat based package management suites (rpm and yum). 

This manual will teach you how to build your system utilizing the Debian set of package management tools by utilizing the temporary system environment created in Linux From Scratch.

## Purpose for this project

I chose to make this manual because I have seen woefully old guides on the internet teaching others how to get dpkg and apt running on their own custom linux, and people asking on various forums on how to install dpkg and apt, but not getting the help that they need. These guides are outdated and no longer contain up-to-date information, which I intend to fix here in this manual.

This project intends to be a community resource to help those interested in creating their own custom system from the ground up, while fully taking advantage of the Debian suite of package management, dpkg and apt, in order to solve the problems of package dependency installation and management.

## How to use this manual? 

This manual is designed to be used after completing all instructions up to the end of Chapter 5 of the Linux From Scratch book, version 7.9. One first follows the instructions of the original LFS book, and builds the temporary system that is created in LFS Chapter 5. One is required to have a fully-functional temporary system which the the outcome of Chapter 5. 

After completing the preliminary preparartions, one then consults this manual and follows it step-by-step.

Like the original LFS manual, when dealing with packages to be compiled, each section already assumes that you have extracted the source code and have changed your main directory into the main folder of the extracted content. However, when dealing with .deb files no such extraction is needed. One only needs to follow the instructions while having the .deb file in your current directory.

## Overview of our Method for Building a Custom Debian System

In the original Linux From Scratch book, we created a cross-toolchain, using the toolchain native to our system. We then used this cross-toolchain to create a native toolchain, which ended up being the temporary '/tools' system environment. This was Chapter 5's goal. We then used this temporary system to build our final system, which was Chapter 6's goal.

In Debian From Scratch, we branch off from the end of Chapter 5. Instead of using the toolchain and other utilies installed in `/tools` to compile every single part of the final system, we instead extend this toolset to include the minimum required dependencies for installation of Debian's package manager, dpkg.

We then compile and install dpkg as the first part of our final system, and use dpkg along with some clever dependency hacking to satisfy all remaining dependencies needed to install apt. This allows us to rely on apt for the overwhelming majority of tasks involving the installation of software onto our new system, and to allow us to avoid the exercise in tedium that is manual dependency management.

##Preparing the virtual kernel filesystem mount points
First we create the directories which are supposed to contain virtual kernel filesystems. These are filesystems which are located in memory only and created dynamically every time the kernel is loaded. 

Each kind has a different purpose. The `devpts` contains device files for each pseudo terminal on your system. The `proc` contains information about every single process. The `sysfs` contains driver and device information. The `tmpfs` is a freely usable space which programs may use to store information in memory. 

Since we have not built our kernel yet, we are forced to use the ones existing on our host system by mounting them in the appropriate locations in our target system. When our system is fully built, the new kernel will automatically mount these filesystems in their appropriate places.
```
mkdir -pv $LFS/{dev,proc,sys,run}
mknod -m 600 $LFS/dev/console c 5 1
mknod -m 666 $LFS/dev/null c 1 3
mount -v --bind /dev $LFS/dev
mount -vt devpts devpts $LFS/dev/pts -o gid=5,mode=620
mount -vt proc proc $LFS/proc
mount -vt sysfs sysfs $LFS/sys
mount -vt tmpfs tmpfs $LFS/run

if [ -h $LFS/dev/shm ]; then
  mkdir -pv $LFS/$(readlink $LFS/dev/shm)
fi
```

### Entering our chroot environment

We must now, as the `root` user on our host system, enter our base environment by changing our root directory into the final system's root directory, and use the temporary environment we've previously constructed to build our final system. Use the following command after you have become `root` on your host:

```
chroot "$LFS" /tools/bin/env -i \
HOME=/root                  \
TERM="$TERM"                \
PS1='\[\033[01m\][ \[\033[01;34m\]\u@\h\[\033[00m\]\[\033[01m\]]\[\033[01;32m\]\w\[\033[00m\]\n\[\033[01;34m\]$\[\033[00m\]> ' \
PATH=/bin:/usr/bin:/sbin:/usr/sbin:/tools/bin:/tools/sbin \
/tools/bin/bash --login +h
```

##Creating a standard Linux file structure
```
mkdir -pv /{bin,boot,etc/{opt,sysconfig},home,lib/firmware,mnt,opt}
mkdir -pv /{media/{floppy,cdrom},sbin,srv,var}
install -dv -m 0750 /root
install -dv -m 1777 /tmp /var/tmp
mkdir -pv /usr/{,local/}{bin,include,lib,sbin,src}
mkdir -pv /usr/{,local/}share/{color,dict,doc,info,locale,man}
mkdir -v  /usr/{,local/}share/{misc,terminfo,zoneinfo}
mkdir -v  /usr/libexec
mkdir -pv /usr/{,local/}share/man/man{1..8}

case $(uname -m) in
 x86_64) ln -sv lib /lib64
         ln -sv lib /usr/lib64
         ln -sv lib /usr/local/lib64 ;;
esac

mkdir -v /var/{log,mail,spool}
ln -sv /run /var/run
ln -sv /run/lock /var/lock
mkdir -pv /var/{opt,cache,lib/{color,misc,locate},local}
```


### Installing dpkg

Before we can install `dpkg`, we must first complete the dependencies needed for it to compile using the toolchain located in our `/tools` directory. The following figure shows the dependencies needed for `dpkg` to compile properly:  

<center>![](https://cdn.rawgit.com/scottwilliambeasley/debian-from-scratch/master/images/dpkg.svg)</center>
#####Figure 1 - The dpkg dependency tree

All of of these dependencies, only `gettext` has already been installed on our system. Thus we must compile all of these one by one until such a time comes where we have satisfied all of `dpkg`'s dependencies.

**autoconf** [![source](http://ftp.gnu.org/gnu/autoconf/autoconf-2.69.tar.xz)]
```
./configure --prefix=/tools
make
make install
```

**automake** [![source](http://ftp.gnu.org/gnu/automake/automake-1.15.tar.xz)]
```
sed -i 's:/\\\${:/\\\$\\{:' bin/automake.in
./configure --prefix=/tools
make
make install
```

**libtool** [![source](http://ftp.gnu.org/gnu/libtool/libtool-2.4.6.tar.xz)]
```
./configure --prefix=/tools
make
make install
```

**bison** [![source](http://ftp.gnu.org/gnu/bison/bison-3.0.4.tar.xz)]
```
./configure --prefix=/tools
make
make install
```

**flex** [![source](http://prdownloads.sourceforge.net/flex/flex-2.6.0.tar.xz)]
```
./configure --prefix=/tools
make
make install
```

**dpkg** [![source](http.debian.net/debian/pool/main/d/dpkg/dpkg_1.17.26.tar.xz)] 
```
./configure --prefix=/usr --sysconfdir=/etc --localstatedir=/var --build=x86_64-unknown-linux-gnu
make
make install
```

Notice how all of the prerequisite software needed to compile `dpkg` is still installed in `/tools`, while the `dpkg` binary itself is installed in the `/usr` directory (specifically `/usr/bin`), its configuration files being placed in `/etc`, and its local state directory (which holds the dpkg database and other files) will be located within `/var`(specifically `/var/lib/dpkg`).

###Installing apt

Before we can install `apt`, and use this to automatically install the most of the rest of our system software, we have to install its immediate dependencies on our target system first.

![](https://cdn.rawgit.com/scottwilliambeasley/debian-from-scratch/master/images/apt-dependencies.svg)
#####Figure 2 - The apt dependency tree, one level deep

Each of these immediate dependencies has their own set of dependencies to fulfill. We shall start by completing the dependency tree for `debian-archive-keyring`. Unlike the compilation process needed to install `dpkg`, the process we now use to install software is by installing .`deb` files using `dpkg`. 
