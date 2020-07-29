# wsl2-vnc
Resources to run linux-based VNC that can be remotely controlled


## Introduction
This repo is only to consolidate the resources I have found to enable linux-based VNC server on WSL2. This is currently as of Jul 2020. This worked for me and so I wanted to write this down. Since WSL2 is currently at beta, I expect this to change and be less difficult later.

## Step 1: Enable Beta Windows to be installed
Join the [Windows Insider Program](https://insider.windows.com/en-us/)
Go to Settings in Windows and update until the build is at least 19041.
Also, download Windows Terminal in Microsoft Store.

## Step 2: Install Ubuntu with WSL2
Open a PowerShell in the [Windows Terminal](https://www.microsoft.com/en-us/p/windows-terminal/9n0dx20hk701).
Execute the following
```
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
wsl --set-default-version 2

```
Then download the latest [Ubuntu](https://www.microsoft.com/en-us/p/windows-terminal/9n0dx20hk701) from Microsoft Store.
Add this section to the settings text file of Windows Terminal
```
{
    "acrylicOpacity" : 0.75,
    "closeOnExit" : true,
    "colorScheme" : "Campbell",
    "commandline" : "ubuntu",
    "cursorColor" : "#FFFFFF",
    "cursorShape" : "bar",
    "fontFace" : "Consolas",
    "fontSize" : 10,
    "guid" : "{GUID}",
    "historySize" : 9001,
    "icon" : "Absolute path of icon",
    "name" : "Ubuntu",
    "padding" : "0, 0, 0, 0",
    "snapOnInput" : true,
    "startingDirectory" : "%USERPROFILE%",
    "useAcrylic" : true
}
```

## Step 3: Install oh-my-zsh and vncserver on Ubuntu
Open a new Ubuntu window in Windows Terminal (with admin permission).
Then execute
```
sudo apt install zsh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
sudo apt update
sudo apt install xfce4 xfce4-goodies xorg dbus-x11 x11-xserver-utils
sudo apt install tigervnc-standalone-server tigervnc-common
```
## Step 4: Install systemd-genie
The following is based on contribution from [tdcosta100/WSL2GUIXvnc-en.md](https://gist.github.com/tdcosta100/385636cbae39fc8cd0937139e87b1c74)
The details are copied here in case the page is gone.
```
sudo apt install tasksel
sudo tasksel
```
Select the desired GUI package
```
wget https://packages.microsoft.com/config/ubuntu/20.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
sudo apt update
sudo apt install dotnet-runtime-3.1
curl -s https://packagecloud.io/install/repositories/arkane-systems/wsl-translinux/script.deb.sh | sudo bash
sudo apt install systemd-genie
sudo mv /usr/bin/Xorg /usr/bin/Xorg_old
sudo nano /usr/bin/Xorg
```
In the nano window, paste the following
```
#!/bin/bash
for arg do
  shift
  case $arg in
    # Xvnc doesn't support vtxx argument. So we convert to ttyxx instead
    vt*)
      set -- "$@" "${arg//vt/tty}"
      ;;
    # -keeptty is not supported at all by Xvnc
    -keeptty)
      ;;
    # -novtswitch is not supported at all by Xvnc
    -novtswitch)
      ;;
    # other arguments are kept intact
    *)
      set -- "$@" "$arg"
      ;;
  esac
done

# Here you can change or add options to fit your needs
command=("/usr/bin/Xvnc" "-geometry" "1024x768" "-PasswordFile" "${HOME:-/root}/.vnc/passwd" "$@") 

systemd-cat -t /usr/bin/Xorg echo "Starting Xvnc:" "${command[@]}"

exec "${command[@]}"
```
Then open the config file by 
```
sudo nano /usr/lib/genie/deviated-preverts.conf
```
And paste the following
```
{
  "daemonize": "/usr/bin/daemonize"
}
```

Then execute
```
sudo chmod 0755 /usr/bin/Xorg
genie -s
```

If the kernel is shutdown and restarted, you only need to run the last line to re-boot the VNC session.

## Step 5: Port forwarding
If you stopped here, you will be able to access the VNC session by `localhost:0` from the same machine. However, if you wish to access it from other computer on the same network, you will have to forward the ip and port using windows.

This is directly copied from this [contribution](https://github.com/microsoft/WSL/issues/4150).

Paste the following to a text file named `wsl2port.ps1` in Windows.
```
$remoteport = bash.exe -c "ip addr | grep -Ee 'inet.*eth0'"
$found = $remoteport -match '\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}';

if( $found ){
  $remoteport = $matches[0];
} else{
  echo "The Script Exited, the ip address of WSL 2 cannot be found";
  exit;
}

#[Ports]

#All the ports you want to forward separated by coma
$ports=@(80,443,10000,3000,5000);


#[Static ip]
#You can change the addr to your ip config to listen to a specific address
$addr='0.0.0.0';
$ports_a = $ports -join ",";


#Remove Firewall Exception Rules
iex "Remove-NetFireWallRule -DisplayName 'WSL 2 Firewall Unlock' ";

#adding Exception Rules for inbound and outbound Rules
iex "New-NetFireWallRule -DisplayName 'WSL 2 Firewall Unlock' -Direction Outbound -LocalPort $ports_a -Action Allow -Protocol TCP";
iex "New-NetFireWallRule -DisplayName 'WSL 2 Firewall Unlock' -Direction Inbound -LocalPort $ports_a -Action Allow -Protocol TCP";

for( $i = 0; $i -lt $ports.length; $i++ ){
  $port = $ports[$i];
  iex "netsh interface portproxy delete v4tov4 listenport=$port listenaddress=$addr";
  iex "netsh interface portproxy add v4tov4 listenport=$port listenaddress=$addr connectport=$port connectaddress=$remoteport";
}
```
Then execute this file on PowerShell. You may have to change the ExecutionPolicy.

Every time the computer is rebooted, you will need to re-run this script. Or you can also use the Windows Task Scheduler to run this automatically after startup.

## Step 6: Access VNC remotely
Now you shoule be able to access the VNC session by <COMPUTE_NAME>:0. You can find your comeputer name by [This PC] > [Properties]. You can use the RealVNC on different platforms for this purpose.
