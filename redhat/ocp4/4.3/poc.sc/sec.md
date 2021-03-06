# poc for sc

## rhel host maintain

### aliyun host

```bash

ssh-copy-id root@

cat << EOF > /root/.ssh/config
StrictHostKeyChecking no
UserKnownHostsFile=/dev/null

EOF

export VULTR_HOST=helper.hsc.redhat.ren

rsync -e ssh --info=progress2 -P --delete -arz /data/rhel-data/data ${VULTR_HOST}:/data/rhel-data

rsync -e ssh --info=progress2 -P --delete -arz /data/registry ${VULTR_HOST}:/data/

rsync -e ssh --info=progress2 -P --delete -arz /data/ocp4 ${VULTR_HOST}:/data/

rsync -e ssh --info=progress2 -P --delete -arz /data/is.samples ${VULTR_HOST}:/data/

cd /data
tar -cvf - registry/ | pigz -c > registry.tgz
tar -cvf - ocp4/ | pigz -c > ocp4.tgz
tar -cvf - data/ | pigz -c > rhel-data.tgz
tar -cvf - is.samples/ | pigz -c > /data_hdd/down/is.samples.tgz

```

### helper host

```bash
######################################################
# on helper

find . -name vsftp*
yum -y install ./data/rhel-7-server-rpms/Packages/vsftpd-3.0.2-25.el7.x86_64.rpm
systemctl start vsftpd
systemctl restart vsftpd
systemctl enable vsftpd

firewall-cmd --permanent --add-service=ftp
firewall-cmd --reload

mv data /var/ftp/
chcon -R -t public_content_t /var/ftp/data

mkdir /etc/yum.repos.d.bak
mv /etc/yum.repos.d/* /etc/yum.repos.d.bak

cat << EOF > /etc/yum.repos.d/remote.repo
[remote]
name=RHEL FTP
baseurl=ftp://117.177.241.16/data
enabled=1
gpgcheck=0

EOF

yum clean all
yum repolist

yum -y update

hostnamectl set-hostname helper.hsc.redhat.ren
nmcli connection modify em1 ipv4.dns 114.114.114.114
nmcli connection reload
nmcli connection up em1

yum -y install fail2ban

cat << EOF > /etc/fail2ban/jail.d/wzh.conf
[sshd]
enabled = true

[recidive]
enabled = true

EOF

systemctl enable fail2ban
systemctl restart fail2ban

fail2ban-client status sshd
fail2ban-client status recidive
systemctl status fail2ban
tail -F /var/log/fail2ban.log

cp /etc/ssh/sshd_config /etc/ssh/sshd_config.BAK
sed -i 's/#UseDNS yes/UseDNS no/g' /etc/ssh/sshd_config

diff /etc/ssh/sshd_config /etc/ssh/sshd_config.BAK

systemctl restart sshd

passwd

useradd -m wzh

lsblk | grep 446 | awk '{print $1}' | xargs -I DEMO echo -n "/dev/DEMO "
# /dev/sdb /dev/sdc /dev/sdd /dev/sde /dev/sdf /dev/sdg /dev/sdh /dev/sdi /dev/sdj /dev/sdk /dev/sdl /dev/sdm
lsblk | grep 446 | awk '{print $1}' | wc -l
# 12

# https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_logical_volumes/assembly_configure-mange-raid-configuring-and-managing-logical-volumes
yum install -y lvm2

pvcreate -y /dev/sdb /dev/sdc /dev/sdd /dev/sde /dev/sdf /dev/sdg /dev/sdh /dev/sdi /dev/sdj /dev/sdk /dev/sdl /dev/sdm

vgcreate datavg /dev/sdb /dev/sdc /dev/sdd /dev/sde /dev/sdf /dev/sdg /dev/sdh /dev/sdi /dev/sdj /dev/sdk /dev/sdl /dev/sdm

vgs

lvcreate --type raid10 -l 100%FREE --stripes 6 -n datalv datavg

umount /data_hdd
lvremove /dev/datavg/datalv

mkfs.xfs /dev/datavg/datalv

lvdisplay /dev/datavg/datalv -m

mkdir -p /data

cp /etc/fstab /etc/fstab.bak

cat << EOF >> /etc/fstab
/dev/datavg/datalv /data                   xfs     defaults        0 0

EOF

mount -a

yum install -y sysstat
lsblk | grep disk | awk '{print $1}' | xargs -I DEMO echo -n "DEMO "
# sda sdb sdc sdd sde sdf sdg sdh sdi sdj sdk sdl sdm
iostat -h -m -x sda sdb sdc sdd sde sdf sdg sdh sdi sdj sdk sdl sdm 5
iostat -m -x dm-24 5

yum install -y chrony
systemctl enable chronyd
systemctl restart chronyd
systemctl status chronyd
chronyc tracking

firewall-cmd --get-zones
# block dmz drop external home internal public trusted work
firewall-cmd --zone=public --list-all

firewall-cmd --permanent --zone=public --remove-port=2049/tcp

firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" port port="2049" protocol="tcp" source address="117.177.241.0/24" accept'

firewall-cmd --reload

showmount -a
exportfs -s

cd /data_ssd/
scp *.tgz root@117.177.241.17:/data_hdd/down/

# https://access.redhat.com/solutions/3341191
# subscription-manager register --org=ORG ID --activationkey= Key Name
cat /var/log/rhsm/rhsm.log

subscription-manager config --rhsm.manage_repos=0
cp /etc/yum/pluginconf.d/subscription-manager.conf /etc/yum/pluginconf.d/subscription-manager.conf.orig
cat << EOF  > /etc/yum/pluginconf.d/subscription-manager.conf
[main]
enabled=0
EOF

# https://access.redhat.com/products/red-hat-insights/#getstarted
subscription-manager register --auto-attach
yum --disableplugin=subscription-manager install insights-client
insights-client --register

yum --disableplugin=subscription-manager install ncdu


```

### bootstrap host

```bash
######################################################
# bootstrap

mkdir /etc/yum.repos.d.bak
mv /etc/yum.repos.d/* /etc/yum.repos.d.bak

cat << EOF > /etc/yum.repos.d/remote.repo
[remote]
name=RHEL FTP
baseurl=ftp://117.177.241.16/data
enabled=1
gpgcheck=0

EOF

yum clean all
yum repolist

yum -y update

hostnamectl set-hostname bootstrap.hsc.redhat.ren

nmcli connection modify em1 ipv4.dns 117.177.241.16
nmcli connection reload
nmcli connection up em1

yum -y install fail2ban

cat << EOF > /etc/fail2ban/jail.d/wzh.conf
[sshd]
enabled = true

[recidive]
enabled = true

EOF

systemctl enable fail2ban
systemctl restart fail2ban

fail2ban-client status sshd
fail2ban-client status recidive
systemctl status fail2ban
tail -F /var/log/fail2ban.log

cp /etc/ssh/sshd_config /etc/ssh/sshd_config.BAK
sed -i 's/#UseDNS yes/UseDNS no/g' /etc/ssh/sshd_config

diff /etc/ssh/sshd_config /etc/ssh/sshd_config.BAK

systemctl restart sshd

passwd

useradd -m wzh

lsblk | grep 446 | awk '{print $1}' | xargs -I DEMO echo -n "/dev/DEMO "
# /dev/sdb /dev/sdc /dev/sdd /dev/sde /dev/sdf /dev/sdg /dev/sdh /dev/sdi /dev/sdj /dev/sdk /dev/sdl /dev/sdm
lsblk | grep 446 | awk '{print $1}' | wc -l
# 12

# https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_logical_volumes/assembly_configure-mange-raid-configuring-and-managing-logical-volumes
yum install -y lvm2

pvcreate -y /dev/sdb /dev/sdc /dev/sdd /dev/sde /dev/sdf /dev/sdg /dev/sdh /dev/sdi /dev/sdj /dev/sdk /dev/sdl /dev/sdm

vgcreate datavg /dev/sdb /dev/sdc /dev/sdd /dev/sde /dev/sdf /dev/sdg /dev/sdh /dev/sdi /dev/sdj /dev/sdk /dev/sdl /dev/sdm

vgs

lvcreate --type raid10 -l 100%FREE --stripes 6 -n datalv datavg

mkfs.xfs /dev/datavg/datalv

lvdisplay /dev/datavg/datalv -m

mkdir -p /data

cp /etc/fstab /etc/fstab.bak

cat << EOF >> /etc/fstab
/dev/datavg/datalv /data                   xfs     defaults        0 0

EOF

mount -a

yum install -y sysstat
lsblk | grep disk | awk '{print $1}' | xargs -I DEMO echo -n "DEMO "
# sda sdb sdc sdd sde sdf sdg sdh sdi sdj sdk sdl sdm
iostat -h -m -x sda sdb sdc sdd sde sdf sdg sdh sdi sdj sdk sdl sdm 5
iostat -m -x dm-24 5

yum install -y chrony
systemctl enable chronyd
systemctl restart chronyd
systemctl status chronyd
chronyc tracking

```

