# WSL2

## Enabling WSL2
If you install docker desktop then it will ask you to install WSL2 libraries. Or you can google WSL installation. If you install WSL2 libraries then you might not get the ubuntu OS along with it. For that google Ubuntu WSL download and an Microsoft store link should appear. Download via MS Store & you will see a new Ubuntu dropdown in windows terminal.

# Install / Enable WSL
- Follow instructions at [Install WSL](https://docs.microsoft.com/en-us/windows/wsl/install)
- Installing WSL should install ubuntu distro by default. If not install one from [Microsoft Store](https://apps.microsoft.com/store/detail/ubuntu/9PDXGNCFSCZV?hl=en-us&gl=US)
- You can launch default distro using ```wsl``` command in cmd or any terminal.
- You will be asked to create a local user & password when launching ubuntu. 

# Network connectivity
## Understanding newtwork connectivity
- Disconnect kerry VPN and do a ping or telnet to google.com. You should be able to reach it.
- Connect your corperate VPN and do a ping to any kerry internal website & it should fail. (aka **Issue#1**)

## Solution 1 for Issue 1 : Network connectivity over cooperate VPN
When you open ubuntu wsl terminal and ping google it works when not connected to VPN but not when VPN connected. Follow below steps to get resolution.
Note: You will have to repeat step 1-3 everytime you reconnect vpn or reboot main machine
Note: Name resolution used to work when using this method (google.com) when connected to VPN but this no longer works on my machine. You can still use this method to connect to corperate sites but you will have to disconnect VPN on host machine to access external site. Please see Solution 2 for an alternative.

1. Set your VPN adapter (if you have Cisco AnyConnect) open a admin powershell

- Find out your VPN adapter name: Get-NetIPInterface or  Get-NetAdapter (in my case: "Cisco AnyConnect")
- Set adapter metric (Replace -Match with your name), in my case I have to run this after ever reboot or VPN reconnect:
- Interface metric is used to determine route, windows use interface with lowest metric

```Get-NetAdapter | Where-Object {$_.InterfaceDescription -Match "Cisco AnyConnect"} | Set-NetIPInterface -InterfaceMetric 6000```

2. Find out nameserver with windows powershell (during VPN Session). Copy the Address part.
```nslookup```

3. Add your corperate nameserver to resolv.conf

```bash
# Removes existing resolv conf
sudo rm /etc/resolv.conf

# If rm throws an error then mark the file as mutable
sudo chattr -a -i /etc/resolv.conf

# writes nameserver line to resolv.conf. 8.8.8.8 is google dns, you can either try that or the corperate dns address (nslookup).
sudo bash -c 'echo "nameserver 8.8.8.8" > /etc/resolv.conf'
```

4. Disable resolv conf generation in wsl config.
```bash
sudo bash -c 'echo "[network]" > /etc/wsl.conf'
sudo bash -c 'echo "generateResolvConf = false" >> /etc/wsl.conf'
sudo chattr +i /etc/resolv.conf
```

5. Restart wsl in powershell:
```bash
wsl.exe --shutdown
```
6. Re open ubuntu & ping any corperate intranet site using name or ipaddress & it should work. Other sites like google should also be reachable. 

## Solution 2 for Issue 1
Above solution worked for about a week after which I was having name resolution issues. 

After a day of trial and error what was found was if VPN dns addresses are at top of the resolve config then kerry websites work but other websites wont (Ex: google.com but 8.8.8.8 works), & when Wifi dns address was added at the top google.com was working but not kerry website. 

I've written the below script to semi automate the priority switch. When you need to build something with external site dependencies like docker store then user "script.ps1 -p wifi" else when corporate network is needed use  "script.ps1 -p vpn". There are two parts to this 
1. powershell script will read the windows adapter configs & will generate a file with those details 
2. Bash script which will copy & replace that file from windows dir. Make sure you change the output ($new_resolv_conf_path) & input dirs as needed.

**Benefits over solution 1**
- No need to disconnect VPN when accessing external domain sites.
- No need to fetch cisco & wifi dns addresses
- resolv.conf is updated semi automatically

```powershell
param (
    [string]$p = 'vpn'
)

# Set cisco adapter to highest priority
Get-NetAdapter | Where-Object {$_.InterfaceDescription -Match "Cisco AnyConnect"} | Set-NetIPInterface -InterfaceMetric 6000

# Get Cisco IPV4 addresses (for kerry communication ex: ipo.amer.kerry.com)
$vpn_addresses = (Get-NetAdapter | Where-Object InterfaceDescription -like "*Cisco*" | Get-DnsClientServerAddress).ServerAddresses


# Get Wifi IPV4 addresses (for non kerry communication ex: google.com)
$non_vpn_addresses += (Get-NetAdapter | Where-Object InterfaceDescription -like "*Wi-Fi*" | Get-DnsClientServerAddress).ServerAddresses


# Prepare resolv.conf contents
$resolve_conf_contents = ''
$OFS = "`n"
$addresses = New-Object System.Collections.Generic.List[System.Object]

If ($p -eq 'vpn') {

    foreach ($address in $vpn_addresses) {
        $addresses.Add($address);
    }
    
    foreach ($address in $non_vpn_addresses) {
        $addresses.Add($address);
    }

} 
Else {
    
    foreach ($address in $non_vpn_addresses) {
        $addresses.Add($address);
    }

    foreach ($address in $vpn_addresses) {
        $addresses.Add($address);
    }
}


foreach ($address in $addresses) {
 $resolve_conf_contents += ('nameserver ' + $address + $OFS)
}

# Note do not exceed the name server list line above 240 chars. To get that list use 'ipconfig /all' and look for DNS Suffix Search List at the top.
$resolve_conf_contents += 'search your_corporate_nameserver1 your_corporate_nameserver2 your_corporate_nameserver3'
$resolve_conf_contents = $resolve_conf_contents.Trim();

$new_resolv_conf_path = "D:\_Documents\WSL\resolv.conf";

Write-Host('Generated file output');

# Verify file & output to file
if (-not(Test-Path -Path $new_resolv_conf_path -PathType Leaf)) {
     try {
         $null = New-Item -ItemType File -Path $new_resolv_conf_path -Force -ErrorAction Stop
         Write-Host('New resolv.conf file create at ' + $new_resolv_conf_path)
     }
     catch {
         throw $_.Exception.Message
     }
 }
 else {
    Write-Host('Clearing current resolv.conf contents');
    Clear-Content $new_resolv_conf_path
 }

Write-Host('Writing to resolv.conf');
[System.IO.File]::WriteAllText($new_resolv_conf_path, $resolve_conf_contents)

Write-Host('Done');
```


### Bash script

Create a bash script (ex: script.sh) in wsl distro & paste the below code. Then make it executable using 
```chmod u+x script.sh```. Run this script using 'sudo .\script.sh'.
```bash
#!/bin/bash

sudo chattr -a -i /etc/resolv.conf

sudo rm /etc/resolv.conf

sudo cp /mnt/d/_Documents/WSL/resolv.conf /etc/resolv.conf

sudo chattr +i /etc/resolv.conf
```




## If issues happen on main windows after above workaround
In my case I get DNS issues when try to connect to internal stuff via browser (on Windows 10, f.e.: intranet), caused by the high metric value set in step 4 (basically kind of disabling VPN Route). So here is the workaround for the workaround:

1. Check your default metric (of VPNs Interface) in powershell (replace -Match with your interface name)
```Get-NetAdapter | Where-Object {$_.InterfaceDescription -Match "Cisco AnyConnect"} | Get-NetIPInterface```
2. When running into problems on Windows 10 restore this default value with admin powershell (replace value at the end with your default value):
```Get-NetAdapter | Where-Object {$_.InterfaceDescription -Match "Cisco AnyConnect"} | Set-NetIPInterface -InterfaceMetric 1```



## Issue 2 : Network connectivity over corperate VPN over HTTPS

if you try you might be facing a SSL cert problem issue. 
```curl -fsSL https://download.docker.com/linux/ubuntu/gpg```
You can also mitigate this temporarily by adding -J option which makes insure connections. But it is not a long term solution & is insecure. This is not a curl issue as curl request is being handled b the OS itself. To fix it follow below steps.

Windows drivers can be accessed in WSL2 by navigating to /mnt/c where c is the drive letter. 

**Export root ca cert from host OS** 
1. Go to windows crendential manager (Win + R -> credmgr.msc)
2. Trusted Root Certification -> Ceritificates -> ZScaler -> Open -> Details -> Copy to file
3. Select format as DER encoded
4. Save it as zscaler_root.cer

**Copy to WSL2 ubuntu**
1. open your wsl2 ubuntu instance
2. sudo cp /mnt/c/Users/your_name/Downloads/zscaler_root.cer ~/any_folder_or_path

**Convert cer to crt format**
1. sudo openssl x509 -inform DER -in ~/path_to_your_cer_file/zscaler_root.cer -out ~/path_to_your_crt_file/zscaler_root.crt
2. You can also extract it as base64 file then you will use (untested). Notice PEM vs DER.
3. openssl x509 -inform PEM -in <filepath>/certificate.cert -out certificate.crt



**Copy to user level ca cert folder**
1. sudo cp ~/path_to_your_crt_file/zscaler_root.crt /usr/local/share/ca-certificates/
2. sudo chmod 644 /usr/local/share/ca-certificates/zscaler_root.crt

**Update ca cert store to include new crt file**
1. sudo update-ca-certificates
2. You should see 1 added. 
3. Try out the same curl command & you should not see the SSL error.
  
  
 ## Set communication btw windows app & docker in WSL ubuntu
 ```
 https://github.com/frcs6/DockerWSL-WindowsHost-Tutorial
 ```
  
  
## Useful commands & things to know

 **Copy file && Move**
  ```
  cp target_file_path destination_file_path
  mv target_file_path destination_file_path
  ```
 **Create folder**
 ```mkdir dir_name``` 

 **Remove**
  ```
     rm target_file_path
     rmdir your_directory_path (if dir not empty this will complain use sudo rm -rf your_dir)
  ```
  
   **Remove all files within a dir**
  /* is very important here.
  ```
     rm -r target_dir_path/*
  ```
  
  **Rename or Move**
  Instead of rename we can use move command to rename certs folder to _certs folder. This does not require -r flag.
  ```
  mv certs _certs
  ```
  
  **Display list of files**
  ```
  ls (shows only non hidden files)
  ls -al (for hidden files)
  ```
  
  **Search command history**
  ```
  history | gerp your_search_keyword
  ex: history | getp cert - Then all previous commands which you used containing the cert keyword will be shown
  ```
  
  **Windows directories**
  ```/mnt/c/ - If where your c drive is. The alphabet is the driver.```
  
  **show text file contents**
  cat your_textfile
  
  **WSL global config**
  ```
  You can configure the settings for your installed Linux distributions that will automatically be applied every time you launch WSL in two ways, by using:
  .wslconfig to configure settings globally across all installed distributions running on WSL 2. This is located at %userprofile%
  wsl.conf to configure settings per-distribution for Linux distros running on WSL 1 or WSL 2. This is located within ubuntu at /etc/wsl.conf
  ```
  
  **ubuntu service status**
  Plus sign indicates running & - indicates stopped. 
  ```
  service --status-all
  sudo service docker start
  ```
  
  **ubuntu temp dirs**
  Both /tmp and /var/tmp are used by programs as well as the system itself to store data temporarily. By default, all the files and data that gets stored in /var/tmp live for up to 30 days. Whereas in /tmp, the data gets automatically deleted after ten days.
  ``` 
  /var/tmp & /tmp
  ```
  
  **Search**
  ``` 
  find / -type f -name "your_filename" (case sensitive)
  find / -type f -iname "filename*" (case insensitive)
  remove -type to search for directories) 
  ```
  
  **Reverse Search**
  ```
  ctrl + r will reverse search your previous commands in wondows & ubuntu. Only upto 1000 lines
  ```
  
  **Search inline**
  ```
  ipconfig.exe /all | grep -i "DNS Server"
  Pipe grep is used to search the results returned from ipconfig.exe /all. You can use pipe grep on other commands too. If you want also want to include next lines then add -A1. Where 1 is the lines you want. Ex: -A20 will get you next 20 lines.
  
  ipconfig.exe /all | grep -i -A1 "DNS Server"
  ```
  
  **No sudo password prompt**
  ````
  sudo nano /etc/sudoers
  find %admin ALL=(ALL) ALL or something like that. Below that paste your username. % symbol is a must.
  %YOUR_USER_NAME ALL=(ALL) NOPASSWD: ALL
  Save & close. Note: You will still need to use sudo but there wont be a password prompt.
  ````
 
  **Shells**
 ````
   Therea are multiple shells available for linux. BASH is the default shell. Others are ZShell (ZSH), CornShell(KSH), CShell(CSH). These are usually found at /bin folder (binary folder). You also have your cp, mkdir, mv default OS level binaries under /bin folder. You can also find bash & zsh & others under /bin. User level binaries are found at /user/local/bin folder.
 ````
    
  **launch docker container interactive shell / accessing php logs in docker container**
  -ti is interactive sh terminal (/bin/sh)  
  ```
   docker exec -ti container_name /bin/sh
    
    ls
    sudo 777 -R ./writable (have to do this on first docker build as the php api has to write to this folder logs)
    cd ./writable/logs
    cat your_log_file_name.log
    
  ```
