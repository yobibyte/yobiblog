#Setting up a dev environment

During my dev life I tried a huge amount of ways of creating a convenient environment for development. There has been almost something that I didn't like, and I tried more. And today I will explain another way that I'm currently using. The most attractive thing about it: you can use any major OS (GNU/Linux, OS X or Win (yes, even win)) and be completely happy with it.

Yes, today we have a bunch of platform-free languages that enables us to use the OS we like for development. But for me it has been always easier to set up a dev environment in GNU/Linux as there are cool package managers that will make your life more comfortable (or sometimes harder when you have problems with dependencies).

The main idea is to create a GNU/Linux virtual dev machine, but use it only for code execution and some accompanying stuff like version control, for instance. All the code will be kept on the host machine and edited also from the host.

##  Steps
I use VirtualBox, so, I do not know how to do the same stuff in VM_something, but I'm pretty sure that most of the actions should be really similar or even the same.

* VB installing
  * if you are reading this, you know how to install a VB on your computer.
  * Create a VM and install a GNU/Linux distro inside
    * 15GB of space
    * 4096MB of memory
    * Processor VM settings tab
      * Enabled 3D acceleration in processor tab (you open GUI version of VM, it will work faster)
      * PAE/NX is enabled
    * Acceleration VM settings tab
      * Enable VT-x/AMD-V
      * Enable Nested Paging
    * Network VM settings tab
      * Add new rule for port forwarding. You can put only ports there (for me is 2222 on host and 22 on guest)

* Now it's time to install guest additions inside VM in order to enable auto-mount of the shared folder.

```bash
sudo apt-get install build-essential linux-headers-generic virtualbox-guest-x11
```

* Now we install ssh daemon that will enable us to connect to VM through console.

```bash
apt-get install openssh-server
sudo service restart ssh
```

* The following command will start our VM without running the X server

```bash
VBoxManage start <vmname> --type headless
```

* Now, you need to generate an ssh-key on your host machine to log in without entering the password

```bash
ssh-keygen -t rsa -b 4096 -C "yourmail@mail.com"
ssh-copy-id user@127.0.0.1 #Copy the key to our VM
```

* Now we can connect to our VM

```bash
ssh user@127.0.0.1 -p 2222
```

* If you need some additional ports for dev(web service, for example, it's easier to forward ports through ssh)

```bash
ssh user@127.0.0.1 -p 2222 -L 4000:127.0.0.1:4000 #local_port:<VM IP>:<VM port>
```

* In VM Shared Folders settings tab:
  * choose shared folder host machine path
  * choose the name
  * choose "auto-mount" and "Make Permanent"

* Add user to vboxfs group

```bash
 sudo usermod -a -G vboxsf <username>
```
* Restart VM
* Now /media/sf_yourdirname is your shared folder from the host
* Now you can do your favorite 'npm install' or any other 'hipster-technology install' and enjoy the experience.

<img class='center' src="https://github.com/yobibyte/yobiblog/blob/master/pics/no_idea_dog.jpg"/>


## What is good

* Current system is not changed. There will never be such a situation when you installed a library and broke half of your system. The main system should be stable and not change every day.
* Stackoverflow is full of posts about how to resolve screen resolution problems in VM, copy-paste problem or something like that. As we code in the host machine, our eyes are happy and we can use our super 100500K monitors to code. The full power and convenience of your host system is at your disposal.
Easy browsing and copy/paste and all the stuff
* Easy replicating and backups

## What is not so good

* No access for GPU, so Deep Learning guys will be sad. But this configuration is more laptop-like for me. For GPU usage we should have a high-end computer with ssh access to it.
* Yes, performance will be less than if we used our bare hardware to execute code. But in most cases it won't be an issue. Return to the previous bullet point.
* I still don't know how to use <your favorite IDE> and its debugger with this config. In theory it's possible like [here](https://www.jetbrains.com/help/idea/2016.1/run-debug-configuration-remote.html?origin=old_help) or [there](https://www.jetbrains.com/help/pycharm/2016.1/remote-debugging.html?origin=old_help). But as it's been a time since I coded a really big project, a text editor like Atom or Vim is completely enough for me.
* Yes, we can use Vagrant. I tried, but switched back again.
