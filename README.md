# OVERVIEW

This is minitrav, a bare-bones, minimal implementation of the Travis-CI worker.

I really like the whole Travis-CI environment, but there are times when
I want to run the tests without pushing to GitHub and I don't need all the
reporting overhead of Travis-CI. This tool does just enough of the CI worker
part of Travis-CI to get the tests done.

Basically, I have a Ubuntu VM with the CPAN modules I need
installed locally using perlbrew and cpanm. The minitrav script
clones the repo and branch I specify into a temporary build
directory and runs the tests in a way similar to the Travis-CI.

In fact, this uses the .travis.yml configuration file to decide
what to do. Please note, though, that only a subset of the keywords
in .travis.yml have been implemented. This is work-in-progress, so
look at the source code when using this for your project.

Also, I should note that this only works for Perl projects.

# SETUP

To use this with the current project I am mainly working on, OpenXPKI,
I started with a fresh Ubuntu Server 12 installation in VMWare and ran 
the following steps.

## Install Ubuntu

Follow the VMWare instructions, specifying 512MB RAM, 4GB HD, no USB or
sound card).

## Install Additional Packages

For convenience, I use SSH to access the guest OS. In addition, the
build tools (gcc, make), as well as curl and git will be needed for
further steps.

    $ sudo aptitude install -y openssh-server make gcc curl git

## Add Hostname for Development Workstation

I do my development work on the Macbook where I'm running VMWare, so
in order for the guest OS to find my local Git repository, I add an
entry to the guest /etc/hosts, replacing *YOUR\_WS\_IP\_ADDRESS* and
*YOUR\_WS\_HOSTNAME* as appropriate:

    $ echo "YOUR_WS_IP_ADDRESS YOUR_WS_HOSTNAME" | sudo tee -a /etc/hosts

By the way, if you are wondering why I use *tee* rather than just file 
redirection, it is because you can prepend it with *sudo* to append to
system files.

## Add Public SSH Key to Guest

This is pure convenience, but very helpful during development.

    $ mkdir $HOME/.ssh
    $ tee -a $HOME/.ssh/authorized_keys <<EOF
    INSERT_YOUR_SSH_PUBLIC_KEY_HERE
    EOF

## Install minitrav

    $ git clone https://github.com/mrscotty/minitrav.git

## Install perlbrew

Note: you must logout and login again after this step.

    $ curl -kL http://install.perlbrew.pl | bash
    $ echo "source ~/perl5/perlbrew/etc/bashrc" | tee -a $HOME/.profile

## Install Local Perl

As of this writing, perl-5.16.1 seemed like the newest choice. To check,
just run 'perlbrew available' to see all the choices. Also, the YAML is
installed here so that we can read the .travis.yml.

    $ perlbrew install perl-5.16.1
    $ perlbrew switch perl-5.16.1
    curl -L http://cpanmin.us | perl - App::cpanminus
    cpanm YAML

## Wrap Up

At this point, the minitrav can be used to install remaining prereqs and then
run the tests. To save yourself the trouble of running the above steps repeatedly,
I suggest either cloning the new guest VM or taking a snapshot to be able
to revert to the clean installation.

# USAGE

Now that the prerequisites for minitrav are installed, there are currently just
a few tasks that can be run: before\_install, install and test.

Both of these commands will clone your repository to a temporary directory
with the prefix $HOME/build-. This directory is *not* automatically removed
by *minitrav*. Also, the *.travis.yml* file from your repository is read to
determine the actual commands to be run.

Note: the Travis-CI worker actually runs all the preinstallation steps and
the tests *each* time it is run. Since I needed a bit more granularity
at the moment, I chose to run each block manually.

## Before Install

This is used to install Ubuntu packages that will be required for the
CPAN dependencies, etc. The commands listed in the *before\_install*
section of *.travis.yml* are run.

    $ minitrav/bin/minitrav before_install mymacbook:git/openxpki feature/my-current-work

## Install

This is used to install the CPAN dependencies. The commands listed in 
the *install* section of *.travis.yml* are run.

    $ minitrav/bin/minitrav install mymacbook:git/openxpki feature/my-current-work

This should only need to be run once. In contrast to the Travis-CI worker,
which is a fresh install *each* time, I don't usually re-install things
each time since there are so many prerequisites for OpenXPKI.

## Test

This task reads the *.travis.yml* file from your git repository and runs the
commands defined there for testing your software. Currently, only the 
*script* configuration keyword is supported. If this is not defined, *minitrav*
assumes that your project is a Perl project and looks for Build.PL and
Makefile.PL. As in Travis-CI, it will also fall back to "make test", which 
happens to be what OpenXPKI uses.

    $ minitrav/bin/minitrav test mymacbook:git/openxpki feature/my-current-work