### master0 host

```bash
#####################################################
# master0

mkdir /etc/yum.repos.d.bak
mv /etc/yum.repos.d/* /etc/yum.repos.d.bak

cat << EOF > /etc/yum.repos.d/remote.repo
[remote]
name=RHEL FTP
baseurl=ftp://117.177.241.16/data
enabled=1
gpgcheck=0

EOF

yum clean all
yum repolist

yum -y update

hostnamectl set-hostname master0.hsc.redhat.ren

nmcli connection modify em1 ipv4.dns 117.177.241.16
nmcli connection reload
nmcli connection up em1

yum -y install fail2ban

cat << EOF > /etc/fail2ban/jail.d/wzh.conf
[sshd]
enabled = true

EOF

systemctl enable fail2ban
systemctl restart fail2ban

fail2ban-client status sshd
fail2ban-client status recidive
systemctl status fail2ban
tail -F /var/log/fail2ban.log

cat << EOF > /etc/fail2ban/jail.d/wzh.conf
[sshd]
enabled = true

[recidive]
enabled = true

EOF

cp /etc/ssh/sshd_config /etc/ssh/sshd_config.BAK
sed -i 's/#UseDNS yes/UseDNS no/g' /etc/ssh/sshd_config

diff /etc/ssh/sshd_config /etc/ssh/sshd_config.BAK

systemctl restart sshd

passwd

useradd -m wzh

yum install -y chrony
systemctl enable chronyd
systemctl restart chronyd
systemctl status chronyd
chronyc tracking

lsblk | grep 446 | awk '{print $1}' | xargs -I DEMO echo -n "/dev/DEMO "
# /dev/sdb /dev/sdc /dev/sdd /dev/sde /dev/sdf /dev/sdg /dev/sdh /dev/sdi /dev/sdj /dev/sdk /dev/sdl /dev/sdm
lsblk | grep 446 | awk '{print $1}' | wc -l
# 12

# https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_logical_volumes/assembly_configure-mange-raid-configuring-and-managing-logical-volumes
yum install -y lvm2

pvcreate -y /dev/sdb /dev/sdc /dev/sdd /dev/sde /dev/sdf /dev/sdg /dev/sdh /dev/sdi /dev/sdj /dev/sdk /dev/sdl /dev/sdm

vgcreate datavg /dev/sdb /dev/sdc /dev/sdd /dev/sde /dev/sdf /dev/sdg /dev/sdh /dev/sdi /dev/sdj /dev/sdk /dev/sdl /dev/sdm

vgs

lvcreate --type raid0 -l 100%FREE --stripes 12 -n datalv datavg

mkfs.xfs /dev/datavg/datalv

lvdisplay /dev/datavg/datalv -m

mkdir -p /data
mkdir -p /data_hdd

cp /etc/fstab /etc/fstab.bak

cat << EOF >> /etc/fstab
/dev/datavg/datalv /data_hdd                  xfs     defaults        0 0

EOF

mount -a
```

### master1 host

```bash
######################################################
# master1

mkdir /etc/yum.repos.d.bak
mv /etc/yum.repos.d/* /etc/yum.repos.d.bak

cat << EOF > /etc/yum.repos.d/remote.repo
[remote]
name=RHEL FTP
baseurl=ftp://117.177.241.16/data
enabled=1
gpgcheck=0

EOF

yum clean all
yum repolist

yum -y update

hostnamectl set-hostname master1.hsc.redhat.ren

nmcli connection modify em1 ipv4.dns 117.177.241.16
nmcli connection reload
nmcli connection up em1

yum -y install fail2ban

cat << EOF > /etc/fail2ban/jail.d/wzh.conf
[sshd]
enabled = true

[recidive]
enabled = true

EOF

systemctl enable fail2ban
systemctl restart fail2ban

fail2ban-client status sshd
fail2ban-client status recidive
systemctl status fail2ban
tail -F /var/log/fail2ban.log

cp /etc/ssh/sshd_config /etc/ssh/sshd_config.BAK
sed -i 's/#UseDNS yes/UseDNS no/g' /etc/ssh/sshd_config

diff /etc/ssh/sshd_config /etc/ssh/sshd_config.BAK

systemctl restart sshd

passwd

useradd -m wzh

yum install -y chrony
systemctl enable chronyd
systemctl restart chronyd
systemctl status chronyd
chronyc tracking

mkdir -p /data_hdd
mkfs.xfs -f /dev/sdb

cat << EOF >> /etc/fstab
/dev/sdb /data_hdd                   xfs     defaults        0 0
EOF

mount -a
```

### master2 host

```bash
######################################################
# master2

mkdir /etc/yum.repos.d.bak
mv /etc/yum.repos.d/* /etc/yum.repos.d.bak

cat << EOF > /etc/yum.repos.d/remote.repo
[remote]
name=RHEL FTP
baseurl=ftp://117.177.241.16/data
enabled=1
gpgcheck=0

EOF

yum clean all
yum repolist

yum -y update

hostnamectl set-hostname master2.hsc.redhat.ren

nmcli connection modify em1 ipv4.dns 117.177.241.16
nmcli connection reload
nmcli connection up em1

yum -y install fail2ban

cat << EOF > /etc/fail2ban/jail.d/wzh.conf
[sshd]
enabled = true
EOF

systemctl enable fail2ban
systemctl restart fail2ban

cat << EOF > /etc/fail2ban/jail.d/wzh.conf
[sshd]
enabled = true

[recidive]
enabled = true

EOF

fail2ban-client status
systemctl status fail2ban
tail -F /var/log/fail2ban.log

cp /etc/ssh/sshd_config /etc/ssh/sshd_config.BAK
sed -i 's/#UseDNS yes/UseDNS no/g' /etc/ssh/sshd_config

diff /etc/ssh/sshd_config /etc/ssh/sshd_config.BAK

systemctl restart sshd

passwd

useradd -m wzh

lsblk | grep 446 | awk '{print $1}' | xargs -I DEMO echo -n "/dev/DEMO "
# /dev/sdb /dev/sdc /dev/sdd /dev/sde /dev/sdf /dev/sdg /dev/sdh /dev/sdi /dev/sdj /dev/sdk /dev/sdl /dev/sdm
lsblk | grep 446 | awk '{print $1}' | wc -l
# 12

# https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_logical_volumes/assembly_configure-mange-raid-configuring-and-managing-logical-volumes
yum install -y lvm2

pvcreate -y /dev/sdb /dev/sdc /dev/sdd /dev/sde /dev/sdf /dev/sdg /dev/sdh /dev/sdi /dev/sdj /dev/sdk /dev/sdl /dev/sdm

vgcreate datavg /dev/sdb /dev/sdc /dev/sdd /dev/sde /dev/sdf /dev/sdg /dev/sdh /dev/sdi /dev/sdj /dev/sdk /dev/sdl /dev/sdm

vgs

lvcreate --type raid0 -l 100%FREE --stripes 12 -n datalv datavg

mkfs.xfs /dev/datavg/datalv

lvdisplay /dev/datavg/datalv -m

mkdir -p /data
mkdir -p /data_hdd

cp /etc/fstab /etc/fstab.bak

cat << EOF >> /etc/fstab
/dev/datavg/datalv /data_hdd                   xfs     defaults        0 0

EOF

mount -a

yum install -y sysstat
lsblk | grep disk | awk '{print $1}' | xargs -I DEMO echo -n "DEMO "
# sda sdb sdc sdd sde sdf sdg sdh sdi sdj sdk sdl sdm
iostat -m -x sda sdb sdc sdd sde sdf sdg sdh sdi sdj sdk sdl sdm 5
iostat -m -x dm-12 5

yum install -y chrony
systemctl enable chronyd
systemctl restart chronyd
systemctl status chronyd
chronyc tracking
```

### infra0 host

