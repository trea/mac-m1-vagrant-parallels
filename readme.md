# Vagrant Mac M1 Setup

**IMPORTANT**: **This setup assumes that the operating system and software you want to run in a VM are able to run under ARM. There is no CPU architecture emulation, etc. here.**

VirtualBox doesn't run on M1 Macs. This setup will have you running a Linux virtual machine guest via Vagrant and the Parallels virtualization provider.

If your team is using a vanilla Vagrant box and then provisions that box through Vagrant's provisioning, this could work for you. You may however find that some software packages your provisioners install aren't available for ARM64. 

My PHP projects' dependencies (php, php-redis, php-mysql, etc.) were all available directly through apt. I have since added xdebug which wasn't available in apt by downloading the source code and compiling it inside of the VM.

If your team is building and provisioning a base box and then your members are importing that box, **following this is almost definitely a bad idea**.

## Requirements

- [Vagrant](https://vagrantup.com)
    - ```brew install vagrant```
    - Vagrant doesn't ship an ARM binary yet, you'll be prompted to setup Rosetta 2 when you run Vagrant the first time.
- [Parallels Desktop](https://www.parallels.com/)
    - Trial does work in this setup if you don't want to pay up front
- Parallels Provider for Vagrant
    - ```vagrant plugin install vagrant-parallels```
- [Packer](https://www.packer.io/downloads)
    - ```brew tap hashicorp/tap && brew install hashicorp/tap/packer```
- [Parallels Virtualization SDK](https://www.parallels.com/products/desktop/download/#AdditionalDownloads)

## Vagrant Boxes

A key piece of making Vagrant work is the [box](https://www.vagrantup.com/docs/boxes) which is the packaging format for Vagrant. Historically you could pretty well count on finding an existing box created by trustworthy vendor.

I see some advertised boxes for M1 on [Vagrant Cloud](https://app.vagrantup.com/boxes/search), but none fit the following requirements I had:

1. Ubuntu Linux distribution
1. Compatible with M1 Mac ARM64/Aarch64 architecture
1. Well known publisher (hashicorp, chef bento, ubuntu, etc.)

## Build a Ubuntu Box with Packer

The nice folks over at [Chef](https://www.chef.io) keep and curate Packer templates for a bunch of Linux distributions. You should clone the repository:

```
git clone https://github.com/chef/bento
```

Support for ARM is new in Bento: the template I'm using came from this pull request: https://github.com/chef/bento/pull/1374

>If you want to use a different Linux distribution, you could potentially create a Packer template or modify an existing template to point to ARM64 compatible versions of your desired distribution ISO.

Next we need to be in the chef/bento repository we cloned and we'll run Packer to build us the box.

```
cd bento/packer_templates/ubuntu
packer build --only=parallels-iso ubuntu-20.04-arm64.json
```

When the build is complete you should see something along the lines of:

```
==> Wait completed after 6 minutes 24 seconds

==> Builds finished. The artifacts of successful builds are:
--> parallels-iso: 'parallels' provider box: ../../builds/ubuntu-20.04-live.parallels.box
```

Locate our built .box file:

```
cd ../../builds
ls
```

Then import that box into Vagrant:

```
vagrant box add --name=ubuntu-20.04 ubuntu-20.04-live.parallels.box
```

## Using the New Box in an Existing Project

First, you'll need to alter the `Vagrantfile` to have a section for your Parallels provider where `override.vm.box` will refer to the name you provided in the `vagrant box add` step:

```rb
  config.vm.provider :parallels do |prl, override|
    override.vm.box = "ubuntu-20.04"
    prl.memory = 2048
    prl.cpus = 2
  end        
```

>More provider specific documentation is here: https://parallels.github.io/vagrant-parallels/docs/configuration.html 


Depending on how you provision your Vagrant boxes, you might have more work to do. I lucked out and the shell provisioning scripts in my project already worked as-is.

Run `vagrant up` and good luck!


## Using the New Box on a New Project

Just like if you were starting a new project with a public Vagrant box, you'll call `vagrant init` but you'll refer to `ubuntu-20.04` instead of a public box:

```
vagrant init ubuntu-20.04
```

You can optionally tweak the new `Vagrantfile` to set the amount of memory and CPU available or make other configuration changes:

```rb
  config.vm.provider :parallels do |prl, override|
    override.vm.box = "ubuntu-20.04"
    prl.memory = 2048
    prl.cpus = 2
  end        
```

Now would be a good time to figure out your Vagrant VM provisioning as well: https://www.vagrantup.com/docs/provisioning

Last but not least, start your new VM:

```
vagrant up
```