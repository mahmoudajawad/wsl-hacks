# WSL2 Development Environment

*CONTINUE AT YOUR OWN RISK: This is a hack guide meant to share ideas about how to overcome a specific situation in WSL eco-system. It's shared here for eductional purposes and not to be used in production environments. It hasn't beed tested thoroughly for any security issues. The methods explained in this document refer to customisation, modification and root-level permanent changes which might cause serious damage to the device it's used on and/or data loss. I take no responsibility for any incident that arise from the use of the methods mentioned in this document.*

---

When I got my Surface Pro 4, I was so excited to start using WSL in order to keep using the same GNU/Linux tools I love using for development and system administration tasks. And, although it was great addition to Windows world, it lagged huge time with performance, which left me using mixture of Windows and WSL environments. However, WSL2 came exactly to mitigate this issue and resolve all the performance woes and some low-level issues such as prevented docker from running on WSL[1].

Recently, I decided to get Surface Pro X as soon as it's available, but knowing much of the tools I use aren't yet compiled for aarch64-based SPX, I found the relieve in getting confirmation from the community WSL2 is working great, and allows developers to use it fully. I, then, decided to move my development environment completely into WSL2 in order to test such environment and make some judgment even before getting into SPX and ARM-based hardware with Windows 10 on ARM.

