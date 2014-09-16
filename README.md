Requirements:

- DONE Be able to spin up a new droplet and have it automatically in config management.
- Be able to spin up a new droplet and have it ready to go with Docker and my preferred env.
- Take an action across all Docker droplets.
- Chromebook (and other clients like vagrant on Windows) register with config master for env updates. SECURITY!

# How we want to initiate new servers

    # apt-get update
    # apt-get dist-upgrade -y
    # reboot
    # apt-get upgrade -y
    # apt-get autoremove -y

# Salt concepts

Grain - a grain of information about a minion. Sent from minion -> master. Used for targeting commands.
Pillar - a tree of data kept on the master. Sent from master -> minions on a need-to-know basis. A State can have parameters filled in with data taken from a Pillar.
Nodegroup - a logical group of minions defined by a bunch of selectors. Used for targeting commands.
State - 

# Set up master State

Ever used `/srv` before? It's where you keep data for services, as defined in http://www.tldp.org/LDP/Linux-Filesystem-Hierarchy/html/srv.html

[Reference for file.managed](http://salt.readthedocs.org/en/latest/ref/states/all/salt.states.file.html). You will be using this **a lot**.

## IDEMPOTENCY

You define the target state. The tool will try to get there. If it's already there, you can keep running the tool and nothing bad will happen.

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

This is a formula. The naming of the formula is the `vim:` line. The filename could be anything.

Use the formula name with the function `state.sls`, viz. `sudo salt '*' state.sls vim`.

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

Run `salt-call` on the minion to see the debug logs for executing a given function. This is more info than the stream pushed back to the master.


# Misc

syndic - a proxy that runs alongside master X, accepting commands from a different master Y.