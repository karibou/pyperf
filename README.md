# pyperf

# Introduction

This set of script run performance tests on locally built python 2.7.6 and 2.7.12 interpreters

# Building the interpreters

This was done on an up to date Xenial host

## Installing the build environment

    $ sudo -s
    # apt install devscripts ubuntu-dev-tools sbuild lxc
    # sudo adduser <username> sbuild
    # sbuild-update --keygen
    # cat - <<EOM >~/.mk-sbuild.rc 
    # mk-sbuild build tunables -- SOURCE_CHROOTS_TGZ used with 'file' and SOURCE_CHROOTS_DIR with 'directory'
    SCHROOT_CONF_SUFFIX="source-root-users=root,sbuild,admin
    source-root-groups=root,sbuild,admin
    preserve-environment=true"
    SKIP_UPDATES="1"
    DEBOOTSTRAP_INCLUDE="devscripts"
    EOM

## Create the build environments
    # mk-sbuild --arch=amd64 --distro=ubuntu trusty
    # mk-sbuild --arch=amd64 --distro=ubuntu --name=trusty-updates trusty
    # mk-sbuild --arch=amd64 --distro=ubuntu xenial

The trusty schroots will be setup with GCC 4.8.2 which is the version used to 
build the package in the archive. The schroot called trusty-updates will be 
updated to use the latest GCC 4.8.4 to compare both compilers.
    

# Build the Python interpreters

## Build with the original GCC 4.8.2 compiler
    # schroot -c source:trusty-amd64 -u root
    (trusty-amd64) mkdir trusty
    (trusty-amd64) export LC_ALL="C"
    (trusty-amd64) cd trusty
    (trusty-amd64) dget http://archive.ubuntu.com/ubuntu/pool/main/p/python2.7/python2.7_2.7.6-8ubuntu0.3.dsc
    (trusty-amd64) dpkg-source -x \*.dsc
    (trusty-amd64) cd python2.7-2.7.6/
    (trusty-amd64) DEB_BUILD_OPTIONS=parallel=$(nproc) DEB_BUILD_OPTIONS="nocheck nobench" fakeroot debian/rules build
    (trusty-amd64) cd
    (trusty-amd64) mkdir xenial
    (trusty-amd64) cd xenial
    (trusty-amd64) dget http://archive.ubuntu.com/ubuntu/pool/main/p/python2.7/python2.7_2.7.12-1ubuntu0~16.04.1.dsc
    (trusty-amd64) dpkg-source -x \*.dsc
    (trusty-amd64) cd python2.7-2.7.12
    (trusty-amd64) DEB_BUILD_OPTIONS=parallel=$(nproc) DEB_BUILD_OPTIONS="nocheck nobench" fakeroot debian/rules build


## Build with the updated GCC 4.8.4 compiler
    # schroot -c source:trusty-updates-amd64 -u root
    # apt update
    # apt install gcc

The trusty and xenial builds method uses the same commands as listed above.

## Build with Xenial's GCC 5.3 compiler
    # schroot -c source:xenial-amd64 -u root

The trusty and xenial builds method uses the same commands as listed above.

# Copy the rebuilt interpreters

    # mkdir ~/orig_gcc
    # sudo cp -pr /var/lib/schroot/chroots/trusty-amd64/home/ubuntu/* ~/orig_gcc/trusty_build
    # sudo cp -pr /var/lib/schroot/chroots/xenial-amd64/home/ubuntu/* ~/orig_gcc/xenial_build
    # mkdir ~/update_gcc
    # sudo cp -pr /var/lib/schroot/chroots/trusty-updates-amd64/home/ubuntu/* ~/update_gcc/trusty_build

The final directory structure should be :

    orig_gcc
    orig_gcc/trusty_build/trusty/python2.7-2.7.6
    orig_gcc/trusty_build/xenial/python2.7-2.7.12
    orig_gcc/xenial_build/trusty/python2.7-2.7.6
    orig_gcc/xenial_build/xenial/python2.7-2.7.12
    update_gcc
    update_gcc/trusty_build/trusty/python2.7-2.7.6
    update_gcc/trusty_build/xenial/python2.7-2.7.12


# Setting up the virtualenv

The pyperformance module relies on virtualenv to run test performance tests.
Though there is a *-p* option, it will only instruct pyperformance to use this
specified interperter to run the pyperformance script, but not to use this
interpreter to run the performance tests.

So we will create two reference virtualenv which will be used to create specific
virtualenv for each performance test run. This specific virtualenv will have its
interpreter replaced by the locally built one. This is to work around a limitation
where virtualenv expects the interpreter's prefix to be /usr which is not our case.

We alson need to take into account that the *datetime* module is **not** a builtin
in version 2.7.6 of python.

## Building the reference virtualenv

The 2.7.6 reference virtualenv needs to be built in a *Trusty* environment so
we will use an LXC container for that :

    # lxc launch ubuntu-daily:t trusty-venv
    # lxc list
    # ssh ubuntu@192.168.100.142

    In a trusty lxc container :

    $ sudo -s
    # apt install python-pip python-virtualenv python-dev
    # python -mpip install performance
    # pyperformance run -p=python2 --bench=2to3 --debug -o /tmp/template.json
    # mv venv/cpython* venv/venv_template_276

On the Xenial host:

    # rsync -av ubuntu@{xenial container}:venv .
    # pyperformance run -p=python2 --bench=2to3 --debug -o /tmp/template.json
    # mv venv/cpython* venv/venv_template_2712

The virtualenv templates are now ready to be used.

# Prepare the environments

Simply run the script *prepare_venv* included in this repository. Here is what it does :

1. Cleanup previous virtual environment
2. Copy the appropriate template into a specific venv
3. If version 2.7.6, copy the datetime.so module
4. Fix the VIRTUAL_ENV path to the specific venv name
5. Copy the locally built interpreter in the venv
6. Replace the python2.7 symlink

# Run the tests

The script *run_tests* will run all the tests sequentially. You can use the
*--test* switch to verify that your environment is correctly setup.

    # mkdir perf
    # ./run_tests
