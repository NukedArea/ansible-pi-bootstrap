# ansible-pi-bootstrap version 1.0.0

This ansible project allows to bootstrap a Raspberry Pi OS or Rocky Linux. That
means that everything needed to run ansible on the managed node is installed
and a specific ansible user is created with a random password. It must be and
can be used only once on each Raspberry you want to manage with ansible.

At the end, the default user password is set with a random value. The only
authentication method is the public key.

It is assumed your ansible deployment machine is running Linux or macOS,
because Windows is not supported for the control node.

## First, create a SSH key on your local machine

The SSH key authenticates you on your Raspberry Pi OS or Rocky Linux in a
secure way. If you already have one, you can skip this section.

    mkdir -p ${HOME}/.ssh/
    chmod 0700 ${HOME}/.ssh/
    # use whatever your like as comment string '-C'
    ssh-keygen -t rsa -b 8192 -C "my@mydomain.tld - ansible deployment authentication" -f ${HOME}/.ssh/ansible_key
    # type your passphrase twice, don't use blank one

Keep your private key file in a safe place!

## Install your operating system

You will need [Balena Etcher](https://www.balena.io/etcher/) to flash your SD card.

### Rocky Linux

Download the [Rocky Linux for Raspberry
Pi](https://rockylinux.org/alternative-images). As the time of writing, the
available version is 8.5. Use Balena Etcher to flash the image on your SD card.

That's the only thing you have to do with Rocky Linux for Raspberry Pi.

Boot your Raspberry Pi, find your IP address and check the password
(rockylinux) is working by connecting with SSH: `ssh
rocky@your.raspberrypi.hostname.or.ip`.

### Raspberry Pi OS

Download the [Raspberry Pi OS
image](https://www.raspberrypi.com/software/operating-systems/), choose the
Lite version that correspond to your hardware. At the time of writing, the
available Raspberry Pi OS is bullseye version. Use Balena Etcher to flash the
image on your SD card.

Remount your SD card and find the mount point, on macOS, it will be
`/Volumes/boot/`, on Linux it should be something starting with `/mnt` or
`/media`.

This entire section must be repeated on each Raspberry Pi you want to manage
with ansible.

#### Enable SSH at boot on Raspberry Pi OS

Just create a `ssh` file in the `boot` partition.

    # example, on macOS, find yours, if you are on Linux
    touch /Volumes/boot/ssh

#### Create a pi user, with a temporary password on Raspberry Pi OS

Create a `pi` user with a temporary password by creating a `userconf` file in
the `boot` partition with the specified content.

    # if your want a different temporary password, generate one with openssl
    # (on macOS, you may need to install another openssl version with
    # https://brew.sh/).

    echo 'mYp@SSw0rd' | openssl passwd -6 -stdin
    # $6$sqk2GfpigUa6wK8Z$yDqGDLdi3AqhlpIV4p04f.Inczdlr2blx3fnArRUbgotaJSUVfVkhuJv0988hII8j/mk1IqvChPEukFHZtd4j.

    # put the generated password after the pi: string.
    echo $'pi:$6$sqk2GfpigUa6wK8Z$yDqGDLdi3AqhlpIV4p04f.Inczdlr2blx3fnArRUbgotaJSUVfVkhuJv0988hII8j/mk1IqvChPEukFHZtd4j.' > /Volumes/boot/userconf

Boot your Raspberry Pi, find your IP address and check the password is working
by connecting with SSH: `ssh pi@your.raspberrypi.hostname.or.ip`.

## Install python packages from requirements.txt file

On the ansible manager machine, create a python virtual environment
([python}](https://www.python.org/downloads/) 3.8 or newer). This step only
needs to be done once. You may use [pyenv](https://github.com/pyenv/pyenv) or
what you prefer.

    # using pyenv

    # install latest python version
    pyenv install 3.10.4

    # create a virtualenv named ansible-pi from this installed version
    pyenv virtualenv 3.10.4 ansible-pi

    # go to the directory where you cloned this repository and enable the
    # ansible-pi virtualenv in this directory
    pyenv local ansible-pi

Then, install the requirements with pip.

    pip install -r requirements.txt

## Create ansible hosts file

Just copy `inventory/hosts.sample` to `inventory/hosts` (warning, there is a
`.gitignore` rule on this filename) and edit its content or copy it to another
location.

## Set environment variables

You will have to export some environment variables directly in you shell or you
may use [direnv](https://github.com/direnv/direnv).

Set `ANSIBLE_INVENTORY` to point to your inventory file according to your
choice above.

    export ANSIBLE_INVENTORY=${PWD}/inventory/hosts

Set path to your public SSH keys. The public keys will be installed on the
ansible user `authorized_keys` file. If you want to install more than one
public keys, just separate them with a colon character `:`. Warning, this
process is exclusive. Old public keys already present in the `authorized_keys`
file will be dropped.

    # export only one key
    export SSH_PUBKEYS_TO_INSTALL=${HOME}/.ssh/ansible_key.pub

    # export more than one key
    export SSH_PUBKEYS_TO_INSTALL=${HOME}/.ssh/ansible_key.pub:${HOME}/.ssh/another_key.pub

## Test the ansible connection

Playbook must be run in two steps, using subsets. The defined password will be
prompted.

    ansible raspberrypios -m ping -u pi --ask-pass

    ansible rockylinux -m ping -u rocky --ask-pass

## Bootstrap time

Now, boostrap by using the bootstrap playbook. Playbook must be run in two
steps, using subsets.

    ansible-playbook bootstrap.yml --limit raspberrypios --ask-pass

    ansible-playbook bootstrap.yml --limit rockylinux --ask-pass --ask-become-pass

That's done. Raspberry Pi have been boostrapped. Now, you can login through
SSH, using your private key and become root via `sudo` command without
password.

    ssh -i $HOME/.ssh/ansible_key ansible@your.raspberrypi.hostname.or.ip

    # become root
    sudo -i

## Caveats

You may encounter problems with packages manager if your system date is wrong.
To set the date on all hosts, you may run these ad hoc commands. The new date
will be approximate, because `date +%s` command is run at the beginning of the
playbook, but this will be enough to fix the date problem for now.

    # to fix date on raspberrypios hosts (asked password is the one you set when flashing SD card)
    ansible raspberrypios -u pi -b --ask-pass -m ansible.builtin.raw -a "date +%s -s @$(date +%s)"

    # to fix date on rockylinux hosts (asked password is rockylinux)
    ansible rockylinux -u rocky -b --ask-pass --ask-become-pass -m ansible.builtin.raw -a "date +%s -s @$(date +%s)"

Then, run the bootstrap section again.

## Development

This repository uses [pre-commit](https://pre-commit.com/) hooks and
[ansible-lint](https://ansible-lint.readthedocs.io/). Install the requirements
with pip.

    pip install -r requirements.dev.txt

Then install hooks.

    pre-commit install

To lint, just run this command or do a commit.

    pre-commit run --all-files
