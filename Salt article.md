# How to use Digital Ocean snapshots to create a Salt minion army

### Introduction

This article explains how to combine [Salt's](http://www.saltstack.com/community/) simple config management with the power of Digital Ocean's snapshot feature.

First, the basics. You can find a [tutorial series on this site](https://www.digitalocean.com/community/tutorial_series/an-introduction-to-salt) to get you up and running with Salt, namely:

1. [How to Install Salt on Ubuntu](https://www.digitalocean.com/community/tutorials/how-to-install-salt-on-ubuntu-12-04) illustrates how to get a simple master/minion pair running on the same server. You might want to try this first to get a feel for how masters and minions communicate. In the guide below we'll install a more practical setup.
1. [How to Create Your First Salt Formula](https://www.digitalocean.com/community/tutorials/how-to-create-your-first-salt-formula). Once you're up and running, this is essential reading to learn how to command your new army of droplets.

This guide is standalone and assumes you're starting from scratch.

If you want to reuse your setup from tutorial 1, you can skip to **Set up a minion** below. A clean start is preferred though, since we're going to create a Salt user rather than work directly with `root`.

## What's my motivation?

Sooner or later you're going to have a bunch of Digital Ocean droplets all doing excitingly different things. Over time, they're going to need maintenance: packages become outdated; the OS needs security patches; you need to rotate passwords or keys.

Wouldn't it be cool if all your droplets were seamlessly added to a management network?

Wouldn't it be cool if you could run `sudo apt-get dist-upgrade -y` on all of them **in one shot**?

Yes it would. Let's get going.

## Wait, why not salt-cloud?

You can do this with another tool called [salt-cloud](http://salt-cloud.readthedocs.org/en/latest/). It uses the DigitalOcean API to provision droplets. It runs a complex script called [salt-bootstrap](https://github.com/saltstack/salt-bootstrap) on all the new machines.

We're not going to do that.

Let's get clever with our own template, because:

- [Keeping it simple](http://en.wikipedia.org/wiki/KISS_principle) is good. Salt-cloud is overkill for a small number of servers.
- You will have hands-on control over what's happening.
- You will learn deeply about Salt, snapshots and images! Learning is fun.

## Here's the plan

1. Create a *master* server. Optionally you could use an existing machine - say, [from the first tutorial](https://www.digitalocean.com/community/tutorials/how-to-install-salt-on-ubuntu-12-04) - to save a few beer tokens, but we'll assume you're starting from scratch.
1. Create a *minion* - thank you Salt developers for using this word - and test it.
1. The fun part: Tweak and snapshot the minion as a template for future droplets.
1. Demonstrate your new powers.
1. Execute a complex victory dance.

## The Salt chain of command

You need a single Salt *master* - a package called `salt-master` - in order to issue imperious commands to the unwashed, quivering minions.

Your minions have a client installed - `salt-minion`, not surprisingly - and are told where the master is. The minions ask for permission to join up, the master says yes, and off we go.

### Should the master be a minion?

The [first Salt tutorial](https://www.digitalocean.com/community/tutorials/how-to-install-salt-on-ubuntu-12-04) does this as a test scenario. The implications of doing this are:

- The master server can be managed along with everyone else. This is handy.
- You run a small risk of screwing up the master when you accidentally send a `sudo rm -rf /*` to all minions. Hint: Don't do this.

## Step one - Create the master

Create a new Ubuntu droplet called `configmaster` and log in as root. At create time, **specify an SSH key** because we are adults (it's way more secure than a password). [This tutorial is a great explainer on SSH keys](https://www.digitalocean.com/community/tutorials/how-to-use-ssh-keys-with-digitalocean-droplets).

Let's set up a new user for Salt:

    adduser salt

When prompted, set a password and default all the other questions:

    Adding user `salt' ...
    Adding new group `salt' (1001) ...
    Adding new user `salt' (1001) with group `salt' ...
    Creating home directory `/home/salt' ...
    Copying files from `/etc/skel' ...
    Enter new UNIX password:
    Retype new UNIX password:
    passwd: password updated successfully
    Changing the user information for deleteme
    Enter the new value, or press ENTER for the default
            Full Name []:
            Room Number []:
            Work Phone []:
            Home Phone []:
            Other []:
    Is the information correct? [Y/n] y

Finally, we need to edit sudo privileges:

    visudo

Notice that `visudo` launches an editor on the `/etc/sudoers` file. Make it look like this and save it out with `Ctrl-X`:

    # User privilege specification
    root    ALL=(ALL:ALL) ALL
    salt    ALL=(ALL:ALL) ALL

That's our dirty root-work done. Log out and SSH back in as `salt` using your new password.

### From zero to salt-master in one minute

As user `salt` we want to install the `salt-master` package:

    sudo echo "This verifies sudo is working, I feel like an internet hero"
    sudo add-apt-repository -y ppa:saltstack/salt
    sudo apt-get update
    sudo apt-get install -y salt-master

We can check the service exists with:

    service --status-all 2>&1 | grep salt

Which should display:

     [ + ]  salt-master

Finally we can ask the master to list any minions it knows about:

    sudo salt-key -L
    
Which should show the empty groups:

    Accepted Keys:
    Unaccepted Keys:
    Rejected Keys:

Your master is now done. Told you it was simple.

### What have I created?

Minions ask for permission to join the network by sending over a key. The `salt-key -L` command means '(L)ist all the minions currently connected'.

- Minions who are connected but remain patiently waiting for a decision are listed as *Unaccepted*.
- Notice that a master can choose to *Reject* a minion that has the wrong sort of haircut (that is, if you suspect it's not legit).

For debug purposes, the salt-master log is at e.g. `cat /var/log/salt/master`. With default settings, the log will be empty unless something is broken.

By default the Salt master listens on ports 4505 and 4506 on all interfaces (0.0.0.0). This is good to know when you start adding firewalls.

## Step two - Set up a minion

We're going to do the absolute minimum to *bootstrap* this new server.

Crucially, we're only going to do this **once**. All minions after the first will come from our template.

Create a new droplet called `salt01`. Log in as root and:

    add-apt-repository -y ppa:saltstack/salt
    apt-get update
    apt-get install -y salt-minion
    service --status-all 2>&1 | grep salt

There's one item of config business. We need to set the location of the master (an IP address for the moment). Edit the file:

    vi /etc/salt/minion

Remove the comment from the `master` line, and add your master's IP address:

    master: <^><your configmaster IP><^>

And restart the service to pick up the changes:

    service salt-minion restart

You can understand what the minion is doing by watching the log:

    cat /var/log/salt/minion

Have a look now. You can see that at first, the minion couldn't find a master at the default hostname `salt`. Once this was fixed and restarted, we see

    [salt.crypt       ][ERROR   ] The Salt Master has cached the public key for this node, this salt minion will wait for 10 seconds before attempting to re-authenticate

This means the minion is connected but the master hasn't yet *Accepted* the key.

### I for one welcome our new minion

Back on the master (as user `salt`), you can see a minion wants to register using:

    sudo salt-key -L

Which should display:

    Accepted Keys:
    Unaccepted Keys:
    salt01
    Rejected Keys:

So let's be a nice doorman and let them in:

    sudo salt-key -y -a salt01

Resulting in:

    The following keys are going to be accepted:
    Unaccepted Keys:
    salt01
    Key for minion salt01 accepted.

We can verify with the List command:

    sudo salt-key -L

Resulting in:

    Accepted Keys:
    salt01
    Unaccepted Keys:
    Rejected Keys:

Note that `salt-key -A -y` will accept all keys currently waiting for a decision.

### Test your powers

Let's expand on some of the [basics we tried in the first tutorial](https://www.digitalocean.com/community/tutorials/how-to-install-salt-on-ubuntu-12-04).

Minions can execute *functions* from *execution modules* that are defined by Salt. They have names like `test.ping`.

The [full module list](http://docs.saltstack.com/en/latest/ref/modules/all/index.html#all-salt-modules) is pretty exhaustive.

On the master, you can *target* minions in various ways. You can address a specific minion:

    sudo salt salt01 test.ping

Resulting in:

    salt01:
        True

Or address all minions, to e.g. dump their IPs:

    sudo salt '*' network.ip_addrs

Resulting in:

    salt01:
        - 123.456.78.90

Notice that your minion may return multiple IPs if you're using [Private Networking](https://www.digitalocean.com/community/tutorials/how-to-set-up-and-use-digitalocean-private-networking).

You can get away with not escaping the `*` asterisk but it's probably not a great habit.

The `cmd.run` function allows arbitrary shell commands:

    sudo salt '*' cmd.run 'uname -a'

Resulting in:

    salt01:
        Linux salt01 3.13.0-24-generic #47-Ubuntu SMP Fri May 2 23:30:00 UTC 2014 x86_64 x86_64 x86_64 GNU/Linux

So we've created a minion. Do we want to do this routine every time we add a new one? Nope. Let's create a DigitalOcean image.

## Step three - Make a template from the snapshot

Now we understand what's going on, we're going to make a template out of `salt01`. This requires a spot of minor surgery to remove its identity.

### Reset your minion's identity

When the `salt-minion` service starts up, it generates an identity based on the hostname. This `minion_id`, along with a new key are cached to disk.

We need to remove that for our template - [credit to this post for the method](http://superuser.com/questions/695917/how-to-make-salt-minion-generate-new-keys). On your `salt01` minion:

    service salt-minion stop
    rm /etc/salt/pki/minion/minion_master.pub
    rm /etc/salt/pki/minion/minion.*
    cat /dev/null >/etc/salt/minion_id

If we didn't do this, new machines created from template would all have a cached identity of `salt01`. We would see this on startup:

    salt-minion -l debug

Resulting in:

    [DEBUG   ] Reading configuration from /etc/salt/minion
    [INFO    ] Using cached minion ID from /etc/salt/minion_id: salt01
    ...

By clearing the cached identity we avoid this, ensuring that minions created from template generate a fresh identity.

### Tell the master about the identity change

Since we just blew away salt01's identity, we need to tell the master not to expect that key any more:

    sudo salt-key -D -y

The `-D` means '(D)elete all minion keys'.

### Make an image!

Digital Ocean lets you take a *snapshot*, which creates an *image*. Images can be used to create new droplets just like the old one, except for a brand new hostname.

First we need to power off `salt01`:

    poweroff

And in your DigitalOcean web console, snapshot it. Call the image something like 'Salt minion template'.

When the snapshot finishes, create a new droplet `salt02` from the image.

When `salt02` starts up everything will happen by magic, namely:

1. Thanks to the snapshot, `salt02` already has the `salt-minion` package installed and config ready to go! Great.
1. As the `salt-minion` service starts, it will generate its new identity.
1. `salt02` will then connect to the master.

### What happens to the droplet we snapshotted?

After the snapshot, `salt01` will automatically be powered on and goes through this behaviour:

1. `salt01` notices that cached identity files are not present, and so
1. Generates a new minion identity and caches it, and finally
1. Connects to the master as a new minion.

## Step four - Accept your minion army

We're all done. We have two fresh minions asking to join the master, so let them in:

    sudo salt-key -L

To see the waiting minions:

    Accepted Keys:
    Unaccepted Keys:
    salt01
    salt02
    Rejected Keys:

Let's accept them all in one shot:

    sudo salt-key -A -y

Resulting in:

    The following keys are going to be accepted:
    Unaccepted Keys:
    salt01
    salt02
    Key for minion salt02 accepted.
    Key for minion salt02 accepted.

Let's prove they're different:

    sudo salt '*' cmd.run 'uname -a; uptime'

Showing:

    salt02:
        Linux salt02 3.13.0-24-generic #47-Ubuntu SMP Fri May 2 23:30:00 UTC 2014 x86_64 x86_64 x86_64 GNU/Linux
         18:37:07 up 5 min,  1 user,  load average: 0.01, 0.03, 0.02
    salt01:
        Linux salt01 3.13.0-24-generic #47-Ubuntu SMP Fri May 2 23:30:00 UTC 2014 x86_64 x86_64 x86_64 GNU/Linux
         18:37:07 up 8 min,  1 user,  load average: 0.00, 0.01, 0.01

## Upgrade all your droplets

To illustrate the power of the new approach, how about updating the packages on all droplets in one shot? Try:

    sudo salt '*' cmd.run 'apt-get update; apt-get dist-upgrade -y'

Since this operation can take a minute or two, the command may return while it's still in progress on the minions.
At any time, you can get the ID of the last job you ran with:

    sudo salt-run jobs.list_jobs | head -6

To get a job listing with the job ID listed first:

    '20141005133439695132':
      Arguments:
      - apt-get update; apt-get dist-upgrade -y
      Function: cmd.run
      StartTime: 2014, Oct 05 13:34:39.695132
      Target: '*'

And using this job ID, you can then check full output:

    sudo salt-run jobs.lookup_jid <^>your_job_id<^>

## How to use your new managed system

**Whenever you need a new droplet in future, create it from your Salt Minion image**. As a new droplet starts up, it will automatically register with your config management network - just remember to Accept it with e.g. `sudo salt-key -A -y`.

You're free to update the custom image and make a new snapshot whenever you feel like it - just remember to clean the cached identity beforehand.

## A note on networking

Sysadmins will be comforted to learn that DigitalOcean takes care of both the hostname and MAC address on new servers. You can confirm this with e.g.

    sudo salt '*' network.hw_addr eth0

Resulting in:

    salt02:
        04:06:28:41:19:01
    salt01:
        04:04:29:22:fe:11

## What now?

### Security

Don't forget security! You'll want to take basic safety steps. [This is a great starting tutorial](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-14-04).

Here's the great news: you can set up security steps **once** in Salt and use the master to push to all new servers.

### Ideas for next steps

1. Set up a github repo called `salt`, to store your `.sls` files. Clone this repo onto your `configmaster` droplet in the `/srv` directory. Boom: control and distribution of your config is done.
1. Start writing some `.sls` states to do fun things on droplets, like:
  - Create your user for non-root login
  - Copy in all your favorite user data: `.bashrc`, `.gitconfig`, `ssh_config` and the like
  - Install and update packages
  - Perform basic security steps on a new droplet: securing `sshd`, setting up firewalls

For help on any of this, reach out to the responsive [Salt community](www.saltstack.com/community/) or leave a comment on this article.

Happy minioning!
