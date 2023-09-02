# DC1 - Debian AD DC Setup
Scripts and configuration files needed to set up an Active Directory Domain Controller on Debian.

Reference links:

* https://wiki.samba.org/index.php/Setting_up_Samba_as_an_Active_Directory_Domain_Controller
* https://wiki.samba.org/index.php/Idmap_config_ad
* https://github.com/christgau/wsdd
* https://wiki.samba.org/index.php/Setting_up_a_Share_Using_Windows_ACLs
* https://www.tecmint.com/manage-samba4-ad-from-windows-via-rsat/

Use these Network settings for all machines in VirtualBox:

* Adapter 1: Enabled
  * Attached to: NAT Network
  * Name: NatNetwork  (10.0.2.0/24 – DHCP & IPv6 disabled)
* Adapter 2: Enabled
  * Attached to: Host-only Adapter
  * Name: VirtualBox Host-Only Ethernet Adapter (192.168.56.0/24 – DHCP & IPv6 disabled)

Download the Debian netinstall image. Boot from it to begin the installation.

* Manually set the enp0s3 network interface:
  * address 10.0.2.5/24
  * gateway 10.0.2.1
  * nameserver 8.8.8.8
* Hostname: DC1
* Domain name: samdom.example.com
* Leave the root password blank.
* Enter the desired user name and password for the admin (sudo) account.
* Make your disk partition selections and write changes to disk.
* Software selection: Only “SSH server” and “standard system utilities”.
* Install the GRUB boot loader on /dev/sda
* Finish the installation and reboot.

Login as the admin user and switch to root.
Install git and download these instructions, scripts and configuration files:
```
apt update
apt dist-upgrade
apt install git
git clone https://github.com/TedMichalik/DC1.git
```
Copy config files to their proper location:
```
DC1/CopyFiles1
```
Add a static IP address for the second adapter.
A second adapter was enabled for SSH logins for configuration and testing in VirtualBox.
Make these changes to the **/etc/network/interfaces** file (Done with CopyFiles1):
```
# The primary network interface
auto enp0s3
iface enp0s3 inet static
        address 10.0.2.5/24
        gateway 10.0.2.1

# The secondary network interface
auto enp0s8
iface enp0s8 inet static
        address 192.168.56.5/24
```
Make these changes for resolving the local host name to the **/etc/hosts** file (Done with CopyFiles1):
```
127.0.0.1 localhost
10.0.2.5 DC1.samdom.example.com DC1
```
Change the default UMASK in the **/etc/login.defs** file (Done with CopyFiles1):
```
UMASK 002
```
Reboot the machine to switch to the static IP address.
SSH into the secondary adapter and login as the admin user and switch to root.

Install Samba and packages needed for an AD DC. Use the FQDN (DC1.samdom.example.com) for the servers in the Kerberos setup.
```
apt install samba attr winbind libpam-winbind libnss-winbind libpam-krb5 krb5-config krb5-user
```
Also install some utility programs:
```
apt install smbclient ldb-tools net-tools dnsutils chrony ntpdate isc-dhcp-server rsync
```
Stop and disable all Samba processes,  and remove the default smb.conf file:
```
systemctl stop smbd nmbd winbind
systemctl disable smbd nmbd winbind
mv /etc/samba/smb.conf /etc/samba/smb.conf.orig
```
Provision the Samba AD, giving your desired password for the Administrator:
```
samba-tool domain provision --use-rfc2307 --interactive
Realm=SAMDOM.EXAMPLE.COM
Domain=SAMDOM
Server Role=dc
DNS backend=SAMBA_INTERNAL
DNS forwarder IP address=8.8.8.8
```
Edit the Samba configuration file:
```
DC1/EditSMB
```
These lines are added by the EditSMB script to the [global] section of **/etc/samba/smb.conf**
```
interfaces = enp0s3
winbind nss info = rfc2307
winbind use default domain = yes
winbind offline logon = yes
winbind enum users = yes
winbind enum groups = yes
protocol = SMB3
usershare max shares = 0
```
Use the Samba created Kerberos configuration file for your DC, enable the correct Samba services, and reboot to make sure everything works:
```
cp /var/lib/samba/private/krb5.conf /etc/
systemctl unmask samba-ad-dc
systemctl start samba-ad-dc
systemctl enable samba-ad-dc
reboot
```
Login as the admin user and switch to root.
Verify the File Server shares provided by the DC:
```
smbclient -L localhost -U%
```
Copy more config files to their proper location. This also puts the RFC2307 script in cron.hourly to add uidNumber
and gidNumber to users, computers and groups added to AD. It runs the script and fixes the ownership of Group Policies.
```
DC1/CopyFiles2
```
Make these changes for resolving DNS names to the **/etc/resolv.conf** file (Done with CopyFiles2):
```
domain samdom.example.com
search samdom.example.com
nameserver 10.0.2.5
nameserver 8.8.8.8
```
Verify the DNS configuration works correctly:
```
host -t SRV _ldap._tcp.samdom.example.com.
host -t SRV _kerberos._udp.samdom.example.com.
host -t A dc1.samdom.example.com.
```
Verify Kerberos:
```
kinit administrator
klist
```
## Configure Chrony (Done with CopyFiles2)

