# FlexNet as systemd service

FlexNet is a common license server from [Flexera Software](https://www.flexera.com) used with large number of software e.g. Abaqus. It is a server-client architecture license management software. The FlexNet server is designed to give remote access to licenses usually in a local network. The FlexNet server consists of the license manager daemon `lmgrd` and vendor(s) daemon(s). Both license manager and vendor daemons work with open TCP ports to communicate with clients - programs which check license on the server.

Usually the FlexNet installation procedure is combined with the installation of the software which uses FlexNet as license server. It's generally quite straightforward and well documented procedure. Unfortunately the default, post-installation configuration of the FlexNet server on Linux is very basic, inconvenient to administrate and last but not least it is not secure.

Please find below the instruction of configuration the FLexNet server installed with Abaqus ([DS SIMULIA](https://www.3ds.com/products-services/simulia/)) on the systemd-based Linux distribiutions (e.g. RHEL/CentOS 7) machine which:
- is run as a dedicated user and group,
- saves log files in specified directory (e.g. /var/log/),
- is started automatically at the system restart,
- can be started, stopped and reloaded using systemctl command.

The instruction can be easly adopted in the case on any other vendors.

The short instruction on how to configure FlexNet and firewall is provided as well.

## Create user & group

The FlexNet server should be run as a dedicated (non-root) user with restricted privileges (e.g. no login, limited access to files and directories). Moreover, on Linux usage of `lmdown`, `lmreread`, and `lmremove` commands is restricted to a license administrator who is by default root. Because we don't want to manage the FlexNet server using the root account the dedicated group called lmadmin is created. Thanks to this using `-2 -p` and `-local` option the usage of the server management commands is restricted to members of that group and from local machine only.

To create lmadmin group use command:
```
$ sudo groupadd lmadmin
```
To create flexnet user use command:
```
$ sudo useradd flexnet -d /opt/abaqus/License -c "FlexNet User" -g lmadmin -s /sbin/nologin
```
## Create configuration and log files

Placed log file in dedicated directory using command:
```
$ sudo touch /var/log/flexnet.log
```
Create configuration file to set up license and log files locations:
```
$ cat /etc/flexnet.conf
FLEXLICDIR=/opt/abaqus/License
FLEXLOGDIR=/var/log
FLEXLICFILE=abaquslm.lic
FLEXLOGFILE=flexnet.log
```
Set permissions:
```
$ sudo chown -R flexnet:lmadmin /usr/SIMULIA/License/2021/linux_a64/code/bin
$ sudo chown -R flexnet:lmadmin /opt/abaqus/License
$ sudo chown flexnet:lmadmin /var/log/flexnet.log
$ sudo chown flexnet:lmadmin /etc/flexnet.conf
```
Check permissions:
```
$ ls -al /etc/flexnet.conf
-rw-rw-r-- 1 flexnet lmadmin 206 01-05 01:27 /etc/flexnet.conf
…
```
## Creating and installing a systemd service unit

Create a systemd service unit file in any (e.g. home) directory:
```
$ cat flexnet.service
[Unit]
Description=FlexNet Licence Server
Requires=network.target
After=local_fs.target network.target

[Service]
EnvironmentFile=/etc/flexnet.conf
Type=simple
User=flexnet
Group=lmadmin
Restart=always
WorkingDirectory=/usr/SIMULIA/License/2022/linux_a64/code/bin
ExecStart=/usr/SIMULIA/License/2022/linux_a64/code/bin/lmgrd -c ${FLEXLICDIR}/${FLEXLICFILE} -l +${FLEXLOGDIR}/${FLEXLOGFILE} -z -2 -p -local
ExecReload=/usr/SIMULIA/License/2022/linux_a64/code/bin/lmreread -c ${FLEXLICDIR}/${FLEXLICFILE} -all
ExecStop=/usr/SIMULIA/License/2022/linux_a64/code/bin/lmdown -c ${FLEXLICDIR}/${FLEXLICFILE} -q
SuccessExitStatus=15

[Install]
WantedBy=multi-user.target
```
Please check and modify (if required)  the paths to director where FlexNet is installed.

To install the service - copy the service unit file into the /etc/systemd/system directory:
```
$ sudo cp flexnet.service /etc/systemd/system
$ sudo chmod 664 /etc/systemd/system/flexnet.service
```
Notify systemd that a new flexnet.service file exists: 
```
$ sudo systemctl daemon-reload
```
## Manage a FlexNet service

Enable the service to start it automatically at the next system restart:
```
$ sudo systemctl enable flexnet
```
To run the service use command:
```
$ sudo systemctl start flexnet
```
To check if the service is running you can use the following command:
```
$ sudo systemctl is-active flexnet
```
To check the service status (status of service, not license server) use command:
```
$ sudo systemctl status flexnet
```
To reload license file (e.g. to extend or updated license) first modify the license file and then run command:
```
$ sudo systemctl reload flexnet
```
To stop the service just use command:
```
$ sudo systemctl stop flexnet
```
## FlexNet & firewall configuration

The correctly configured and run firewall is a critical part of the server security for the most cases. The firewall configuration on the Linux server is not discussed here. However, if the firewall is activated its configuration has to be modified to enable access to the license server from other computers in the local network.

To see the status of the firewall service use the following command:
```
$ sudo firewall-cmd --state
running
```
If command output is running that means the firewall is run. Please follow the instructions below to configure the firewall to make the license server accessible in the local network. Please note, that open access to the license server from the public network is not recommended due to legal and security reasons.

First set up two ports for lmgrd and vendor demons in the license file:
```
$ cat /opt/abaqus/License/abaquslm.lic
SERVER this_host 6f9a7d0f33eb 27000
VENDOR ABAQUSLM PORT=53153
…
```
The port 27000 is default for `lmgrd` (server) daemon. By default vendor daemon port is assigned automatically and is changed on the occasion of each lmgrd restart. For firewall configuration vendor daemon port has to be set up permanently and can be any valid number of unused unprivileged ports in the range 1025 to 65535. To check if the selected port is free use command:
```
$ netstat -ltn | grep 53153
$
```
No output means the port is free and can be used for vendor daemon.

To open ports use the following commands:
```
$ sudo firewall-cmd [--zone=specific_zones] --add-port=27000/TCP
$ sudo firewall-cmd [--zone=specific_zones] --add-port=53153/TCP
```
The `--zone` parameter is optional and when it is omitted the default zone is modified. Please be careful with that and use the appropriate zone for the local network.

Check if license servers is available for expected remont machines and make the new settings persistent:
```
$ sudo firewall-cmd --runtime-to-permanent
```
