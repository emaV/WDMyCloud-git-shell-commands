# WDMyCloud git server
How to use WD My Cloud as Git source code repository server.

Thanks to:
* [How To Use Your Network Drive as a Git Server](https://cutecoder.org/software/git-server-network-drive/)
* [Hosting an admin-friendly git server with git-shell](http://planzero.org/blog/2012/10/24/hosting_an_admin-friendly_git_server_with_git-shell)

## Installation Overview
1. Enable SSH login from the web panel.
2. Install Git from the web panel
3. Setup git account and its home directory
4. Setup the repository directory for Git
5. Install git-shell-commands
6. Setup SSH password-less 
7. Make the change permanent
8. Test modified startup file
9. Finally reboot and test

## Enable SSH Login

1. Login on web admin for the WD My Cloud
2. Go to the Settings tab
3. Select Network in the left navigation menu
4. Enable SSH
5. Click on the Configure link and enter a strong password.
6. Click on Save

### Install Git

WD MyCloud control panel makes installing git real easy.

1.  Open the drive’s web control panel.
2.  Select the Apps tab.
3.  Click the Add button in the left panel. A list of installable applications should show.
4.  Find the entry for git and enable its checkmark.
5.  Click on Install.

The steps above will install a version of git command line utilities that is compatible with your drive’s firmware.

_Unfortunately WD’s software stops there and you’re on your own from this point on. They don’t provide a way for you to start serving git repositories from your network drive._

Actually one of the many good things of the little gem known as `git` is that setup server from the client is quite easy. It is the same software, actually.

It is just matter to setup an user and some configuration here and there, some little scripting as icing.


### Setup git account

There are tools available for creating users in the shell but for the `git` user it is simple to do it manually.

Edit the user accounts master file in this path: `/usr/local/config/passwd`.

In this file, each field is separated by a colon (“:”) and each line contains a record describing a user account (which includes some built-in accounts.
The third field should be the unique id,, the fourth should match the other user group.
The second to last field points to the user’s home directory (`/home/git`).
The last field set the shell and we'll use the [git-shell](https://git-scm.com/docs/git-shell).

```
git:x:1009:1004:,,,:/home/git:/usr/local/bin/git-shell
```
The `git` home directory should be created with proper permissions

```
$ mkdir /home/git
$ chown -R git /home/git
$ chgrp -R family /home/git
$ chmod -R og+rwx /home/git
```

### Setup GIT repository directory

Create a directory that will host all the Git repository and set proper permissions

```
$ mkdir /mnt/HD/HD_a2/git
$ ln -s /mnt/HD/HD_a2/git /shares/git
$ chown -R git /shares/git
$ chgrp -R family /shares/git
$ chmod -R og+rwx /shares/git
```

#### Create ssh keys for the git user

You may want to be able to push/pull code to external repository so the user `git` needs  the private/public keys.

```
$ sudo -u git -s
$ cd /home/git
$ ssh-keygen -t rsa -b 4096 -C "git@WD-emaV" -q -N "" -f "/home/git/.ssh/id_rsa"
$ touch .ssh/authorized_keys
$ chmod 600 .ssh/authorized_keys
```

If you get the error `PRNG is not SEEDED` you probably need to check some permissions as per [UNIX Health Check - PRNG is not SEEDED](https://unixhealthcheck.com/blog?id=304).

Check permissions on random numbers generators, the "others" must have "read" access to these devices:

```
root # ls -l /dev/random /dev/urandom
crw-rw-r--    1 root     root        1,   8 Jan  1  1970 /dev/random
crw-rw-r--    1 root     root        1,   9 Jan  1  1970 /dev/urandom
```

If the permissions are not set correctly, change them as follows:

```
$ chmod o+r /dev/random /dev/urandom
```

Now stop and start the SSH daemon again, and retry if ssh works.

### Install git-shell-commands

Install git shell commands and set proper permissions.

```
$ cd /home/git
$ wget --no-check-certificate  https://github.com/emaV/WDMyCloud-git-shell-commands/archive/master.zip
$ unzip -j master.zip -d git-shell-commands
$ chown -R git git-shell-commands
$ chgrp -R family git-shell-commands
$ chmod -R og+rwx git-shell-commands
```

## Setup SSH password-less 

This step will allow remote user to connect the server using their keys.

For every remote users append the public_key to the `authorized_keys` file with the git-shell-commands add-key command.

```
$ cd /home/git
$ sudo -u git -s
$ ./git-shell-commands/key-add
```

Use vi to edit file `/etc/ssh/sshd_config`
Locate the line beginning with AllowUsers

Append the user `git` at the end of it and make sure that it is separated by a space from other user names in that line.

```
AllowUsers root sshd git
```

### Make the change permanent
By default WD’s firmware only allows SSH logins by one special user. At every reboot the SSH configuration gets re-set to to allow login only by the special ssh user – which defeats the purpose of having a git server in the first place. Fortunately there are some startup scripts that are located on the drives themselves – not in the reset-every-boot flash storage – and we can use some of these scripts to re-apply our custom configuration.

We’ll use git’s startup script to do this. Use vi to modify git’s startup file" `/mnt/HD/HD_a2/Nas_Prog/git/start.sh`

And add these two lines at the bottom to enable password-less SSH login at every boot.

```
sed -ir 's/(AllowUsers .*)/\1 git/' /etc/ssh/sshd_config
kill -HUP `cat /var/run/sshd.pid
```
### Test modified startup file
Try running git’s modified startup file and then inspect the sshd_config file and see whether the script changes the configuration file as expected. If you somehow corrupted the startup script, you can uninstall git from the drive’s control panel and then re-install it.

Try to connect from a remote with a user added to `authorize_keys`, you should get the git prompt

```
Run 'help' for help, or 'exit' to leave.  Available commands:
create
delete
key-add
key-delete
key-list
list
git> 
```

### Finally reboot and test

It’s time to test your setup.

Reboot your WD My Cloud from the web interface.

Login to git account and create a repository:

```
$ ssh git@WD
Run 'help' for help, or 'exit' to leave.  Available commands:
create
delete
key-add
key-delete
key-list
list
git> create test
Initialized empty Git repository in /mnt/HD/HD_a2/git/test.git/
Repository test (/shares/git/test.git) created!
exit
```
On the local machine:
```
$ git clone WD:/shares/git/test.git
Cloning into 'test'...
warning: You appear to have cloned an empty repository.
```

*It works!*
