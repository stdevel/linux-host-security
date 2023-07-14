# vagrant

This directory contains a demo machine, that will be automatically set-up by [HashiCorp Vagrant](https://vagrantup.com) and hardening using Ansible.
It requires having a supported hypervisor (e.g. VirtualBox or KVM) installed.
To check compliance, you will need to have [InSpec](https://inspec.io) installed.

## Create VM

Spin-up the VM and display the connection details:

```command
$ vagrant up --no-provision
...
$ vagrant ssh-config | grep -Ei 'hostname|port'
  HostName 192.168.121.105
  Port 22
```

## Check compliance

Clone and run the [`linux-baseline` InSpec profile](https://github.com/dev-sec/linux-baseline):

```command
$ git clone https://github.com/dev-sec/linux-baseline
virtualbox$ inspec exec linux-baseline -t ssh://vagrant@192.168.121.105 -i .vagrant/machines/default/libvirt/private_key --sudo
kvm$ inspec exec linux-baseline -t ssh://vagrant@localhost -i .vagrant/machines/default/virtualbox/private_key --port 2222 --sudo
```

Result should look pretty bad:

```command
...
Profile Summary: 25 successful controls, 32 control failures, 1 control skipped
Test Summary: 122 successful, 68 failures, 1 skipped
```

**Note:** You might need to adjust the IP address/hostname and SSH port.

## Harden the system

Run the Ansible playbook:

```command
$ vagrant provision
```

Run the InSpec baselines again - you should see that the most findings are now fixed:

```command
...
Profile Summary: 54 successful controls, 3 control failures, 1 control skipped
Test Summary: 187 successful, 3 failures, 1 skipped
```

## Testing AIDE

Log into the machine and create a file:

```command
$ vagrant ssh
[vagrant@centos8s ~] $ echo 'chad' | sudo tee -a /etc/sgiertz.conf
```

Let AIDE check the system - it should find a new/changed file:

```command
$ sudo aide -C
AIDE found differences between database and filesystem!!

Summary:
  Total number of entries:      83746
  Added entries:                1
...
f++++++++++++++++: /etc/sgiertz.conf
...
```

## Testing fail2ban

Try to login via SSH multiple times with an incorrect password:

```command
$ ssh vagrant@192.168.121.105
vagrant@192.168.121.105's password: 
Permission denied, please try again.
vagrant@192.168.121.105's password: 
Permission denied, please try again.
vagrant@192.168.121.105's password: 
vagrant@192.168.121.105: Permission denied (publickey,gssapi-keyex,gssapi-with-mic,password).
```

**Note:** You might need to adjust the IP address/hostname and SSH port.

You won't be able to login anymore:

```command
$ ssh vagrant@192.168.121.105
ssh: connect to host 192.168.121.105 port 22: Connection refused
```

Open the VM console and login using username and password `vagrant`. Run the following command to verify that the user was blocked:

```command
$ sudo fail2ban-client get sshd banned
['192.168.121.1']
```

Unblock the user:

```command
$ sudo fail2ban-client set sshd unbanip 10.0.2.2
1
```

Logins are now possible again.
