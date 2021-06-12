---
layout: post
title:  "Windows Subsystem for Linux (WSL) 2: First Impressions"
date:   2019-06-14 12:00:00 +0200
categories: docker devops
image: https://i.imgur.com/beC1CvI.png
ascent_image:
  background: url('https://i.imgur.com/beC1CvI.png') center/cover
  overlay: false
invert_sidebar: false
---

Just over a day ago, WSL 2 was released to Windows Insiders in the Fast Ring. Half a year ago or so, I switched to MacOS because I was tired of not having a *good* Linux experience on Windows. Well Microsoft has heard the cries of Linux lovers, and are shipping with an actual, full fledged Linux kernel now!

# Usability & Setup

Before even getting started, of course WSL 2 had to be set up. To my suprise, it was incredibly simple to do. Download my preferred distro from the Windows Store (Ubuntu in this case), enter a few commands in powershell, and things were up and running. I won’t go into detail about installation, seeing as how there’s a really great guide to that [right here](https://www.thomasmaurer.ch/2019/06/install-wsl-2-on-windows-10/).

Now that things have been set up, I wanted to make some configurations to the shell, so I could feel at home. Lucky for me I’ve spent some time making an installation script for my config files, that’s been tested on both MacOS and Ubuntu when I made it. It was when I ran that, that it really dawned on me, that Windows now has an actual Linux kernel. No errors or anything. It just worked!

My next step was the real test of strength; Docker. I had seen the demos where they run Docker flawlessly, but I wanted to see it for myself before I believed it. Installed like I would otherwise on any other Ubuntu system. No trickery involved, Docker worked almost flawlessly. Tried a few different images, and they spun up just as quick as they do on my Macbook Pro.

You may have noticed I wrote ‘almost flawlessly’. One weird problem I ran into, was that I couldn’t curl my Nginx container at first. I had to put the container on the host network, in order for it to work. You can see the [issue here](https://github.com/microsoft/WSL/issues/4133).

# Networking

All in all, it seems that networking is one of the places that can definitely be improved. I won’t in any way say that it doesn’t work; there are just some minor quirks. One major change from WSL 1 to WSL 2, is that it is now a completely separate Virtual Machine. Took me a little time to figure out why I couldn’t just use localhost, when I wanted to connect to my X server. Turns out that WSL is now running with a host-only network adapter.

At first I expected to start a container in WSL on port 80, and then open chrome, type in `http://localhost` and access that container. Instead I had to type in the actual ip (found by ip addr) in chrome, in order to access the container. This isn’t a problem in and of itself, however it would be nice to have the ability to choose, how you want the container to be networked.

>  NOTE: Since this article was written, the WSL has updated this. While the WSL VM is still on a host-only network adapter, you can access things like docker containers through localhost on your Windows (host) machine

## Ping

One of the things that I can’t figure out, is why my ping sometimes won’t work. If I try to ping 8.8.8.8 from WSL 2, it works fine. However when I then try to ping some device on my home network, it fails 100% of the time. That’s not to say that I can’t connect to any of the devices. I can control my IoT devices. I can setup an X Server on my host and run things like gedit. Only catch there is that I have to export DISPLAY=<ip-of-host>:0.0, instead of export DISPLAY=:0 like I was used to.
>  UPDATE: Personally I no longer experience issues with ping

## Accessing WSL by IP

As mentioned, if you want to access any containers you have running on WSL, you have to access it by the ip. I quickly grew tired of this, and found a little workaround. Inside C:\Windows\System32\drivers\etc, you can edit the hosts file, and add the following line:

```bash
<ip-of-wsl>    wsl
```

Now you can simply type http://wsl in your browser, and access it that way.

# File Performance

Probably one of the biggest hurdles in WSL 1 was file performance. Because WSL was ‘simply’ a translation layer for system calls, something like git clone and yarn install could take an eternity. You may have heard that this has been severely improved in WSL 2, and the rumors are true!

Because WSL is now running as a VM, it is utilising an actual ext4 file system. Yes, this means that there are no translation layers, no magic or anything. Just raw powerful file performance in Linux; but there’s a catch. This is only when you are manipulating Linux files on Linux. Below you can see the incredibly unscientific benchmark I ran:

![](https://cdn-images-1.medium.com/max/3200/1*Wzq6ozYievIclbYFuyOhhg.png)

So what is happening here? I cloned the [react repo](https://github.com/facebook/react) into four different folders, and ran yarn install. In the top two corners, you are seeing native performance. On the left it’s Linux files on WSL, and on the right it’s a folder on my Windows host, running with powershell. Personally I think it’s fascinating that even native, Linux is 4 times faster than Windows.

>  **NOTE: **I am well aware that this is in no way a comprehensive benchmark. I simply wanted to show some form of comparison.

While all that is fascinating in and of itself, I find the bottom two even more fascinating. On the left, you see me trying to run yarn install in a folder mounted from my Windows host. 8 times slower! And yes you can really feel that slowness, even when just browsing through the different directories.

On the right you see Powershell trying to run yarn install in a folder, hosted on WSL. I honestly don’t know what is happening. It says that it finishes in just milliseconds, but it doesn’t actually do anything. No node_modules folder shows up. So as it is right now, it definitely is possible to manipulate files in Linux from Windows and vice versa. Something that you absolutely shouldn’t do in WSL 1.

>  A huge thank you to /u/gurnec on reddit, for telling me what went wrong in Powershell. Here’s the explanation I got, which I can confirm does work:
>  It tells you what’s happening with this:

```powershell
CMD.EXE was started with the above path as the current directory.
UNC paths are not supported.  Defaulting to Windows directory.
```

cmd (apparently yarn uses a cmd batch script) has never supported UNC paths (starting with \\). There's an easy workaround that I believe should work—in PowerShell do:

```powershell
PS C:\Users\Uname> subst z: \\wsl$\Ubuntu
PS C:\Users\Uname> cd z:\home\uname\Documents\react
PS z:\home\uname\Documents\react> yarn install
...
PS z:\home\uname\Documents\react> c:
PS C:\Users\Uname> subst z: /d
```

Under cmd it's even a bit easier—pushd will both create a new drive letter and change to it, and popd will undo this:

```powershell
C:\Users\Uname>pushd \\wsl$\Ubuntu\\home\uname\Documents\react
Z:\home\uname\Documents\react>yarn install
...
Z:\home\uname\Documents\react>popd
C:\Users\Uname>
```

(PowerShell also has pushd and popd, but they work differently and won't work as a workaround here.)

This stems from the fact that Microsoft are now using their 9P Protocol to mount the files. I won’t bore you with all the details, mostly because I don’t know them, but on a high level this means windows is mounting WSL as a network drive. That’s what makes it at least possible to change files whereever you want, without worrying about corruption. This also means that you can open the folder in explorer.exe, create the file in WSL, and it will instantly show up in explorer.exe.

# VSCode

My first thought when I found out file performance is so bad cross-system (for lack of better term), I was worried what this would mean for development. I can have my files on WSL, and manipulate them with amazing speed, but be in hell when using an editor to write code. On the other end, I can have the files on my Windows host, and write code fast, but have a horrible experience whenever I want to manipulate them in a shell. Enter VSCode.

This is probably one of the most exciting things to come out of this, in my opinion. Currently this feature is only in *preview*, but [Remote Development](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack) is an absolute godsend! Now that I’m done hyping it up, let me explain what it actually does.

From a user standpoint, it’s actually really simple. You can use WSL (both 1 and 2) as the backend, and VSCode as the frontend. This means you will get the experience of running VSCode on Linux, meaning that it has access to your Linux file system *directly*, but having the stability and support of the GUI running in Windows. Personally, I think this is the closest you can get to actually having the best of both worlds.

# The Decision to Use Virtualization

When I first heard the news of Microsoft shipping a Linux Kernel with Windows, I got really excited. My excited then died a little, when I found out it would ‘just’ be a lightweight VM. But boy was I surprised at how well this works.

This isn’t just a lightweight VM. It’s a hyper-optimised VM. Even from a complete shutdown, Ubuntu will start in just about a second. That’s almost as fast as opening iTerm on my Macbook. And once it’s open, you won’t even feel it. Looking in Task Manager, it uses 16.2MB RAM when it’s just been started. Compare that to Powershell using 34MB. Keep in mind that you don’t have to manage this VM; Windows does that for you. It automatically allocates more memory when needed, and removes it when you don’t.

Because of the incredible startup times, Microsoft feel confident in shutting down the VM when you’re not using it. After a small period of inactivity, it will check if you have any jobs or daemons running. In the case that you don’t, it will simply terminate the VM. However the goal is that you shouldn’t feel the fact that it’s been turned off, and in my case of a 5 year old laptop, that promise does in fact hold up.

# Final Thoughts

In my opinion, this is a huge step in the right direction. Yes it’s not flawless, but it’s been out for less than two days, so I think we can cut them some slack. Whether you are a fan of Microsoft or not, this is a very big move for them. Combining this with the new Terminal they are working on, and you’ve got a really stellar product on you hands.

If they manage to fix the few quirks present at the moment, I’m really excited for the future of Windows. They really lost me last year, hence why I’m now using a Mac. However seeing the direction they are moving in, both with internal practices and the move to open source, they are starting to win me back. I’m going to make a bold claim, and say that this time next year, there won’t be a discussion about MacOS vs. Windows. Windows will be the clear winner for developers. The discussion by then will be about Apple vs. Microsoft; not the operating systems that they have built. At least that is if Microsoft keeps moving in this direction with their OS.

If you want to know more about WSL 2, there are two great videos available on youtube from the MSBuild conference: [A deep dive here](https://www.youtube.com/watch?v=lwhMThePdIo), and [a small interview here](https://www.youtube.com/watch?v=9ZqeyTjX0TQ).
