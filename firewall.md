## Install
```
sudo pacman -S firewalld
```
```
sudo systemctl enable --now firewalld
```
```
sudo systemctl status firewalld
```

## Check Zone
```
firewall-cmd --list-all-zones
```

## Check specific zone
example
```
firewall-cmd --info-zone=[home]
```

## Set default
example
```
firewall-cmd --set-default-zone=[home]
```

## Remove service
example
```
firewall-cmd --permanent --remove-[service]=[mdns]
```
check again
```
firewall-cmd --info-zone=[home]
```
use if 
```
firewall-cmd --reload
```