```bash
######################################################
# infra0

mkdir /etc/yum.repos.d.bak
mv /etc/yum.repos.d/* /etc/yum.repos.d.bak

cat << EOF > /etc/yum.repos.d/remote.repo
[remote]
name=RHEL FTP
baseurl=ftp://117.177.241.16/data
enabled=1
gpgcheck=0

EOF

yum clean all
yum repolist

yum -y update

hostnamectl set-hostname infra0.hsc.redhat.ren

nmcli connection modify em1 ipv4.dns 117.177.241.16
nmcli connection reload
nmcli connection up em1

yum -y install fail2ban

cat << EOF > /etc/fail2ban/jail.d/wzh.conf
[sshd]
enabled = true

[recidive]
enabled = true

EOF

systemctl enable fail2ban
systemctl restart fail2ban

fail2ban-client status sshd
fail2ban-client status recidive
systemctl status fail2ban
tail -F /var/log/fail2ban.log

cp /etc/ssh/sshd_config /etc/ssh/sshd_config.BAK
sed -i 's/#UseDNS yes/UseDNS no/g' /etc/ssh/sshd_config

diff /etc/ssh/sshd_config /etc/ssh/sshd_config.BAK

systemctl restart sshd

passwd

useradd -m wzh

lsblk | grep 446 | awk '{print $1}' | xargs -I DEMO echo -n "/dev/DEMO "
# /dev/sdb /dev/sdc /dev/sdd /dev/sde /dev/sdf /dev/sdg /dev/sdh /dev/sdi /dev/sdj /dev/sdk /dev/sdl /dev/sdm
lsblk | grep 446 | awk '{print $1}' | wc -l
# 12

# https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_logical_volumes/assembly_configure-mange-raid-configuring-and-managing-logical-volumes
yum install -y lvm2

pvcreate -y /dev/sdb /dev/sdc /dev/sdd /dev/sde /dev/sdf /dev/sdg /dev/sdh /dev/sdi /dev/sdj /dev/sdk /dev/sdl /dev/sdm

vgcreate datavg /dev/sdb /dev/sdc /dev/sdd /dev/sde /dev/sdf /dev/sdg /dev/sdh /dev/sdi /dev/sdj /dev/sdk /dev/sdl /dev/sdm

vgs

lvcreate --type raid0 -l 100%FREE --stripes 12 -n datalv datavg

mkfs.xfs /dev/datavg/datalv

lvdisplay /dev/datavg/datalv -m

mkdir -p /data
mkdir -p /data_hdd

cp /etc/fstab /etc/fstab.bak

cat << EOF >> /etc/fstab
/dev/datavg/datalv /data_hdd                   xfs     defaults        0 0

EOF

mount -a

yum install -y sysstat
lsblk | grep disk | awk '{print $1}' | xargs -I DEMO echo -n "DEMO "
# sda sdb sdc sdd sde sdf sdg sdh sdi sdj sdk sdl sdm
iostat -m -x sda sdb sdc sdd sde sdf sdg sdh sdi sdj sdk sdl sdm 5
iostat -m -x dm-12 5

yum install -y chrony
systemctl enable chronyd
systemctl restart chronyd
systemctl status chronyd
chronyc tracking
```

### infra1 host

```bash
######################################################
# infra1

mkdir /etc/yum.repos.d.bak
mv /etc/yum.repos.d/* /etc/yum.repos.d.bak

cat << EOF > /etc/yum.repos.d/remote.repo
[remote]
name=RHEL FTP
baseurl=ftp://117.177.241.16/data
enabled=1
gpgcheck=0

EOF

yum clean all
yum repolist

yum -y update

hostnamectl set-hostname infra1.hsc.redhat.ren

nmcli connection modify em1 ipv4.dns 117.177.241.16
nmcli connection reload
nmcli connection up em1

yum -y install fail2ban

cat << EOF > /etc/fail2ban/jail.d/wzh.conf
[sshd]
enabled = true

[recidive]
enabled = true

EOF

systemctl enable fail2ban
systemctl restart fail2ban

fail2ban-client status sshd
fail2ban-client status recidive
systemctl status fail2ban
tail -F /var/log/fail2ban.log

cp /etc/ssh/sshd_config /etc/ssh/sshd_config.BAK
sed -i 's/#UseDNS yes/UseDNS no/g' /etc/ssh/sshd_config

diff /etc/ssh/sshd_config /etc/ssh/sshd_config.BAK

systemctl restart sshd

passwd

useradd -m wzh

lsblk | grep 446 | awk '{print $1}' | xargs -I DEMO echo -n "/dev/DEMO "
# /dev/sdb /dev/sdc /dev/sdd /dev/sde /dev/sdf /dev/sdg /dev/sdh /dev/sdi /dev/sdj /dev/sdk /dev/sdl /dev/sdm
lsblk | grep 446 | awk '{print $1}' | wc -l
# 12

# https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_logical_volumes/assembly_configure-mange-raid-configuring-and-managing-logical-volumes
yum install -y lvm2

pvcreate -y /dev/sdb /dev/sdc /dev/sdd /dev/sde /dev/sdf /dev/sdg /dev/sdh /dev/sdi /dev/sdj /dev/sdk /dev/sdl /dev/sdm

vgcreate datavg /dev/sdb /dev/sdc /dev/sdd /dev/sde /dev/sdf /dev/sdg /dev/sdh /dev/sdi /dev/sdj /dev/sdk /dev/sdl /dev/sdm

vgs

lvcreate --type raid0 -l 100%FREE --stripes 12 -n datalv datavg

mkfs.xfs /dev/datavg/datalv

lvdisplay /dev/datavg/datalv -m

mkdir -p /data
mkdir -p /data_hdd

cp /etc/fstab /etc/fstab.bak

cat << EOF >> /etc/fstab
/dev/datavg/datalv /data_hdd                   xfs     defaults        0 0

EOF

mount -a

yum install -y sysstat
lsblk | grep disk | awk '{print $1}' | xargs -I DEMO echo -n "DEMO "
# sda sdb sdc sdd sde sdf sdg sdh sdi sdj sdk sdl sdm
iostat -m -x sda sdb sdc sdd sde sdf sdg sdh sdi sdj sdk sdl sdm 5
iostat -m -x dm-12 5

yum install -y chrony
systemctl enable chronyd
systemctl restart chronyd
systemctl status chronyd
chronyc tracking

```

### worker-0 host

```bash

mkdir /etc/yum.repos.d.bak
mv /etc/yum.repos.d/* /etc/yum.repos.d.bak

cat << EOF > /etc/yum.repos.d/remote.repo
[remote]
name=RHEL FTP
baseurl=ftp://117.177.241.16/data
enabled=1
gpgcheck=0

EOF

yum clean all
yum --disableplugin=subscription-manager  repolist

yum -y update

hostnamectl set-hostname worker-0.ocpsc.redhat.ren

nmcli connection modify em1 ipv4.dns 117.177.241.16
nmcli connection reload
nmcli connection up em1

yum -y install fail2ban

cat << EOF > /etc/fail2ban/jail.d/wzh.conf
[sshd]
enabled = true

EOF

systemctl enable fail2ban
systemctl restart fail2ban

cat << EOF > /etc/fail2ban/jail.d/wzh.conf
[sshd]
enabled = true

[recidive]
enabled = true

EOF

fail2ban-client status sshd
fail2ban-client status recidive
systemctl status fail2ban
tail -F /var/log/fail2ban.log

cp /etc/ssh/sshd_config /etc/ssh/sshd_config.BAK
sed -i 's/#UseDNS yes/UseDNS no/g' /etc/ssh/sshd_config

diff /etc/ssh/sshd_config /etc/ssh/sshd_config.BAK

systemctl restart sshd

passwd

useradd -m wzh

```
## install ocp

### helper node

