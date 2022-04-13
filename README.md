# WSL2

## Issue : Network connectivity over corperate VPN
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
