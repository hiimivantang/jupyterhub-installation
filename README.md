## JupyterHub

This documentation covers prerequisites, installation, configuration for JupyterHub.



### Prerequisites
* Python 3.4 or greater
* nodejs/npm install a recent version of nodejs/npm. For example, install it on Linux (Debian/Ubuntu) using:
```bash
$ sudo apt-get install npm nodejs-legacy
```
The nodejs-legacy package installs the node executable and is currently required for npm to work on Debian/Ubuntu.
* TLS certificate and key for HTTPS communication



### Install packages

JupyterHub can be installed with `pip`, and the proxy with `npm`:
```bash
$ npm install -g configurable-http-proxy
$ pip install jupyterhub
```


### Configurations

Since JupyterHub needs to spawn processes as other users (that's kinda the point of it), the simplest way is to run it as root, spawning user servers with setuid. But this isn't especially safe, because you have a process running on the public web as root. Any vulnerability in the JupyterHub process could be pretty catastrophic.

A more prudent way to run the server while preserving functionality is to create a dedicated user with **sudo access restricted to launching and monitoring single-user servers.**

To do this, first create a user that will run the Hub:
```bash
# creates new user with no login shell or password
$ sudo useradd rhea -r
```


Next, you will need the sudospawner custome Spawner to enable monitoring the single-user servers with sudo:
```bash
$ pip install sudospawner
```


Now we have to configure sudo to allow the Hub user (rhea) to launch the sudospawner script on behalf of our hub users (here zoe and wash). We want to confine these permissions to only what we really need. To do this we add to /etc/sudoers (use visudo for safe editing of sudoers):
1. add JupyterHub's path to the secure_path in sudoers
2. specify the command that rhea can execute on behalf of users (JUPYTER_CMD)
3. give user `rhea` permission to run JUPYTER_CMD on behalf of users in `jupyterhub` group without entering a password

For example:
```bash
Defaults        secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin:/home/vagrant/anaconda3/bin"

# the command(s) the Hub can run on behalf of the above users without needing a password
# the exact path may differ, depending on how sudospawner was installed
Cmnd_Alias JUPYTER_CMD = /home/vagrant/anaconda3/bin/sudospawner

# actually give the Hub user permission to run the above command on behalf
# of the above users without prompting for a password
rhea ALL=(%jupyterhub) NOPASSWD:JUPYTER_CMD

```


Provided that the jupyterhub group exists, there will be no need to edit /etc/sudoers again. A new user will gain access to the application easily just getting added to the group:
```bash
$ sudo addgroup jupyterhub

#for e.g.
$ sudo usermod -aG jupyterhub siyang
$ sudo usermod -aG jupyterhub zongxian
$ sudo usermod -aG jupyterhub ivantang

```


Verify that users in `jupyterhub` group **doesn't need to enter a password** to run the sudospawner command:
```bash
$ sudo -u rhea sudo -n -u $USER /home/vagrant/anaconda3/bin/sudospawner --help
```

And this should fail:
```bash
$ sudo -u rhea sudo -n -u $USER echo 'hello'
```


So now, you have a dedicate user `rhea` for running __only__ sudospawner



PAM authentication is used by JupyterHub. To use PAM, the process may need to be able to read the shadow password database.


```bash
$ ls -l /etc/shadow
-rw-r----- 1 root shadow 1829 Sep 20 02:22 /etc/shadow
```
If there's already a shadow group with the correct permissions as shown above, you are set!

We want our new user to be able to read the shadow passwords, so add it to the shadow group:
```bash
sudo usermod -a -G shadow rhea

```

JupyterHub stores its state in a database, so it needs write access to a directory. The simplest way to deal with this is to make a directory owned by your Hub user, and use that as the CWD when launching the server.
```bash
$ sudo mkdir /etc/jupyterhub
$ sudo chown rhea /etc/jupyterhub
```



Finally, start the server as our newly configured user
```bash
$ cd /etc/jupyterhub
$ sudo -u rhea /home/vagrant/anaconda3/bin/jupyterhub --JupyterHub.spawner_class=sudospawner.SudoSpawner
```

And try logging in.


### Setup JupyterHub as a supervisor process (optional)

Create supervisor .conf file for jupyterhub.
```bash
sudo touch /etc/supervisor/conf.d/jupyterhub.conf
```

Your supervisor `jupyterhub.conf` should look like this:
```bash
[program:jupyterhub]
command=sudo -u rhea /home/vagrant/anaconda3/bin/jupyterhub --JupyterHub.spawner_class=sudospawner.SudoSpawner
user=vagrant
autostart=true
autorestart=true
directory=/etc/jupyterhub
stderr_logfile=/var/log/jupyterhub.err.log
stdout_logfile=/var/log/jupyterhub.out.log
```