```bash
############################################################
# on macbook
mkdir -p /Users/wzh/Documents/redhat/tools/redhat.ren/etc
mkdir -p /Users/wzh/Documents/redhat/tools/redhat.ren/lib
mkdir -p /Users/wzh/Documents/redhat/tools/ocpsc.redhat.ren/etc
mkdir -p /Users/wzh/Documents/redhat/tools/ocpsc.redhat.ren/lib
mkdir -p /Users/wzh/Documents/redhat/tools/apps.ocpsc.redhat.ren/etc
mkdir -p /Users/wzh/Documents/redhat/tools/apps.ocpsc.redhat.ren/lib

cd /Users/wzh/Documents/redhat/tools/redhat.ren/
docker run -it --rm --name certbot \
            -v "/Users/wzh/Documents/redhat/tools/redhat.ren/etc:/etc/letsencrypt" \
            -v "/Users/wzh/Documents/redhat/tools/redhat.ren/lib:/var/lib/letsencrypt" \
            certbot/certbot certonly  -d "*.redhat.ren" --manual --preferred-challenges dns-01  --server https://acme-v02.api.letsencrypt.org/directory

cp ./etc/archive/redhat.ren/fullchain4.pem redhat.ren.crt
cp ./etc/archive/redhat.ren/privkey4.pem redhat.ren.key

cd /Users/wzh/Documents/redhat/tools/ocpsc.redhat.ren/
docker run -it --rm --name certbot \
            -v "/Users/wzh/Documents/redhat/tools/ocpsc.redhat.ren/etc:/etc/letsencrypt" \
            -v "/Users/wzh/Documents/redhat/tools/ocpsc.redhat.ren/lib:/var/lib/letsencrypt" \
            certbot/certbot certonly  -d "*.ocpsc.redhat.ren" --manual --preferred-challenges dns-01  --server https://acme-v02.api.letsencrypt.org/directory

cp ./etc/archive/ocpsc.redhat.ren/fullchain1.pem ocpsc.redhat.ren.crt
cp ./etc/archive/ocpsc.redhat.ren/privkey1.pem ocpsc.redhat.ren.key


cd /Users/wzh/Documents/redhat/tools/apps.ocpsc.redhat.ren/
docker run -it --rm --name certbot \
            -v "/Users/wzh/Documents/redhat/tools/apps.ocpsc.redhat.ren/etc:/etc/letsencrypt" \
            -v "/Users/wzh/Documents/redhat/tools/apps.ocpsc.redhat.ren/lib:/var/lib/letsencrypt" \
            certbot/certbot certonly  -d "*.apps.ocpsc.redhat.ren" --manual --preferred-challenges dns-01  --server https://acme-v02.api.letsencrypt.org/directory

cp ./etc/archive/apps.ocpsc.redhat.ren/fullchain1.pem apps.ocpsc.redhat.ren.crt
cp ./etc/archive/apps.ocpsc.redhat.ren/privkey1.pem apps.ocpsc.redhat.ren.key

# scp these keys to helper
# /data/cert/*

####################################################
# on helper node
yum -y install podman docker-distribution pigz skopeo httpd-tools

# https://access.redhat.com/solutions/3175391
htpasswd -cbB /etc/docker-distribution/registry_passwd admin ***************

cat << EOF > /etc/docker-distribution/registry/config.yml
version: 0.1
log:
  fields:
    service: registry
storage:
    cache:
        layerinfo: inmemory
    filesystem:
        rootdirectory: /data/registry
    delete:
        enabled: true
http:
    addr: :5443
    tls:
       certificate: /data/cert/redhat.ren.crt
       key: /data/cert/redhat.ren.key
auth:
  htpasswd:
    realm: basic‑realm
    path: /etc/docker-distribution/registry_passwd
EOF
# systemctl restart docker
systemctl stop docker-distribution
systemctl enable docker-distribution
systemctl restart docker-distribution
# 

firewall-cmd --permanent --add-port=5443/tcp
firewall-cmd --reload

podman login registry.redhat.ren:5443 -u admin -p *******************

yum install -y docker
systemctl start docker
docker login registry.redhat.ren:5443 -u admin

# upload vars-static.yaml to helper
yum -y install ansible-2.8.10 git unzip podman python36

cd /data/ocp4/ocp4-upi-helpernode
ansible-playbook -e @vars-static.yaml -e staticips=true tasks/main.yml

# upload install-config.yaml to helper /data/ocp4
cd /data/ocp4

/bin/rm -rf *.ign .openshift_install_state.json auth bootstrap master0 master1 master2 worker0 worker1 worker2

openshift-install create ignition-configs --dir=/data/ocp4

/bin/cp -f bootstrap.ign /var/www/html/ignition/bootstrap-static.ign
/bin/cp -f master.ign /var/www/html/ignition/master-0.ign
/bin/cp -f master.ign /var/www/html/ignition/master-1.ign
/bin/cp -f master.ign /var/www/html/ignition/master-2.ign
/bin/cp -f worker.ign /var/www/html/ignition/worker-0.ign
/bin/cp -f worker.ign /var/www/html/ignition/worker-1.ign
/bin/cp -f worker.ign /var/www/html/ignition/worker-2.ign

chmod 644 /var/www/html/ignition/*

########################################################
# on helper node, create iso
yum -y install genisoimage libguestfs-tools
systemctl start libvirtd

export NGINX_DIRECTORY=/data/ocp4
export RHCOSVERSION=4.3.0
export VOLID=$(isoinfo -d -i ${NGINX_DIRECTORY}/rhcos-${RHCOSVERSION}-x86_64-installer.iso | awk '/Volume id/ { print $3 }')
TEMPDIR=$(mktemp -d)
echo $VOLID
echo $TEMPDIR

cd ${TEMPDIR}
# Extract the ISO content using guestfish (to avoid sudo mount)
guestfish -a ${NGINX_DIRECTORY}/rhcos-${RHCOSVERSION}-x86_64-installer.iso \
  -m /dev/sda tar-out / - | tar xvf -

# Helper function to modify the config files
modify_cfg(){
  for file in "EFI/redhat/grub.cfg" "isolinux/isolinux.cfg"; do
    # Append the proper image and ignition urls
    sed -e '/coreos.inst=yes/s|$| coreos.inst.install_dev=vda coreos.inst.image_url='"${URL}"'\/install\/'"${BIOSMODE}"'.raw.gz coreos.inst.ignition_url='"${URL}"'\/ignition\/'"${NODE}"'.ign ip='"${IP}"'::'"${GATEWAY}"':'"${NETMASK}"':'"${FQDN}"':'"${NET_INTERFACE}"':none:'"${DNS}"' nameserver='"${DNS}"'|' ${file} > $(pwd)/${NODE}_${file##*/}
    # Boot directly in the installation
    sed -i -e 's/default vesamenu.c32/default linux/g' -e 's/timeout 600/timeout 10/g' $(pwd)/${NODE}_${file##*/}
  done
}

URL="http://117.177.241.16:8080/"
GATEWAY="117.177.241.1"
NETMASK="255.255.255.0"
DNS="117.177.241.16"

# BOOTSTRAP
# TYPE="bootstrap"
NODE="bootstrap-static"
IP="117.177.241.243"
FQDN="vm-bootstrap"
BIOSMODE="bios"
NET_INTERFACE="ens3"
modify_cfg

# MASTERS
# TYPE="master"
# MASTER-0
NODE="master-0"
IP="117.177.241.240"
FQDN="vm-master0"
BIOSMODE="bios"
NET_INTERFACE="ens3"
modify_cfg

# MASTER-1
NODE="master-1"
IP="117.177.241.241"
FQDN="vm-master1"
BIOSMODE="bios"
NET_INTERFACE="ens3"
modify_cfg

# MASTER-2
NODE="master-2"
IP="117.177.241.242"
FQDN="vm-master2"
BIOSMODE="bios"
NET_INTERFACE="ens3"
modify_cfg

# WORKERS
NODE="worker-0"
IP="117.177.241.244"
FQDN="vm-worker0"
BIOSMODE="bios"
NET_INTERFACE="ens3"
modify_cfg

NODE="worker-1"
IP="117.177.241.245"
FQDN="vm-worker1"
BIOSMODE="bios"
NET_INTERFACE="ens3"
modify_cfg


# Generate the images, one per node as the IP configuration is different...
# https://github.com/coreos/coreos-assembler/blob/master/src/cmd-buildextend-installer#L97-L103
for node in master-0 master-1 master-2 worker-0 worker-1 worker-2 bootstrap-static; do
  # Overwrite the grub.cfg and isolinux.cfg files for each node type
  for file in "EFI/redhat/grub.cfg" "isolinux/isolinux.cfg"; do
    /bin/cp -f $(pwd)/${node}_${file##*/} ${file}
  done
  # As regular user!
  genisoimage -verbose -rock -J -joliet-long -volset ${VOLID} \
    -eltorito-boot isolinux/isolinux.bin -eltorito-catalog isolinux/boot.cat \
    -no-emul-boot -boot-load-size 4 -boot-info-table \
    -eltorito-alt-boot -efi-boot images/efiboot.img -no-emul-boot \
    -o ${NGINX_DIRECTORY}/${node}.iso .
done

# Optionally, clean up
cd /data/ocp4
rm -Rf ${TEMPDIR}

cd ${NGINX_DIRECTORY}

# mkdir -p /data/ocp4
# mkdir -p /data/kvm
scp master-*.iso root@117.177.241.17:/data/ocp4/

scp master-*.iso root@117.177.241.21:/data/ocp4/
scp worker-*.iso root@117.177.241.21:/data/ocp4/
scp bootstrap-*.iso root@117.177.241.21:/data/ocp4/

scp master-*.iso root@117.177.241.18:/data/ocp4/

# after you create and boot master vm, worker vm, you can track the result
export KUBECONFIG=/data/ocp4/auth/kubeconfig
echo "export KUBECONFIG=/data/ocp4/auth/kubeconfig" >> ~/.bashrc
source ~/.bashrc
oc get nodes

openshift-install wait-for bootstrap-complete --log-level debug

oc get csr

openshift-install wait-for install-complete

bash add.image.load.sh /data_ssd/is.samples/mirror_dir/

oc apply -f ./99-worker-zzz-container-registries.yaml -n openshift-config
oc apply -f ./99-master-zzz-container-registries.yaml -n openshift-config

```