For starter, I would like to explain that I've always been a fedora/CentOS/RHEL user; Debian never appealed to me for reasons not part of this article, for that I didn't even try to get into WSL2 with default Ubuntu distros available in Microsoft Store. Instead I took this journey with two distros:
1. [CentWSL](https://github.com/yuk7/CentWSL)
2. [Fedora Remix](https://github.com/WhitewaterFoundry/Fedora-Remix-for-WSL)

All steps I took were applied to both distros and overall I had the same experience completely.

# Installation
Microsoft has perfect [guide on how to get WSL2 on your device](https://docs.microsoft.com/en-us/windows/wsl/wsl2-install), whether you had WSL or not. For this you just need to start with this guide and then set default distro version to 2 with:
```powershell
wsl --set-default-version 2
```
Once you have WSL ready to load GNU/Linux distros as WSL2, start with either:
1. Installation of [CentWSL](https://github.com/yuk7/CentWSL).
2. Get [Fedora Remix from Microsoft Store](https://www.microsoft.com/store/apps/9N6GDM4K2HNC?ocid=badge)

You can also do as I did and use both if you care much about testing all available options.

# Docker
One of the most advertised aspects of WSL2 is ability to use docker on WSL2. RHEL-based distros aren't exception and getting Docker to run on them is quite straightforward, quite not totally though.

## Adding Docker Repo
Both Centos and Fedora have their own stable versions of Docker in the official repos. However, to get latest versions it would be better idea to get official Docker repos for [Fedora](https://docs.docker.com/install/linux/docker-ce/fedora/) and [CentOS](https://docs.docker.com/install/linux/docker-ce/centos/).

## Installing Docker
Once you follow the previous guides to add Docker to your distro go ahead and install latest stable version with:
```bash
sudo yum install docker-ce docker-ce-cli containerd.io
```

## Running Docker
Here comes the annoying part--Systemd, which controls the services on modern GNU/Linux distros, isn't available with WSL regardless of the version in use. This is an ongoing issue that requires some work from both [systemd and Microsoft but it's unclear whether it's ever going to happen](https://github.com/systemd/systemd/issues/8036). For this, using the following usual RHEL-wise command to start Docker won't do it:
```bash
Sudo systemctl start docker
```
However, more direct method would be to simply run:
```bash
sudo dockerd
```
This command runs docker daemon in your current console. Move to new one and try running docker with:
```bash
sudo docker run hello-world
```

### Running Docker without sudo
This is not WSL-specific but worth bringing up since I realised some don't know it's quite an easy task, and Docker docs has the article for you: [https://docs.docker.com/install/linux/linux-postinstall/](https://docs.docker.com/install/linux/linux-postinstall/).

## Automating Docker Launch
Although the previous task to start Docker isn't much of hassle, yet it isn't the way I like it. To overcome the obstacle of requiring manual launching I created sample script which starts `dockerd` with `setsid` tool which can be used to start executables as background tasks.
```bash
if [ -z "$(ps aux | grep [d]ockerd | awk '{ print $2; }')" ]; then
        echo "Docker is not running! Attempting to start Docker"
        sudo setsid dockerd >/dev/null 2>&1 < /dev/null &
else
        echo "Docker is running!"
fi
```
I saved this script in home directory with name `docker-check` then I gave it executable permission:
```bash
chmod +x ~/docker-check
```
One last step is to give `setsid` tool `NOPASSWD` exception in `/etc/sudoers`. You can do this by opening the said file as root in vim:
```bash
sudo vi /etc/sudoers
```
Then move to line `111`, add new line:
```vim
:111 [Enter]
o
```
Then type (or paste) the following:
```
%wheel        ALL=(ALL)       NOPASSWD: /usr/bin/setsid
```
Then save file with:
```vim
:wq! [Enter]
```

### HELP! I SCREWED FILE UP!
If anything goes wrong in `/etc/sudoers` file you won't be able to use `sudo` anymore. To over come this start your WSL2 distro as root from another window:
```powershell
wsl -d DISTRO_NAME -u root
```
Then open `/etc/sudoers` in vim:
```bash
vi /etc/sudoers
```
And delete the line you added:
```vim
:112 [Enter]
dd
:wq! [Enter]
```

If everything goes alright you can now launch `dockerd` by simply running the script with:
```bash
~/docker-check
```

One last addition would be to add this script to `~/.bash_profile` file in order to make run every time you open new session in your distro:
```bash
vi ~/.bash_profile
```
Now, go to the end of the file, add new line, and type:
```
~/docker-check
```
Then save the file, exit vim.

Try, opening new console and you will be greeted with:
```
Docker is not running! Attempting to start Docker
```
Close the console and open it again, to get:
```
Docker is running!
```

# Git
Getting `git` can be done from the official repos of Fedora 31 in Fedora Remix distro, and although it's not latest, it's still very recent version. However, with CentWSL/CentOS it's quite obsolete. If you are on CentWSL consider either:

## Installing Git from IUS
"IUS is a yum repository that provides newer versions of select software for RHEL and CentOS", as mentioned on [their website](https://ius.io/). You can get IUS by following the guideline on [Get Started page](https://ius.io/setup) of the website. Then, install `git` with:
```bash
sudo yum install git2u
```

## Building Git from Source
The other more time requiring option is to build `git` on your own. DigitalOcean has perfect article on this: [https://www.digitalocean.com/community/tutorials/how-to-install-git-on-centos-7](https://www.digitalocean.com/community/tutorials/how-to-install-git-on-centos-7).

# Python
*[WIP]*

# NPM
*[WIP]*


# Visual Studio Code
Luckily, `vscode` has the ability to work with WSL natively using [Remote Development Extension](https://code.visualstudio.com/docs/remote/wsl). To begin, install the extension on your Windows `vscode` (not in WSL). Then, open your project folder in WSL console, then type:
```bash
code .
```
On first time run you will get something like:
```bash
Installing VS Code Server for x64 (8795a9889db74563ddd43eb0a897a2384129a619)
Downloading: 100%
Unpacking: 100%
Unpacked 2036 files and folders to /home/USER/.vscode-server/bin/8795a9889db74563ddd43eb0a897a2384129a619.
```
What happened here is you tried to run `code` which refers to `vscode` on Windows, and because it detected it's run from within WSL, Remote Development extension began to install `vscode` server on your WSL distro, enabling your Windows `vscode` to interact with your project as if it's run from inside WSL and has nothing to do with Windows at all.


# Happy Coding!
To all [HTML programmers](https://www.reddit.com/r/ProgrammerHumor/comments/e0iqkv/one_million_html_programmers) out there, I hope this guide is benefecial to you. This document mentiones guides focusing on tools I use on daily basis, however more tools are required by different users. If you come across this document and found it partially helpful by still missing a small tip, let me know in a comment to this comment so we can improve this document together.
