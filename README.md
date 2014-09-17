Requirements:

- DONE Be able to spin up a new droplet and have it automatically in config management.
- Be able to spin up a new droplet and have it ready to go with Docker and my preferred env.
- Take an action across all Docker droplets.
- Chromebook (and other clients like vagrant on Windows) register with config master for env updates. SECURITY!

# Next:
http://docs.saltstack.com/en/latest/topics/tutorials/starting_states.html
http://docs.saltstack.com/en/latest/ref/states/ordering.html
add YAML parsing rule for .sls files to vimrc on config master
do install.sh in order

## My topfile:
vim
terminal_tools
pkg.list_upgrades (look at output)
upgrade_packages
autoremove_packages
reboot_required (look at output)
system.reboot (if needed)

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

Do we believe this?:
"Two major philosophies exist on the subject, to either execute in an imperative fashion where things are executed in the order in which they are defined, or in a declarative fashion where dependencies need to be mapped between objects.
Imperative ordering is finite and generally considered easier to write, but declarative ordering is much more powerful and flexible but generally considered more difficult to create.
Salt has been created to get the best of both worlds. States are evaluated in a finite order, which guarantees that states are always executed in the same order, and the states runtime is declarative, making Salt fully aware of dependencies via the requisite system."

The 'pkg' shorthand where it takes the parent key (?) as name is confusing when you try and infer a model from it.

## Weirdness

### Gotchas

The default shell is /bin/sh, not /bin/bash.

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