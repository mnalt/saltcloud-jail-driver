# saltcloud-iocage-jail-driver
Saltcloud driver for using FreeBSD jails with iocage

## Introduction

This is a driver for managing FreeBSD jails with
[salt-cloud](https://docs.saltstack.com/en/latest/topics/cloud/index.html),
using the `jail` command and `jail.conf` configuration.

It allows you to manage jails on any salt minion. It supports all of the
default salt-cloud functionality. In particular the bootstrap and deployment
scripts will be run in the newly created jail (unless `--no-deploy` is given -
see salt-cloud documentation).

The jails' root directories can be populated by copying or `zfs clone`ing from
a configured list of directories or zfs snapshots. 

Since jails can be accessed from the host, minion keys, configuration and
deployment scripts necessary for bootstrapping are not copied by ssh but using
the salt minion on the host instead. This means that you do not need ssh access
to your newly created jails.


## Installation

Put the `fbsd_jail.py` file in a directory that saltstack will search for cloud drivers. For example, put

    extension_modules: /usr/local/etc/salt/extension_modules

in your `master` configuration file and put the `fbsd_jail.py` file in the directory `/usr/local/etc/salt/extension_modules/clouds`.



## Configuration

### Provider

Sample provider configuration for a provider named `iocage-example`:

    jail-example:
      host_list:
        - host1.example.com
	      - always.use.complete.minion.id.com
      ignore_host_list:
        - dontuse.example.com
      jail_parent: /usr/local/jails
      images:
        directories:
          - /usr/local/jails/templates/10.0-RELEASE
          - /usr/local/jails/templates/11.0-RELEASE
        snapshots:
          - zpool/root/jails/templates/10.0-RELEASE@p5
          - zpool/root/jails/templates/11.0-RELEASE@p10
        scripts:
          - /usr/local/sbin/create_jail_root.sh
      driver: fbsd_jail

The following options are available:
 * `host_list`: lists all minions that should be accessed by salt-cloud. If
   this parameter is not set, all available minions will be used where the
   command `iocage list` can successfully be executed. Always specify the
   complete minion id (usually the fqdn)
 * `ignore_host_list`: remove the named hosts from the list of hosts that will
   be used. Always specify the complete minion id. Only really makes sense if
   no `host_list` is provided (though you can provide both)
 * `jail_parent`: the parent directory for new jails
 * `images`: a list of directories, zfs snapshots or scripts, used to initialise a jail root directory
    * `directories`: when creating a jail from a directory, the content is `cp -a`'d to the target directory
    * `snapshots`:  are `zfs clone`'d and mounted at the target directory. A
      zfs volume must be mounted at `jail_parent` for this to work, and the
      clones will be children of that volume
    * `scripts`: are called with the target directory as a parameter and responsible for populating the jail root
 * `driver`: set to `fbsd_jail` in order to use this driver

### Profile and Command-Line Parameters

Sample profile configuration:

    profile1:
      provider: jail-example
      properties:
        ip4_addr: em0|192.168.1.42
      minion:
        master: 192.168.105.2
    profile2:
      provider: jail-example
      properties:
        ip4_addr: lo1|192.168.105.144
        exec_timeout: 120
      image: "release:9.3-RELEASE"
    profile3:
      provider: jail-example
        properties:
          ip4_addr: lo1|192.168.105.145
        image: "template:pre-saltet"

The profile configuration allows you to specify the following parameters:

 * `properties`: arbitrary jail.conf properties. In particular, you can set the
   `ip4_addr`.
 
 * `image`: This parameter is used to specify which image to use for creating a
   jail. Images are configured as part of the provider configuration but you
   can also list the available releases and templates using the `salt-cloud
   --list-images=fbsd_jail` command.

 * `location`: This specifies the host where you want to create the jail. Use
   the host's minion id.
   You have to specify the `location` if the list of available hosts (see
   `host_list` and `ignore_host_list` above) contains more then one host.

   Alternatively you can provide the location on the command-line using the
   `--location` parameter

In addition to those parameters, you can set most of the default salt-cloud
parameters for profiles, such as `minion` configuration, `script` and
`inline_script`, `file_map` etc (see salt-cloud documentation
[here](https://docs.saltstack.com/en/latest/topics/cloud/misc.html)).  Note
that the iocage driver uses the jail-host's minion to setup the jail, and does
not use ssh to communicate with the jail. The ssh relevant options therefore
have no effect.


Note that jails are identified by iocage tags only. The salt-cloud iocage
currently does not support the use of UUIDs.


