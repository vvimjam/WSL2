# WSL2

## Issue 1 : Network connectivity over corperate VPN
When you open ubuntu wsl terminal and ping google it works when not connected to VPN but not when VPN connected. Follow below steps to get resolution.
Note: You will have to repeat step 4 everytime you reconnect vpn or reboot main machine

1. Find out nameserver with windows powershell (during VPN Session). Copy the Address part.
```nslookup```

2. Add your corperate nameserver to resolv.conf

```bash
# Removes existing resolv conf
sudo rm /etc/resolv.conf

# writes nameserver line to resolv.conf. 8.8.8.8 is google dns, you can either try that or the corperate dns address (nslookup).
sudo bash -c 'echo "nameserver 8.8.8.8" > /etc/resolv.conf'
```

3. Disable resolv conf generation in wsl config.
```bash
sudo bash -c 'echo "[network]" > /etc/wsl.conf'
sudo bash -c 'echo "generateResolvConf = false" >> /etc/wsl.conf'
sudo chattr +i /etc/resolv.conf
```

4. Set your VPN adapter (if you have Cisco AnyConnect) open a admin powershell

- Find out your VPN adapter name: Get-NetIPInterface or  Get-NetAdapter (in my case: "Cisco AnyConnect")
- Set adapter metric (Replace -Match with your name), in my case I have to run this after ever reboot or VPN reconnect:
- Interface metric is used to determine route, windows use interface with lowest metric

```Get-NetAdapter | Where-Object {$_.InterfaceDescription -Match "Cisco AnyConnect"} | Set-NetIPInterface -InterfaceMetric 6000```

5. Restart wsl in powershell: wsl.exe --shutdown

6. Test it in wsl run: ping google.com - if this command works, you are done.


## If issues happen on main windows after above workaround
In my case I get DNS issues when try to connect to internal stuff via browser (on Windows 10, f.e.: intranet), caused by the high metric value set in step 4 (basically kind of disabling VPN Route). So here is the workaround for the workaround:

1. Check your default metric (of VPNs Interface) in powershell (replace -Match with your interface name)
```Get-NetAdapter | Where-Object {$_.InterfaceDescription -Match "Cisco AnyConnect"} | Get-NetIPInterface```
2. When running into problems on Windows 10 restore this default value with admin powershell (replace value at the end with your default value):
```Get-NetAdapter | Where-Object {$_.InterfaceDescription -Match "Cisco AnyConnect"} | Set-NetIPInterface -InterfaceMetric 1```



## Issue 2 : Network connectivity over corperate VPN over HTTPS

if you try you might be facing a SSL cert problem issue. This is not a curl issue as curl request is being handled b the OS itself. To fix it follow below steps.
```curl -fsSL https://download.docker.com/linux/ubuntu/gpg```

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
  
  
## Useful commands

 **Copy file && Move*
  cp target_file_path destination_file_path
  mv target_file_path destination_file_path

 **Remove*
  rm target_file_path
  
  ** display list of files**
  ls (shows only non hidden files)
  ls -al (for hidden files)
  
  ** Search command history**
  history | gerp your_search_keyword
  ex: history | getp cert - Then all previous commands which you used containing the cert keyword will be shown
  
  ** Windows directories**
  /mnt/c/ - If where your c drive is. The alphabet is the driver.
  
