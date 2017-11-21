## Preparing the server you want to provision with ansible

Target OS
- Archlinux
- Debian9 stretch
- Centos7

#### When creating a Archlinux server

Create user to use with ansible as root

User name should be ansible

	pacman -S curl zsh openssh
	useradd -m -G wheel -s /bin/zsh ansible
	su - ansible
	ssh-keygen -t rsa -b 4096
	cd .ssh/
	mv id_rsa.pub authorized_keys
	chmod 600 authorized_keys
	curl https://github.com/masasam.keys >> ~/.ssh/authorized_keys ← Register public key registered with github

Return to root

	systemctl enable sshd
	systemctl start sshd

Set host name

	hostname archlinux

vi /etc/hosts

	127.0.0.1   localhost.localdomain   localhost archlinux

vi /etc/pam.d/su

	# Remove comment out
	auth required pam_wheel.so use_uid

visudo

	echo 'ansible ALL=(ALL) ALL' | sudo EDITOR='tee -a' visudo
	echo '%wheel ALL=(ALL) ALL' | sudo EDITOR='tee -a' visudo
	echo '%wheel ALL=(ALL) NOPASSWD: ALL' | sudo EDITOR='tee -a' visudo

## Install ansible on my machine

	sudo pacman -S ansible
	ghq get -p masasam/ansible-setup-server

## Perform provisioning by ansible

	ansible-playbook main.yml

Write variables and passwords in group_vars/server.yml
Encrypt server.yml in advance with the following command

	ansible-vault encrypt server.yml

If you write the location of the cryptographic password in
ansible.cfg, you do not have to hit the password each time
(The contents are only the password)

	vault_password_file = ~/Dropbox/ansible/vault_pass

What is in server.yml

	hostname: 'yourhost' ← Linux host name
	domain: 'yourdomain' ← Main domain
	subdomain: 'subdomain.yourdomain' ← sub domain(using blog)
	username: 'ansible' ← User name ansible ssh
	mailroot: 'youremailaddress' ← E-mail address to transfer root's mail
	monitalert: 'youremailaddress' ← Destination of alert mail from monit
	infopassword: '913336a8ecba7764cd81245c2c6b'
	mariadbrootpassword: 'mariadbrootpassword' ← The password of the mariadb root user
	mackerelapikey: 'yourmackerelapikey' ← mackerel's apikey
	dbname: 'yourdbbame' ← DB name used in mariadb
	dbpassword: 'yourdbpassword' ← That password
	docroot: '/home/html' ← Main document route for nginx
	docrootblog: '/home/blog' ← Document root of blog for nginx
	docrootadminer: '/usr/share/webapps/adminer' ← Adminer's document root for nginx

Infopassword will be the password for the email address of info@yourdomain
How to make infopassword

	doveadm pw
	Enter new password: yourpassword
	Retype new password: yourpassword

With

	{CRAM-MD5}913336a8ecba7764cd81245c2c6b

Because it is

	infopassword: '913336a8ecba7764cd81245c2c6b'

#### Update the server only playbook

	ansible-playbook update.yml

## Make a test guest container locally

---- If a test environment is unnecessary, you do not need the following ----

The guest environment is vps of the production environment In the
guest test environment systemd-nspawn
In the following, systemd-nspawn is constructed by reading with ssh in vps

Preparing a test container

	sudo pacman -S arch-install-scripts
	mkdir systemdcontainer
	sudo pacstrap -i -c -d ~/systemdcontainer base base-devel --ignore linux
	sudo systemd-nspawn -b -D ~/systemdcontainer --bind=/var/cache/pacman/pkg

In the container

	pacman -Sy bash-completion openssh

Arch linux is the default for python 3 Make ansible use python 2
It seems that centos8 also becomes python 3, so be careful

	pacman -Sy python2 zsh

Create user to use with ansible

	useradd -m -G wheel -s /bin/zsh ansible
	su - ansible
	ssh-keygen -t rsa -b 4096
	cd .ssh/
	mv id_rsa.pub authorized_keys
	chmod 600 authorized_keys
	curl https://github.com/masasam.keys >> ~/.ssh/authorized_keys ← Register public key registered with github