### helper node day1 oper

```bash

# https://docs.openshift.com/container-platform/4.3/openshift_images/managing_images/using-image-pull-secrets.html#images-update-global-pull-secret_using-image-pull-secrets
oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=/data/pull-secret.json

# https://docs.openshift.com/container-platform/4.3/networking/ingress-operator.html#nw-ingress-controller-tls-profiles_configuring-ingress
oc --namespace openshift-ingress-operator get ingresscontrollers

oc --namespace openshift-ingress create secret tls custom-certs-default --cert=/data/cert/apps.ocpsc.redhat.ren.crt --key=/data/cert/apps.ocpsc.redhat.ren.key

oc patch --type=merge --namespace openshift-ingress-operator ingresscontrollers/default \
  --patch '{"spec":{"defaultCertificate":{"name":"custom-certs-default"}}}'

oc get --namespace openshift-ingress-operator ingresscontrollers/default \
  --output jsonpath='{.spec.defaultCertificate}'

##################################################3
# add rhel hw node, and remove vm worker node
ssh-copy-id root@infra-0.ocpsc.redhat.ren
ssh root@infra-0.ocpsc.redhat.ren

ssh-copy-id root@infra-1.ocpsc.redhat.ren
ssh root@infra-1.ocpsc.redhat.ren

# disable firewalld on infra-0, infra-1

yum -y install openshift-ansible openshift-clients jq

# create rhel-ansible-host
cat <<EOF > /data/ocp4/rhel-ansible-host
[all:vars]
ansible_user=root 
#ansible_become=True 

openshift_kubeconfig_path="/data/ocp4/auth/kubeconfig" 

[new_workers] 
infra-0.ocpsc.redhat.ren
infra-1.ocpsc.redhat.ren

EOF

ansible-playbook -i /data/ocp4/rhel-ansible-host /usr/share/ansible/openshift-ansible/playbooks/scaleup.yml

# then remove old vm-worker0, vm-worker1
oc get nodes -o wide
oc adm cordon vm-worker-0.ocpsc.redhat.ren
oc adm cordon vm-worker-1.ocpsc.redhat.ren
oc adm drain vm-worker-0.ocpsc.redhat.ren --force --delete-local-data --ignore-daemonsets
oc adm drain vm-worker-1.ocpsc.redhat.ren --force --delete-local-data --ignore-daemonsets  
oc delete nodes vm-worker-0.ocpsc.redhat.ren
oc delete nodes vm-worker-1.ocpsc.redhat.ren
oc get nodes -o wide

# create nfs storage and enable image operator
bash ocp4-upi-helpernode/files/nfs-provisioner-setup.sh

oc patch configs.imageregistry.operator.openshift.io cluster -p '{"spec":{"managementState": "Managed","storage":{"pvc":{"claim":""}}}}' --type=merge

# create operator catalog
oc patch OperatorHub cluster --type json \
    -p '[{"op": "add", "path": "/spec/disableAllDefaultSources", "value": true}]'

cat <<EOF > redhat-operator-catalog.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: redhat-operator-catalog
  namespace: openshift-marketplace
spec:
  displayName: Redhat Operator Catalog
  sourceType: grpc
  image: registry.redhat.ren:5443/docker.io/wangzheng422/operator-catalog:redhat-2020-03-23
  publisher: Red Hat
EOF
oc create -f redhat-operator-catalog.yaml

# create infra node
# https://access.redhat.com/solutions/4287111
oc get node

oc label node infra0.hsc.redhat.ren node-role.kubernetes.io/infra=""
oc label node infra1.hsc.redhat.ren node-role.kubernetes.io/infra=""

oc patch ingresscontroller default -n openshift-ingress-operator --type=merge --patch='{"spec":{"nodePlacement":{"nodeSelector": {"matchLabels":{"node-role.kubernetes.io/infra":""}}}}}'

oc patch configs.imageregistry.operator.openshift.io/cluster -n openshift-image-registry --type=merge --patch '{"spec":{"nodeSelector":{"node-role.kubernetes.io/infra":""}}}'

oc get pod -o wide -n openshift-image-registry --sort-by=".spec.nodeName"

cat <<EOF > /data/ocp4/monitoring-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |+
    alertmanagerMain:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    prometheusK8s:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    prometheusOperator:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    grafana:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    k8sPrometheusAdapter:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    kubeStateMetrics:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    telemeterClient:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
EOF

oc create -f /data/ocp4/monitoring-cm.yaml -n openshift-monitoring

oc get pods -n openshift-monitoring -o wide --sort-by=".spec.nodeName"

###########################################
## add user for zte
cd /data/ocp4
touch /data/ocp4/htpasswd
htpasswd -B /data/ocp4/htpasswd zteca
htpasswd -B /data/ocp4/htpasswd zteadm

oc create secret generic htpasswd --from-file=/data/ocp4/htpasswd -n openshift-config

oc apply -f - <<EOF
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: Local Password
    mappingMethod: claim
    type: HTPasswd
    htpasswd:
      fileData:
        name: htpasswd
EOF

watch oc get pod -n openshift-authentication

oc adm policy add-cluster-role-to-user cluster-admin  zteca

oc new-project zte
oc adm policy add-role-to-user admin zteadm -n zte

oc get clusterrolebinding.rbac

oc get clusterrole.rbac

oc adm policy add-cluster-role-to-user cluster-reader  zteadm
oc adm policy remove-cluster-role-from-user cluster-reader  zteadm

```

### helper node quay
```bash
# on helper node
firewall-cmd --permanent --zone=public --add-port=4443/tcp
firewall-cmd --reload

podman pod create --infra-image registry.redhat.ren:5443/gcr.io/google_containers/pause-amd64:3.0 --name quay -p 4443:8443 

cd /data
rm -rf /data/quay
podman run -d --name quay-fs --entrypoint "tail" registry.redhat.ren:5443/docker.io/wangzheng422/quay-fs:3.2.0-init -f /dev/null
podman cp quay-fs:/quay.tgz /data/
tar zxf quay.tgz
podman rm -fv quay-fs

export MYSQL_CONTAINER_NAME=quay-mysql
export MYSQL_DATABASE=enterpriseregistrydb
export MYSQL_PASSWORD=zvbk3fzp5f5m2a8j
export MYSQL_USER=quayuser
export MYSQL_ROOT_PASSWORD=q98u335musckfqxe

podman run \
    --detach \
    --restart=always \
    --env MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD} \
    --env MYSQL_USER=${MYSQL_USER} \
    --env MYSQL_PASSWORD=${MYSQL_PASSWORD} \
    --env MYSQL_DATABASE=${MYSQL_DATABASE} \
    --name ${MYSQL_CONTAINER_NAME} \
    --privileged=true \
    --pod quay \
    -v /data/quay/lib/mysql:/var/lib/mysql/data:Z \
    registry.redhat.ren:5443/registry.access.redhat.com/rhscl/mysql-57-rhel7

podman run -d --restart=always \
    --pod quay \
    --privileged=true \
    --name quay-redis \
    -v  /data/quay/lib/redis:/var/lib/redis/data:Z \
    registry.redhat.ren:5443/registry.access.redhat.com/rhscl/redis-32-rhel7

sleep 10

/bin/cp -f /data/cert/redhat.ren.crt /data/quay/config/extra_ca_certs/redhat.ren.crt
/bin/cp -f /data/cert/redhat.ren.crt /data/quay/config/ssl.cert
/bin/cp -f /data/cert/redhat.ren.key /data/quay/config/ssl.key

podman run --restart=always \
    --sysctl net.core.somaxconn=4096 \
    --privileged=true \
    --name quay-master \
    --pod quay \
    --add-host mysql:127.0.0.1 \
    --add-host redis:127.0.0.1 \
    --add-host clair:127.0.0.1 \
    -v /data/quay/config:/conf/stack:Z \
    -v /data/quay/storage:/datastorage:Z \
    -d registry.redhat.ren:5443/quay.io/redhat/quay:v3.2.1

# https://registry.redhat.ren:4443/

podman run --name clair-postgres --pod quay \
    -v /data/quay/lib/postgresql/data:/var/lib/postgresql/data:Z \
    -d registry.redhat.ren:5443/docker.io/library/postgres

podman run --restart=always -d \
    --name clair \
    -v /data/quay/clair-config:/clair/config \
    -v /data/quay/clair-config/ca.crt:/etc/pki/ca-trust/source/anchors/ca.crt  \
    --pod quay \
    --add-host clair:127.0.0.1 \
    registry.redhat.ren:5443/quay.io/redhat/clair-jwt:v3.2.1

```