Add these two lines in the **/etc/chrony/chrony.conf** file:
```
allow 0.0.0.0/0
ntpsigndsocket  /var/lib/samba/ntp_signd
```
Create the **ntp_signed** directory:
```
mkdir /var/lib/samba/ntp_signd/
chown root:_chrony /var/lib/samba/ntp_signd/
chmod 750 /var/lib/samba/ntp_signd/
```
Restart the Chrony service:
```
systemctl restart chronyd.service
```
## Check Chrony

Verify the Chrony service has open sockets:
```
netstat -tunlp | grep chrony
```
Verify the Chrony service is syncing with other servers:
```
chronyc sources
```
## Ease AD password restrictions for testing, if desired:
```
samba-tool domain passwordsettings set --complexity=off
samba-tool domain passwordsettings set --min-pwd-length=6
samba-tool domain passwordsettings set --max-pwd-age=0
samba-tool user setexpiry administrator --noexpiry
```
## Configure AD Accounts Authentication

Enable  entries required for winbind service to automatically create home directories for each domain account at the first login:
```
pam-auth-update
```
Give sudo access to members of “domain admins” (Done with CopyFiles2):
```
echo "%SAMDOM\\domain\ admins ALL=(ALL) ALL" > /etc/sudoers.d/SAMDOM
chmod 0440 /etc/sudoers.d/SAMDOM
```
## Configure the DHCP Service (Done with CopyFiles2):

Just use IPv4 on the NatNetwork with these edits to the /etc/default/isc-dhcp-server configuration file:
```
# On what interfaces should the DHCP server (dhcpd) serve DHCP requests?
# Separate multiple interfaces with spaces, e.g. "eth0 eth1".
INTERFACESv4="enp0s3"
#INTERFACESv6=""
```
Edit the /etc/dhcp/dhcpd.conf configuration file:
```
# dhcpd.conf
#
# Sample configuration file for ISC dhcpd
#
#
# option definitions common to all supported networks...
option domain-name "samdom.example.com";
option domain-name-servers 10.0.2.5;
#
default-lease-time 86400;
max-lease-time 604800;
#
# The ddns-updates-style parameter controls whether or not the server will
# attempt to do a DNS update when a lease is confirmed. We default to the
# behavior of the version 2 packages ('none', since DHCP v2 didn't
# have support for DDNS.)
ddns-update-style none;
#
# If this DHCP server is the official DHCP server for the local
# network, the authoritative directive should be uncommented.
authoritative;
#
# Use this to send dhcp log messages to a different log file (you also
# have to hack syslog.conf to complete the redirection).
#log-facility local7;
#
# No service will be given on this subnet, but declaring it helps the
# DHCP server to understand the network topology.
#
subnet 192.168.56.0 netmask 255.255.255.0 {
}
#
# This is a very basic subnet declaration.
#
subnet 10.0.2.0 netmask 255.255.255.0 {
range 10.0.2.50 10.0.2.100;
option routers 10.0.2.1;
}
```
Restart the service:
```
systemctl restart isc-dhcp-server.service
```
## Test the AD DC

Create an AD account for yourself and add it to the **Domain Admins** group with the commands:
```
samba-tool user create ted
/etc/cron.hourly/RFC2307
samba-tool group addmembers "Domain Admins" ted
```
Verify the domain users are shown by both commands:
```
wbinfo -u
getent passwd
```
Verify the domain groups are shown by both commands:
```
wbinfo -g
getent group
```
Verify the domain ownership on a test file:
```
touch /tmp/testfile
chown ted:"Domain Admins" /tmp/testfile
ls -l /tmp/testfile
```

## Join a Windows 10 Pro Desktop to the SAMDOM Domain

After joining the Windows desktop to the Domain, login with your **Domain Admins** account.

Go to **Settings | Apps & Features | Optional features** and make sure the following are installed:
* RSAT: Active Directory Domain Services and Lightweight Directory Services Tools
* RSAT: DNS Server Tools
* RSAT: Group Policy Management Tools

Run **Active Directory Users and Computers**:
* Make the **Domain Admins** group a member of the **Group Policy Creator Owners** group.
* Make the **Domain Computers** group a member of the **DnsAdmins** group.

Create a GPO  with the instructions at [This Link](https://wiki.samba.org/index.php/Time_Synchronisation#Configuring_Time_Synchronisation_on_a_Windows_Domain_Member)
