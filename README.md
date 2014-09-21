Requirements:

- DONE Be able to spin up a new droplet and have it automatically in config management.
- Be able to spin up a new droplet and have it ready to go with Docker and my preferred env.
- Take an action across all Docker droplets.
- Chromebook (and other clients like vagrant on Windows) register with config master for env updates. SECURITY!

# Next:
http://docs.saltstack.com/en/latest/topics/tutorials/starting_states.html
http://docs.saltstack.com/en/latest/ref/states/ordering.html
add YAML parsing rule for .sls files to vimrc on config master

# Salt concepts

Grain - a grain of information about a minion. Sent from minion -> master. Used for targeting commands.
Pillar - a tree of data kept on the master. Sent from master -> minions on a need-to-know basis. A State can have parameters filled in with data taken from a Pillar.
Nodegroup - a logical group of minions defined by a bunch of selectors. Used for targeting commands.
State - 
Highstate - 
Top file - 
Watcher - 

# Set up master State

Ever used `/srv` before? It's where you keep data for services, as defined in http://www.tldp.org/LDP/Linux-Filesystem-Hierarchy/html/srv.html

[Reference for file.managed](http://salt.readthedocs.org/en/latest/ref/states/all/salt.states.file.html). You will be using this **a lot**.

## IDEMPOTENCY

You define the target state. The tool will try to get there. If it's already there, you can keep running the tool and nothing bad will happen.

## Help yourself, don't wreck yourself

### Increase master timeout

In `/etc/salt/master`, set `timeout: 30` or similar.

### Use verbose

The `salt` command can be run with `-v` to dump the job ID for everything you run. This is great for getting the results of historical commands.

The master config file doesn't seem to allow a permanent setting, so I use `alias salt='sudo salt -v'` to force it every time.

### Test runs are your friend

Whack a `test=True` on the end of your salt commands to do a dry run. The stuff that *would* happen comes up in yellow.

# Understanding States

## 1. Standalone formula

Make the vim.sls and the vimrc. The master will pick up new stuff at runtime, no refresh needed.

    vim:
      pkg:
        - installed

    /etc/vimrc:
      file.managed:
        - require:
          - pkg: vim
        - source: salt://vimrc
        - mode: 644
        - user: root
        - group: root

Note:

- The first line is the **ID Declaration**. This is your formula name (`vim` here). When you invoke this formula, everything in this file will be executed.
- `pkg` is a **state module** for doing fun things with Linux packages. [List of state modules](http://docs.saltstack.com/en/latest/ref/states/all/index.html).
  - For convenience, `pkg` assumes that the parent key (`vim` here) is an installable package name. If it isn't, you'll need `-name: foo`.
- The `salt://` protocol is just the filesystem based at `/srv/salt`.
- Lines 2 and 3 can be short-handed into just `pkg.installed`. This is a YAMLism.

This is a formula. An aside:

Use the formula name with the function `state.sls`, viz. `sudo salt '*' state.sls vim`.

It's a gotcha so a sidebar:

### SLS File Namespace

    The namespace for SLS files follows a few simple rules:

    1. The .sls is discarded (i.e. webserver.sls becomes webserver).

    2. Subdirectories can be used for better organization.
      - Each subdirectory is represented by a dot.
      - webserver/dev.sls is referred to as webserver.dev.

    3. A file called init.sls in a subdirectory is referred to by the path of the directory. So, webserver/init.sls is referred to as webserver.

    4. If both webserver.sls and webserver/init.sls happen to exist, webserver/init.sls will be ignored and webserver.sls will be the file referred to as webserver.

## 2. Single formula in a directory

It's tidier to put formulas and their data files into a directory. To keep the clean formula name, in this case the SLS needs to be named `init.sls`.

"When an SLS formula is named init.sls it inherits the name of the directory path that contains it."

We also need to update our `source:` line to `source: salt://vim/vimrc`.

The formula is referenced by the formula name in `init.sls`. This should match the directory name (CONFIRM WHAT HAPPENS).

## 3. Multiple formulae in a directory

You can choose **not** to have an `init.sls` and define a bunch of different .sls files. These formulae are addressed with dirname.formula.

So you could have `edit.vim` referring to `edit/vim.sls` and `edit.emacs` referring to `edit/emacs.sls`.

## Other

You can apply multiple states with `sudo salt '*' state.sls vim,dev_tools,foo`.

# Doing things with packages

## Group a bunch of packages

dev_tools:
  pkg:
    - installed
    - pkgs:
      vim
      nodejs

Notice that when a `pkgs` key is present, the `pkg` module won't try and use the ID Declaration (`dev_tools`) as a package name.

## YAML

YAML is keys and values. Keys are in a hierarchy.

[YAML quick reference](http://www.yaml.org/refcard.html).

# Manage your /srv/salt as a git repo called 'salt'

# Requisites: like asserts

# Service: running

# Troubleshooting

My minion isn't responding. Seems common, even test.ping.
- Increase timeout - definitely
- How to check jobs?
  - job results are cached for 24hrs!
  - On master, `salt-run jobs.list_jobs`, get the job ID, `salt-run jobs.lookup_jid <id>`

## Debug minion

Run `salt-call` on the minion to see the debug logs for executing a given function. This is more info than the stream pushed back to the master.

# Good practices

Start developing on a flat foo.sls and only put in a directory when you need it.

# Is Salt good?

Easy to get up and running.

Do we believe this?:
"Two major philosophies exist on the subject, to either execute in an imperative fashion where things are executed in the order in which they are defined, or in a declarative fashion where dependencies need to be mapped between objects.
Imperative ordering is finite and generally considered easier to write, but declarative ordering is much more powerful and flexible but generally considered more difficult to create.
Salt has been created to get the best of both worlds. States are evaluated in a finite order, which guarantees that states are always executed in the same order, and the states runtime is declarative, making Salt fully aware of dependencies via the requisite system."

The 'pkg' shorthand where it takes the parent key (?) as name is confusing when you try and infer a model from it.

## Weirdness

http://steveko.wordpress.com/2014/02/17/one-week-of-salt-frustrations-and-reflections/

### Gotchas

The default shell is /bin/sh, not /bin/bash.

### States are not modules

But the docs look alike. pkg.installed is a state, but pkg.refresh_db is a module.

You will just get `State pkg.list_upgrades found in sls upgrade_packages is unavailable`, which isn't too helpful.

Running salt can do different things:
  1. Reach a state, e.g. with a state.sls file 
  1. Run a module.

Don't get them confused. You can't put a module function in a state.

### YAML will get you.

1. Look at:

    vim:
      pkg:
        - installed
        - pkgs:
          - foo
          - bar

Why does pkgs need a dash but pkg does not?

2. Special characters in strings. Good luck setting your cmd.run name: to [[ ! -f /var/run/reboot-required.pkgs ]] || cat /var/run/reboot-required.pkgs

3. Need an empty list (for example, match 'salt*' in the highstate but don't do anything)? You need:
  my_key: []

### Require

What can you require? pkg: foo, ok. What else? What does this really do?

Why are requires specified LAST in all the examples?

### Which goes first, module function or parameter?

What is this crap?

    apt-get autoremove -y:
      - cmd.run

## Issues

No apt-get autoremove, https://github.com/saltstack/salt/issues/15529

# Misc

syndic - a proxy that runs alongside master X, accepting commands from a different master Y.

# IRC log

[15:48] == Aaron42 [6c1d77b4@gateway/web/freenode/ip.108.29.119.180] has joined #salt
[15:50] <Aaron42> hello, newbie Salt learner here looking for assistance - any time appreciated. Anyone know why pkg.refresh_db is "unavailable" when I try to call it in an SLS?
[15:51] <viq> Aaron42: on what platform?
[15:52] <Aaron42> Ubuntu 14, latest
[15:53] <viq> do you have python-apt installed?
[15:53] <Aaron42> if I run salt '*' pkg.refresh_db it works fine. Just when I use an SLS it chokes.
[15:53] <Aaron42> so I assume that means the right packages are there.
[15:54] <viq> oh
[15:54] <viq> Because modules != states
[15:54] <viq> http://docs.saltstack.com/en/latest/ref/states/all/salt.states.pkg.html
[15:55] <viq> or you could use http://docs.saltstack.com/en/latest/ref/states/all/salt.states.cmd.html
[15:56] <Aaron42> OK I see. The modules are the underlying functionality used to reach a state
[15:56] <viq> yeah
[15:57] <Aaron42> I think the one I want is pkg.uptodate - but there's not a lot of detail in the doc to say what it does
[15:57] <Aaron42> I presume equivalent to pkg.refresh_db and pkg.upgrade
[15:58] <viq> Aaron42: also it's not available in stable versions yet
[15:59] <viq> Actually seems just pkg.upgrade, to have pkg.refresh_db you need to set refresh=True
[15:59] <Aaron42> ah interesting
[15:59] <viq> See that "New in version 2014.7.0" ? As you see in topic, there is Release Candidate of it, but it's not officially out yet
[16:00] <Aaron42> got it. My goal is just to update the system, seems a pretty core use case
[16:00] <viq> salt \* pkg.upgrade refresh=True
[16:02] <viq> Why from SLS?
[16:04] <timoguin> Aaron42: pkg.latest in an SLS should run a refresh first as well.
[16:05] <Aaron42> well my thinking is that a goal when setting up a new server is to update the packages, so it will go in top.sls.
[16:06] <viq> But do you want to upgrade all packages every time you're running highstate?
[16:06] <timoguin> I think of upgrades as more of an ad-hoc thing
[16:06] <timoguin> what viq said.
[16:07] <timoguin> if you put it in an sls you can specifify that SLS to run on minion startup every time
[16:07] <timoguin> probably better outside of highstate if you want to run highstate regularly
[16:07] <Aaron42> ok, maybe not in highstate then. Goal would be a standard set of states that get run once on a new server - update packages, install users, etc.
[16:07] <babilen> Aaron42: You will get the latest packages with pkg.installed if you set "refresh=True", but then I'd just use pkg.install and refresh explicitly.
[16:08] <Aaron42> this is to catch OS upgrades and the like
[16:08] <babilen> Aaron42: I really wouldn't make an upgrade part of the highstate. You want to do that explictly and controlled not whenever you run a highstate.
[16:08] <timoguin> Aaron42: if you really want to call pkg.refresh_db and pkg.upgrade in an SLS you can use module.run to call them.
[16:09] <timoguin> You just can't call them directly from the sls because they're not state modules
[16:09] <Aaron42> ok that's interesting
[16:09] <timoguin> That applies to any non-state module you want to run.
[16:09] <Aaron42> it seems like the new pkg.uptodate state does exactly what I want - just looking for the best way to do it before it ships.
[16:09] <viq> Aaron42: why run them only once, instead of constantly ensuring, that server is in a known desired state, part of which is making sure certain users and packages are present or not?
[16:10] <viq> Aaron42: also, I believe you can tell kickstart/preseed to upgrade packages upon installation
[16:11] <babilen> I'd think of highstate as "invariants that will be ensured to always be true" and upgrades are really more of an explicit change of configuration. Your highstate should, IMHO, be a no-op if you didn't change any states and changes that are due to a changing environment (e.g. package upgrades) should be performed planned an manually.
[16:12] <babilen> viq: You definitely can make it part of a preseeded Debian (et. al) install
[16:12] <babilen> s/an/and
[16:12] <Aaron42> right - I'm still learning e.g. exactly when highstate is run, so not completely on top of the philosophy of where things go.
[16:13] <timoguin> basically whenever you tell it to run. with no configuration, it won't run unless you call it.
[16:13] <timoguin> if you set startup_states: highstate, it'll run on minion startup
[16:14] <babilen> Sure, a highstate is run manually or can be run every k minutes/hours/days on a set schedule. The point is that you don't want it to really change anything unless you made an explicit change.
[16:14] <timoguin> Aaron42: salt-cloud can also run highstate or a set of SLS files after firing up new minion
[16:14] <timoguin> so that's a kind of out-of-band, only ran when the machine is created
[16:14] <Aaron42> I see, that would be useful. I've got a DigitalOcean image that connects to the master automatically at startup, so going to highstate would be good
[16:14] <timoguin> yea salt-cloud works very well with DO
[16:16] <Aaron42> yep I set up something by hand as a learning exercise. In case anyone's interested, http://www.aaronbell.com/lets-make-salt-minions-on-digitalocean/
[16:16] <Aaron42> thanks for all the info, much appreciated!
[16:21] <babilen> Enjoy your experience and feel free to ask any questions. (although I'll head off now as I have to work tomorrow)