Return to root

	systemctl enable sshd
	systemctl start sshd

Set host name

	hostname archtest

vi /etc/hosts

	127.0.0.1   localhost.localdomain   localhost archtest

#### Make the user who can become root only users belonging to the wheel group

	usermod -G wheel ansible

vi /etc/pam.d/su

	# Remove comment out
	auth required pam_wheel.so use_uid

#### Set up a user (group) that sudo can use

visudo

	#Defaults    requiretty(Comment out only for centos)

	## User privilege specification
	root ALL=(ALL) ALL
	ansible ALL=(ALL) ALL

	# Uncomment to allow members of group wheel to execute any command
	%wheel ALL=(ALL) ALL

	## Same thing without a password
	%wheel ALL=(ALL) NOPASSWD: ALL

Shutdown at
>Ctrl-]]]

Since it became possible to connect with ssh, you can start it in the background from next time onwards.
(Now that you can log in with ssh and shut it down)

	sudo systemd-nspawn -b -D ~/systemdcontainer --bind=/var/cache/pacman/pkg &

Set the following in .ssh/config

	Host archtest
						HostName localhost
						User ansible

Login to container with ssh

	ssh archtest

## When creating a debian test container

	sudo pacman debootstrap
	yaourt -S debian-archive-keyring

	mkdir debian
	sudo debootstrap stretch debian http://ftp.jaist.ac.jp/pub/Linux/debian/

	sudo chroot debian
	passwd root

	sudo systemd-nspawn -b -D ~/debian

From here debian virtual server

	apt-get install python openssh-server zsh bash-completion sudo

	useradd -m -G sudo -s /bin/zsh ansible
	su - ansible
	ssh-keygen -t rsa -b 4096
	cd .ssh/
	mv id_rsa.pub authorized_keys
	chmod 600 authorized_keys
	curl https://github.com/masasam.keys >> ~/.ssh/authorized_keys ← Register public key registered with github

Return to root

	systemctl enable ssh
	systemctl start ssh

Set host name

	hostname debian

vi /etc/hosts

	127.0.0.1       localhost debian

Set up a user (group) that sudo can use

	update-alternatives --config editor

visudo

	## User privilege specification
	root ALL=(ALL) ALL
	ansible ALL=(ALL) ALL

	# Uncomment to allow members of group wheel to execute any command
	%sudo ALL=(ALL) ALL

	## Same thing without a password
	%sudo ALL=(ALL) NOPASSWD: ALL

## When creating a test container for centos

	yaourt yum
	mkdir centos

	sudo vim /etc/yum/repos.d/centos.repo
	[centos]
	name=centos
	baseurl=http://ftp.jaist.ac.jp/pub/Linux/CentOS/7/os/x86_64/
	enabled=1

	sudo yum -y --releasever=7 --installroot=~/centos groupinstall "Base"

	sudo chroot centos
	passwd root

	sudo systemd-nspawn -b -D ~/centos

Create user to use with ansible as root
User name should be ansible

	yum install python openssh-server zsh bash-completion sudo
	useradd -m -G wheel -s /bin/zsh ansible
	su - ansible
	ssh-keygen -t rsa -b 4096
	cd .ssh/
	mv id_rsa.pub authorized_keys
	chmod 600 authorized_keys
	curl https://github.com/masasam.keys >> ~/.ssh/authorized_keys ← Register public key registered with github

Return to root

	systemctl enable sshd
	systemctl start sshd

Set host name

	hostname centos

vi /etc/hosts

	127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4 centos

vi /etc/pam.d/su

	# Remove comment out
	auth required pam_wheel.so use_uid

Set up a user (group) that sudo can use

visudo

	#Defaults    requiretty(Confirm whether commented out)

	## User privilege specification
	root ALL=(ALL) ALL
	ansible ALL=(ALL) ALL

	# Uncomment to allow members of group wheel to execute any command
	%wheel ALL=(ALL) ALL

	## Same thing without a password
	%wheel ALL=(ALL) NOPASSWD: ALL