### bootstrap node

```bash
##########################################################3
## on bootstrap
yum -y install tigervnc-server tigervnc gnome-terminal gnome-session gnome-classic-session gnome-terminal nautilus-open-terminal control-center liberation-mono-fonts google-noto-sans-cjk-fonts google-noto-sans-fonts fonts-tweak-tool

yum install -y    qgnomeplatform   xdg-desktop-portal-gtk   NetworkManager-libreswan-gnome   PackageKit-command-not-found   PackageKit-gtk3-module   abrt-desktop   at-spi2-atk   at-spi2-core   avahi   baobab   caribou   caribou-gtk2-module   caribou-gtk3-module   cheese   compat-cheese314   control-center   dconf   empathy   eog   evince   evince-nautilus   file-roller   file-roller-nautilus   firewall-config   firstboot   fprintd-pam   gdm   gedit   glib-networking   gnome-bluetooth   gnome-boxes   gnome-calculator   gnome-classic-session   gnome-clocks   gnome-color-manager   gnome-contacts   gnome-dictionary   gnome-disk-utility   gnome-font-viewer   gnome-getting-started-docs   gnome-icon-theme   gnome-icon-theme-extras   gnome-icon-theme-symbolic   gnome-initial-setup   gnome-packagekit   gnome-packagekit-updater   gnome-screenshot   gnome-session   gnome-session-xsession   gnome-settings-daemon   gnome-shell   gnome-software   gnome-system-log   gnome-system-monitor   gnome-terminal   gnome-terminal-nautilus   gnome-themes-standard   gnome-tweak-tool   nm-connection-editor   orca   redhat-access-gui   sane-backends-drivers-scanners   seahorse   setroubleshoot   sushi   totem   totem-nautilus   vinagre   vino   xdg-user-dirs-gtk   yelp

yum install -y    cjkuni-uming-fonts   dejavu-sans-fonts   dejavu-sans-mono-fonts   dejavu-serif-fonts   gnu-free-mono-fonts   gnu-free-sans-fonts   gnu-free-serif-fonts   google-crosextra-caladea-fonts   google-crosextra-carlito-fonts   google-noto-emoji-fonts   jomolhari-fonts   khmeros-base-fonts   liberation-mono-fonts   liberation-sans-fonts   liberation-serif-fonts   lklug-fonts   lohit-assamese-fonts   lohit-bengali-fonts   lohit-devanagari-fonts   lohit-gujarati-fonts   lohit-kannada-fonts   lohit-malayalam-fonts   lohit-marathi-fonts   lohit-nepali-fonts   lohit-oriya-fonts   lohit-punjabi-fonts   lohit-tamil-fonts   lohit-telugu-fonts   madan-fonts   nhn-nanum-gothic-fonts   open-sans-fonts   overpass-fonts   paktype-naskh-basic-fonts   paratype-pt-sans-fonts   sil-abyssinica-fonts   sil-nuosu-fonts   sil-padauk-fonts   smc-meera-fonts   stix-fonts   thai-scalable-waree-fonts   ucs-miscfixed-fonts   vlgothic-fonts   wqy-microhei-fonts   wqy-zenhei-fonts

vncpasswd

cat << EOF > ~/.vnc/xstartup
#!/bin/sh
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
gnome-session &
EOF
chmod +x ~/.vnc/xstartup

vncserver :1 -geometry 1280x800
# 如果你想停掉vnc server，这么做
vncserver -kill :1

firewall-cmd --permanent --add-port=6001/tcp
firewall-cmd --permanent --add-port=5901/tcp
firewall-cmd --reload

# 配置kvm环境
yum -y install qemu-kvm libvirt libvirt-python libguestfs-tools virt-install virt-viewer virt-manager

systemctl enable libvirtd
systemctl start libvirtd

brctl show
virsh net-list

cat << EOF >  /data/virt-net.xml
<network>
  <name>br0</name>
  <forward mode='bridge'>
    <bridge name='br0'/>
  </forward>
</network>
EOF

virsh net-define --file virt-net.xml
virsh net-dumpxml br0
# virsh net-undefine openshift4
# virsh net-destroy openshift4
virsh net-autostart br0
virsh net-start br0

cp /etc/sysconfig/network-scripts/ifcfg-em1 /etc/sysconfig/network-scripts/ifcfg-em1.orig

cat << EOF > /etc/sysconfig/network-scripts/ifcfg-em1
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=em1
DEVICE=em1
ONBOOT=yes
# IPADDR=117.177.241.21
# PREFIX=24
# GATEWAY=117.177.241.1
IPV6_PRIVACY=no
# DNS1=117.177.241.16
BRIDGE=br0
EOF

cat <<EOF > /etc/sysconfig/network-scripts/ifcfg-br0 
TYPE=Bridge
BOOTPROTO=static
IPADDR=117.177.241.21
GATEWAY=117.177.241.1
DNS1=117.177.241.16
ONBOOT=yes
DEFROUTE=yes
NAME=br0
DEVICE=br0
PREFIX=24
EOF

systemctl restart network

virt-install --name=ocp4-bootstrap --vcpus=2 --ram=16384 \
--disk path=/data/kvm/ocp4-bootstrap.qcow2,bus=virtio,size=200 \
--os-variant rhel8.0 --network bridge=br0,model=virtio \
--boot menu=on --cdrom /data/ocp4/bootstrap-static.iso   

virt-install --name=ocp4-master0 --vcpus=8 --ram=65536 \
--disk path=/data/kvm/ocp4-master0.qcow2,bus=virtio,size=200 \
--os-variant rhel8.0 --network bridge=br0,model=virtio \
--boot menu=on --cdrom /data/ocp4/master-0.iso 

# virt-install --name=ocp4-master1 --vcpus=20 --ram=200704 \
# --disk path=/data/kvm/ocp4-master1.qcow2,bus=virtio,size=200 \
# --os-variant rhel8.0 --network bridge=br0,model=virtio \
# --boot menu=on --cdrom /data/ocp4/master-1.iso 

virt-install --name=ocp4-master2 --vcpus=8 --ram=65536 \
--disk path=/data/kvm/ocp4-master2.qcow2,bus=virtio,size=200 \
--os-variant rhel8.0 --network bridge=br0,model=virtio \
--boot menu=on --cdrom /data/ocp4/master-2.iso 

virt-install --name=ocp4-worker0 --vcpus=4 --ram=32768 \
--disk path=/data/kvm/ocp4-worker0.qcow2,bus=virtio,size=200 \
--os-variant rhel8.0 --network bridge=br0,model=virtio \
--boot menu=on --cdrom /data/ocp4/worker-0.iso 

virt-install --name=ocp4-worker1 --vcpus=4 --ram=32768 \
--disk path=/data/kvm/ocp4-worker1.qcow2,bus=virtio,size=200 \
--os-variant rhel8.0 --network bridge=br0,model=virtio \
--boot menu=on --cdrom /data/ocp4/worker-1.iso 


tar -cvf - ocp4-master0.qcow2 | pigz -c > /data/kvm/ocp4-master0.qcow2.tgz
rsync -e "ssh -c chacha20-poly1305@openssh.com" --info=progress2 -P -arz  /data/kvm/ocp4-master0.qcow2.tgz root@117.177.241.18:/data/kvm/

tar -cvf - ocp4-master2.qcow2 | pigz -c > /data/kvm/ocp4-master2.qcow2.tgz
rsync -e "ssh -c chacha20-poly1305@openssh.com" --info=progress2 -P -arz  /data/kvm/ocp4-master2.qcow2.tgz root@117.177.241.22:/data/kvm/

```

