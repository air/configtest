Requirements:

- DONE Be able to spin up a new droplet and have it automatically in config management.
- Be able to spin up a new droplet and have it ready to go with Docker and my preferred env.
- Take an action across all Docker droplets.
- Chromebook (and other clients like vagrant on Windows) register with config master for env updates. SECURITY!

# Creating the config master droplet

Use SSH key to log in as root:

    # apt-get update
    # apt-get dist-upgrade -y
    # reboot
    # apt-get upgrade -y
    # apt-get autoremove -y
    # adduser salt
    # visudo

In visudo you want:

    # User privilege specification
    root    ALL=(ALL:ALL) ALL
    salt    ALL=(ALL:ALL) ALL

Log out root and `ssh salt@config` to get back in and go to Salt section.

# Chef

# Puppet

# Ansible

# Saltstack

You need a Salt *master*. Your nodes have a *minion* client installed and told where the master is. The master then sends commands to minions.

By default the Salt master listens on ports 4505 and 4506 on all interfaces (0.0.0.0).

## Install master

As user salt:

    $ sudo echo ok
    $ sudo add-apt-repository -y ppa:saltstack/salt
    $ sudo apt-get update
    $ sudo apt-get install -y salt-master
    $ service --status-all 2>&1 | grep salt
    # You should see ` [ + ]  salt-master`.
    $ sudo salt-key -L
    # You should see an empty list of keys.

The master log can be viewed with `cat /var/log/salt/master` but with default settings, unless something goes wrong this file will be empty.

## Set up a minion

Create a new droplet, log in as root:

    # add-apt-repository -y ppa:saltstack/salt
    # apt-get update
    # apt-get install -y salt-minion
    # service --status-all 2>&1 | grep salt
    # vi /etc/salt/minion
    # # set master: <IP>
    # service salt-minion restart

You can understand what the minion is doing by watching the log:

    # cat /var/log/salt/minion

You can see that first the minion couldn't find a master at the hostname `salt`. Once this was fixed and restarted, we see

    [salt.crypt       ][ERROR   ] The Salt Master has cached the public key for this node, this salt minion will wait for 10 seconds before attempting to re-authenticate

This means the minion is connected but the master hasn't yet accepted the key.

## Tell the master to accept the minion

Back on the master, you can see a minion wants to register:

    $ sudo salt-key -L
    Accepted Keys:
    Unaccepted Keys:
    salt01
    Rejected Keys:

So let's do it and confirm:

    $ sudo salt-key -y -a salt01
    The following keys are going to be accepted:
    Unaccepted Keys:
    salt01
    Key for minion salt01 accepted.
    salt@configmaster:~$ sudo salt-key -L
    Accepted Keys:
    salt01
    Unaccepted Keys:
    Rejected Keys:

Note that `salt-key -A` will accept all keys currently awaiting acceptance.

## Command your minion

Minions will only execute *functions* that are known to Salt. They have names like `test.ping`.

[Full module list](http://docs.saltstack.com/en/latest/ref/modules/all/index.html#all-salt-modules).

    # You can address a specific minion:
    $ sudo salt salt01 test.ping
    salt01:
        True
    # or address all minions to dump their IPs:
    $ sudo salt '*' network.ip_addrs
    salt01:
        - 104.131.47.206

You can get away with not escaping the * asterisk but it's probably not a great idea.

The `cmd.run` function does allow arbitrary shell commands though:

    $ sudo salt '*' cmd.run 'uname -a'
    salt01:
        Linux salt01 3.13.0-24-generic #47-Ubuntu SMP Fri May 2 23:30:00 UTC 2014 x86_64 x86_64 x86_64 GNU/Linux

## How to generate minions easily?

So we're down to two steps:
1. Create a droplet
2. Log in and install `salt-minion`.

How to automate this? Let's snapshot the minion as a template. First we need to power it off remotely:

    $ sudo salt salt01 system.poweroff
    salt01:

And snapshot it in the web console. When that finishes, create a new droplet `salt02` from the image.

The new node looks good. It should generate its minion ID from the hostname.

It does not however show up on the master. Running in debug, we can see we have cached leftovers from salt01:

    # service salt-minion stop
    salt-minion stop/waiting
    # salt-minion -l debug
    [DEBUG   ] Reading configuration from /etc/salt/minion
    [INFO    ] Using cached minion ID from /etc/salt/minion_id: salt01
    ...

Whoops. Time to erase our cached identity to create a true template - [credit to this post](http://superuser.com/questions/695917/how-to-make-salt-minion-generate-new-keys):

    # service salt-minion stop
    # rm /etc/salt/pki/minion/minion_master.pub
    # rm /etc/salt/pki/minion/minion.*
    # cat /dev/null >/etc/salt/minion_id

Take a new snapshot and destroy `salt02`, then recreate from the image. Your SSH client may freak out at the server key changing - I needed to remove the IP from `known_hosts` to forget the old server.

The new minion will generate a new identity on startup - based on its hostname - and show immediately in the master:

    $ sudo salt-key -L
    Accepted Keys:
    salt01
    Unaccepted Keys:
    salt02
    Rejected Keys:

So let's sanity check and we can start doing real work:

    $ sudo salt-key -A -y
    The following keys are going to be accepted:
    Unaccepted Keys:
    salt02
    Key for minion salt02 accepted.
    $ sudo salt-key -L
    Accepted Keys:
    salt01
    salt02
    Unaccepted Keys:
    Rejected Keys:

## Misc

syndic - a proxy that runs alongside master X, accepting commands from a different master Y.