# WSL2 Networking

*CONTINUE AT YOUR OWN RISK: This is a hack guide meant to share ideas about how to overcome a specific situation in WSL eco-system. I't s shared here for eductional purposes and not to be used in production environments. It hasn't beed tested thoroughly for any security issues. The methods explained in this document refer to elevated access in Windows which might cause serious damage to the device it's used on and/or data loss. I take no responsibility for any incident that arise from the use of the methods mentioned in this document.*

---

WSL2 is a major upgrade to WSL. Rather than implemting a Linux/POSIX-compatible runtime environement as in version 1, it directs all the calls an optimised Linux kernel served as VM along WSL eco-system.

This change does have many side effects from version 1, but one major change that left me unable to use WSL2 was networking. Accessing WSL2 distros is much like having a full, standlone VM running on your host and connected with virtual bridge. This is perfect for projects where you need to test real network traffic rather than simple `localhost` calls. This also means, the ease of starting a service on WSL2 distro and connecting to it from Windows using `localhost:[PORT_NUMBER]` is gone*. Following I would explain two different cases, (WSL to Windows) network communication[#wsl-to-windows] and (Windows to WSL) network communication[#windows-to-wsl].

## WSL to Windows
I'm starting with the easy bit. To access any service started on Windows from WSL2 distro, consider the following conditions:
1. Add `Inbound Rule` exception for the service you have on Windows in Windows Firewall on `Private, Public` networks.
2. Start your service to listen on all IPs or on your Windows bridge IP. Starting the service to listen on `localhost` and/or `127.0.0.1` would make it inavailable for your WSL2 distro.
3. Use the correct IP to connect to Windows.

To add rule for any service you have started on Windows, you need to understand the security risks bound to this. This means you are opening the service not only to WSL but to external world. Learn more about what this could mean before continuing. The steps to allow an app through Windows Firewall are:
1. Hit `Win` key on your keyboard.
2. Type `Windows Firewall` and pick `Windows Defender Firewall with Advanced Settings`.
3. From the left bar choose `Inbound Rules`.
4. From the right bar click `New Rule`.
5. On first step of the wizard `select` `Program`.
6. On second step use the `Browse` button to get the location of the executable that is running the service you would like to access from WSL.
7. On third step select `Allow the connection`.
8. On forth step select all the available options `Domain`, `Private` and `Public`.
9. On last step type in a distinguishable name that you can remember so that you can remove this rule in the future when you are done with it, and hit `finish`.

With this you allowed the service to be accessed from WSL2 distro. Next is to connect to the service.

Start your WSL2 distro app. Copy and paste the following command:
> Disclaimer: Do not use commands posted on the internet in any context without understanding what they do. There's always a chance of security risks invloved.

```bash
ipconfig.exe | awk '/WSL/ {getline; getline; getline; getline; print substr($14, 1, length($14)-1)}'
```
The previous command would extract the host, Windows, IP from `ipconfig.exe` Windows program using WSL [`interop`](https://docs.microsoft.com/en-us/windows/wsl/interop) feature. Once you hit `Enter` your terminal would print out Windows IP from WSL bridge. Now copy this IP and use it in your service client in WSL to connect to your Windows service.

To take things to next level, you can save the output of the previous command as an env variable like:
```bash
export WSL_HOST_IP=$(ipconfig.exe | awk '/WSL/ {getline; getline; getline; getline; print substr($14, 1, length($14)-1)}')
```
Now you have an env variable `WSL_HOST_IP` that you can use to access your Windows services like:
```bash
ping $WSL_HOST_IP
```
To take things to next level, you can append the last command to the end of your `~/.profile` for your debian-based distro or `~/.bash_profile` for your redhat-based distro, to make the env variable `WSL_HOST_IP` always available in all contexts as well as updated everytime you start your WSL2 distro.

Another aspect is to add a `hosts` file entry for Windows to allow all the apps that can't use env variables from accessing your Windows services. To acheive this use the following command:
```bash
echo -e "\n$WSL_HOST_IP windows.local" | sudo tee -a /etc/hosts >/dev/null
```
This command would append a new line to your `/etc/hosts` with host `windows.local` available for your use whenever needed. Keep this command handy in a file in your home directory since your `hosts` file would be regenerated after every restart. For instance you can use this:
```bash
echo 'echo -e "\n$WSL_HOST_IP windows.local" | sudo tee -a /etc/hosts >/dev/null' > ~/update_windows_host
chmod +x ~/update_windows_host
```
Now, you have this file available for your use after every restart for Windows.

## Windows to WSL
> Starting Windows 10 Build 18945, WSL has the ability to share ports between Windows and WSL without the need to this hack.

\* WSL team has stated they might consider ports bridging/sharing between Windows and WSL.