### master1 node

```bash
##########################################################3
## on master1
yum -y install tigervnc-server tigervnc gnome-terminal gnome-session gnome-classic-session gnome-terminal nautilus-open-terminal control-center liberation-mono-fonts google-noto-sans-cjk-fonts google-noto-sans-fonts fonts-tweak-tool

yum install -y    qgnomeplatform   xdg-desktop-portal-gtk   NetworkManager-libreswan-gnome   PackageKit-command-not-found   PackageKit-gtk3-module   abrt-desktop   at-spi2-atk   at-spi2-core   avahi   baobab   caribou   caribou-gtk2-module   caribou-gtk3-module   cheese   compat-cheese314   control-center   dconf   empathy   eog   evince   evince-nautilus   file-roller   file-roller-nautilus   firewall-config   firstboot   fprintd-pam   gdm   gedit   glib-networking   gnome-bluetooth   gnome-boxes   gnome-calculator   gnome-classic-session   gnome-clocks   gnome-color-manager   gnome-contacts   gnome-dictionary   gnome-disk-utility   gnome-font-viewer   gnome-getting-started-docs   gnome-icon-theme   gnome-icon-theme-extras   gnome-icon-theme-symbolic   gnome-initial-setup   gnome-packagekit   gnome-packagekit-updater   gnome-screenshot   gnome-session   gnome-session-xsession   gnome-settings-daemon   gnome-shell   gnome-software   gnome-system-log   gnome-system-monitor   gnome-terminal   gnome-terminal-nautilus   gnome-themes-standard   gnome-tweak-tool   nm-connection-editor   orca   redhat-access-gui   sane-backends-drivers-scanners   seahorse   setroubleshoot   sushi   totem   totem-nautilus   vinagre   vino   xdg-user-dirs-gtk   yelp

yum install -y    cjkuni-uming-fonts   dejavu-sans-fonts   dejavu-sans-mono-fonts   dejavu-serif-fonts   gnu-free-mono-fonts   gnu-free-sans-fonts   gnu-free-serif-fonts   google-crosextra-caladea-fonts   google-crosextra-carlito-fonts   google-noto-emoji-fonts   jomolhari-fonts   khmeros-base-fonts   liberation-mono-fonts   liberation-sans-fonts   liberation-serif-fonts   lklug-fonts   lohit-assamese-fonts   lohit-bengali-fonts   lohit-devanagari-fonts   lohit-gujarati-fonts   lohit-kannada-fonts   lohit-malayalam-fonts   lohit-marathi-fonts   lohit-nepali-fonts   lohit-oriya-fonts   lohit-punjabi-fonts   lohit-tamil-fonts   lohit-telugu-fonts   madan-fonts   nhn-nanum-gothic-fonts   open-sans-fonts   overpass-fonts   paktype-naskh-basic-fonts   paratype-pt-sans-fonts   sil-abyssinica-fonts   sil-nuosu-fonts   sil-padauk-fonts   smc-meera-fonts   stix-fonts   thai-scalable-waree-fonts   ucs-miscfixed-fonts   vlgothic-fonts   wqy-microhei-fonts   wqy-zenhei-fonts

vncpasswd

cat << EOF > ~/.vnc/xstartup
#!/bin/sh
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
gnome-session &
EOF
chmod +x ~/.vnc/xstartup

vncserver :1 -geometry 1280x800
# 如果你想停掉vnc server，这么做
vncserver -kill :1

firewall-cmd --permanent --add-port=6001/tcp
firewall-cmd --permanent --add-port=5901/tcp
firewall-cmd --reload

# 配置kvm环境
yum -y install qemu-kvm libvirt libvirt-python libguestfs-tools virt-install virt-viewer virt-manager

systemctl enable libvirtd
systemctl start libvirtd

brctl show
virsh net-list

cat << EOF >  /data/virt-net.xml
<network>
  <name>br0</name>
  <forward mode='bridge'>
    <bridge name='br0'/>
  </forward>
</network>
EOF

virsh net-define --file virt-net.xml
virsh net-dumpxml br0
# virsh net-undefine openshift4
# virsh net-destroy openshift4
virsh net-autostart br0
virsh net-start br0

cp /etc/sysconfig/network-scripts/ifcfg-em1 /etc/sysconfig/network-scripts/ifcfg-em1.orig

cat << EOF > /etc/sysconfig/network-scripts/ifcfg-em1
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=em1
DEVICE=em1
ONBOOT=yes
# IPADDR=117.177.241.17
# PREFIX=24
# GATEWAY=117.177.241.1
IPV6_PRIVACY=no
# DNS1=117.177.241.16
BRIDGE=br0
EOF

cat <<EOF > /etc/sysconfig/network-scripts/ifcfg-br0 
TYPE=Bridge
BOOTPROTO=static
IPADDR=117.177.241.17
GATEWAY=117.177.241.1
DNS1=117.177.241.16
ONBOOT=yes
DEFROUTE=yes
NAME=br0
DEVICE=br0
PREFIX=24
EOF

systemctl restart network

virt-install --name=ocp4-master1 --vcpus=20 --ram=200704 \
--disk path=/data/kvm/ocp4-master1.qcow2,bus=virtio,size=200 \
--os-variant rhel8.0 --network bridge=br0,model=virtio \
--boot menu=on --cdrom /data/ocp4/master-1.iso 

virsh list --all

virsh start ocp4-master1


```

### master0 node

```bash
########################################################
# master0 
yum -y install tigervnc-server tigervnc gnome-terminal gnome-session gnome-classic-session gnome-terminal nautilus-open-terminal control-center liberation-mono-fonts google-noto-sans-cjk-fonts google-noto-sans-fonts fonts-tweak-tool

yum install -y    qgnomeplatform   xdg-desktop-portal-gtk   NetworkManager-libreswan-gnome   PackageKit-command-not-found   PackageKit-gtk3-module   abrt-desktop   at-spi2-atk   at-spi2-core   avahi   baobab   caribou   caribou-gtk2-module   caribou-gtk3-module   cheese   compat-cheese314   control-center   dconf   empathy   eog   evince   evince-nautilus   file-roller   file-roller-nautilus   firewall-config   firstboot   fprintd-pam   gdm   gedit   glib-networking   gnome-bluetooth   gnome-boxes   gnome-calculator   gnome-classic-session   gnome-clocks   gnome-color-manager   gnome-contacts   gnome-dictionary   gnome-disk-utility   gnome-font-viewer   gnome-getting-started-docs   gnome-icon-theme   gnome-icon-theme-extras   gnome-icon-theme-symbolic   gnome-initial-setup   gnome-packagekit   gnome-packagekit-updater   gnome-screenshot   gnome-session   gnome-session-xsession   gnome-settings-daemon   gnome-shell   gnome-software   gnome-system-log   gnome-system-monitor   gnome-terminal   gnome-terminal-nautilus   gnome-themes-standard   gnome-tweak-tool   nm-connection-editor   orca   redhat-access-gui   sane-backends-drivers-scanners   seahorse   setroubleshoot   sushi   totem   totem-nautilus   vinagre   vino   xdg-user-dirs-gtk   yelp

yum install -y    cjkuni-uming-fonts   dejavu-sans-fonts   dejavu-sans-mono-fonts   dejavu-serif-fonts   gnu-free-mono-fonts   gnu-free-sans-fonts   gnu-free-serif-fonts   google-crosextra-caladea-fonts   google-crosextra-carlito-fonts   google-noto-emoji-fonts   jomolhari-fonts   khmeros-base-fonts   liberation-mono-fonts   liberation-sans-fonts   liberation-serif-fonts   lklug-fonts   lohit-assamese-fonts   lohit-bengali-fonts   lohit-devanagari-fonts   lohit-gujarati-fonts   lohit-kannada-fonts   lohit-malayalam-fonts   lohit-marathi-fonts   lohit-nepali-fonts   lohit-oriya-fonts   lohit-punjabi-fonts   lohit-tamil-fonts   lohit-telugu-fonts   madan-fonts   nhn-nanum-gothic-fonts   open-sans-fonts   overpass-fonts   paktype-naskh-basic-fonts   paratype-pt-sans-fonts   sil-abyssinica-fonts   sil-nuosu-fonts   sil-padauk-fonts   smc-meera-fonts   stix-fonts   thai-scalable-waree-fonts   ucs-miscfixed-fonts   vlgothic-fonts   wqy-microhei-fonts   wqy-zenhei-fonts

vncpasswd

cat << EOF > ~/.vnc/xstartup
#!/bin/sh
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
gnome-session &
EOF
chmod +x ~/.vnc/xstartup

vncserver :1 -geometry 1280x800
# 如果你想停掉vnc server，这么做
vncserver -kill :1

firewall-cmd --permanent --add-port=6001/tcp
firewall-cmd --permanent --add-port=5901/tcp
firewall-cmd --reload

# 配置kvm环境
yum -y install qemu-kvm libvirt libvirt-python libguestfs-tools virt-install virt-viewer virt-manager

systemctl enable libvirtd
systemctl start libvirtd

brctl show
virsh net-list

cat << EOF >  /data/virt-net.xml
<network>
  <name>br0</name>
  <forward mode='bridge'>
    <bridge name='br0'/>
  </forward>
</network>
EOF

virsh net-define --file virt-net.xml
virsh net-dumpxml br0
# virsh net-undefine openshift4
# virsh net-destroy openshift4
virsh net-autostart br0
virsh net-start br0

cp /etc/sysconfig/network-scripts/ifcfg-em1 /etc/sysconfig/network-scripts/ifcfg-em1.orig

cat << EOF > /etc/sysconfig/network-scripts/ifcfg-em1
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=em1
DEVICE=em1
ONBOOT=yes
# IPADDR=117.177.241.18
# PREFIX=24
# GATEWAY=117.177.241.1
IPV6_PRIVACY=no
# DNS1=117.177.241.16
BRIDGE=br0
EOF

cat <<EOF > /etc/sysconfig/network-scripts/ifcfg-br0 
TYPE=Bridge
BOOTPROTO=static
IPADDR=117.177.241.18
GATEWAY=117.177.241.1
DNS1=117.177.241.16
ONBOOT=yes
DEFROUTE=yes
NAME=br0
DEVICE=br0
PREFIX=24
EOF

systemctl restart network

mkdir -p /data/ocp4
mkdir -p /data/kvm

pigz -dc ocp4-master0.qcow2.tgz | tar xf -

virt-install --name=ocp4-master0 --vcpus=20 --ram=200704 \
--disk path=/data/kvm/ocp4-master0.qcow2,bus=virtio,size=200 \
--os-variant rhel8.0 --network bridge=br0,model=virtio \
--boot menu=on 

virsh list --all

virsh start ocp4-master0

```

### master2 node

```bash
########################################################
# master2 
yum -y install tigervnc-server tigervnc gnome-terminal gnome-session gnome-classic-session gnome-terminal nautilus-open-terminal control-center liberation-mono-fonts google-noto-sans-cjk-fonts google-noto-sans-fonts fonts-tweak-tool

yum install -y    qgnomeplatform   xdg-desktop-portal-gtk   NetworkManager-libreswan-gnome   PackageKit-command-not-found   PackageKit-gtk3-module   abrt-desktop   at-spi2-atk   at-spi2-core   avahi   baobab   caribou   caribou-gtk2-module   caribou-gtk3-module   cheese   compat-cheese314   control-center   dconf   empathy   eog   evince   evince-nautilus   file-roller   file-roller-nautilus   firewall-config   firstboot   fprintd-pam   gdm   gedit   glib-networking   gnome-bluetooth   gnome-boxes   gnome-calculator   gnome-classic-session   gnome-clocks   gnome-color-manager   gnome-contacts   gnome-dictionary   gnome-disk-utility   gnome-font-viewer   gnome-getting-started-docs   gnome-icon-theme   gnome-icon-theme-extras   gnome-icon-theme-symbolic   gnome-initial-setup   gnome-packagekit   gnome-packagekit-updater   gnome-screenshot   gnome-session   gnome-session-xsession   gnome-settings-daemon   gnome-shell   gnome-software   gnome-system-log   gnome-system-monitor   gnome-terminal   gnome-terminal-nautilus   gnome-themes-standard   gnome-tweak-tool   nm-connection-editor   orca   redhat-access-gui   sane-backends-drivers-scanners   seahorse   setroubleshoot   sushi   totem   totem-nautilus   vinagre   vino   xdg-user-dirs-gtk   yelp

yum install -y    cjkuni-uming-fonts   dejavu-sans-fonts   dejavu-sans-mono-fonts   dejavu-serif-fonts   gnu-free-mono-fonts   gnu-free-sans-fonts   gnu-free-serif-fonts   google-crosextra-caladea-fonts   google-crosextra-carlito-fonts   google-noto-emoji-fonts   jomolhari-fonts   khmeros-base-fonts   liberation-mono-fonts   liberation-sans-fonts   liberation-serif-fonts   lklug-fonts   lohit-assamese-fonts   lohit-bengali-fonts   lohit-devanagari-fonts   lohit-gujarati-fonts   lohit-kannada-fonts   lohit-malayalam-fonts   lohit-marathi-fonts   lohit-nepali-fonts   lohit-oriya-fonts   lohit-punjabi-fonts   lohit-tamil-fonts   lohit-telugu-fonts   madan-fonts   nhn-nanum-gothic-fonts   open-sans-fonts   overpass-fonts   paktype-naskh-basic-fonts   paratype-pt-sans-fonts   sil-abyssinica-fonts   sil-nuosu-fonts   sil-padauk-fonts   smc-meera-fonts   stix-fonts   thai-scalable-waree-fonts   ucs-miscfixed-fonts   vlgothic-fonts   wqy-microhei-fonts   wqy-zenhei-fonts

vncpasswd

cat << EOF > ~/.vnc/xstartup
#!/bin/sh
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
gnome-session &
EOF
chmod +x ~/.vnc/xstartup

vncserver :1 -geometry 1280x800
# 如果你想停掉vnc server，这么做
vncserver -kill :1

firewall-cmd --permanent --add-port=6001/tcp
firewall-cmd --permanent --add-port=5901/tcp
firewall-cmd --reload

# 配置kvm环境
yum -y install qemu-kvm libvirt libvirt-python libguestfs-tools virt-install virt-viewer virt-manager

systemctl enable libvirtd
systemctl start libvirtd

brctl show
virsh net-list

cat << EOF >  /data/virt-net.xml
<network>
  <name>br0</name>
  <forward mode='bridge'>
    <bridge name='br0'/>
  </forward>
</network>
EOF

virsh net-define --file virt-net.xml
virsh net-dumpxml br0
# virsh net-undefine openshift4
# virsh net-destroy openshift4
virsh net-autostart br0
virsh net-start br0

cp /etc/sysconfig/network-scripts/ifcfg-em1 /etc/sysconfig/network-scripts/ifcfg-em1.orig

cat << EOF > /etc/sysconfig/network-scripts/ifcfg-em1
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=em1
DEVICE=em1
ONBOOT=yes
# IPADDR=117.177.241.22
# PREFIX=24
# GATEWAY=117.177.241.1
IPV6_PRIVACY=no
# DNS1=117.177.241.16
BRIDGE=br0
EOF

cat <<EOF > /etc/sysconfig/network-scripts/ifcfg-br0 
TYPE=Bridge
BOOTPROTO=static
IPADDR=117.177.241.22
GATEWAY=117.177.241.1
DNS1=117.177.241.16
ONBOOT=yes
DEFROUTE=yes
NAME=br0
DEVICE=br0
PREFIX=24
EOF

systemctl restart network

mkdir -p /data/ocp4
mkdir -p /data/kvm

pigz -dc ocp4-master2.qcow2.tgz | tar xf -

virt-install --name=ocp4-master2 --vcpus=20 --ram=200704 \
--disk path=/data/kvm/ocp4-master2.qcow2,bus=virtio,size=200 \
--os-variant rhel8.0 --network bridge=br0,model=virtio \
--boot menu=on 

virsh list --all

virsh start ocp4-master2

```



### infra0 node

```bash
systemctl disable firewalld.service
systemctl stop firewalld.service

```

### infra1 node

```bash
systemctl disable firewalld.service
systemctl stop firewalld.service


```