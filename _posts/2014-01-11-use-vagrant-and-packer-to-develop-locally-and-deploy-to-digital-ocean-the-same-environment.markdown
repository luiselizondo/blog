---
layout: post
title: Use Vagrant and Packer to develop locally and deploy to Digital Ocean the same
  environment
date: '2014-01-11 19:39:07'
tags:
- digitalocean
- packer
- vagrant
- virtualbox
---

<p>I've been using Vagrant and Packer.io for the last week and I'm really excited of the new possibilities it opens for developers and how it changes the way we develop, test and deploy. There's not a lot of tutorials on how to integrate Vagrant and Packer so I'll try to explain everything from the beginning.</p><p>Vagrant and Packer.io</p><p>Both Vagrant and Packer.io were created by the same user and both are really cool. Vagrant is just a software that sits on top of a VM software (Virtualbox, VMWare, etc) to build and manage environments. What this means is no matter what VM software you're using, you configure Vagrant throught template files to <strong>manage</strong> the box and always have the same environment. This is really useful since the same box (think of a pre-configured ISO with your Operating System of choice) can be used in development, testing and production. All your libraries, all the software you install on your OS to run that app configured the same way for developement, testing and production. P<span style="line-height: 1.6em;">acker is another beauty, it allows you to create a preconfigured box.</span></p><p>If you got lost in the last paragraph, keep reading, maybe an example of what I'm trying to do will clarify things a bit.</p><p>I have a Node.js app, among the software I need to run the app are:</p><ul><li>node.js</li><li>forever (a node.js module)</li><li>nginx</li><li>ImageMagick</li></ul><p>In an imperfect world, I would need to install the same versions for each of those packages in my development, testing and production environment. I would also need to have the same configuration for each environment. Any difference that I'm not aware of, can cause problems and does not guarantee consistensy across environments. I need the same OS image everywhere, configured the same way, running the same software and on different locations, my development environment runs locally and both testing/stage and production run on Digital Ocean.</p><p>So how to solve this riddle? Easy. With Packer and some scripts we can create a .box image of our operating system preconfigured for both VirtualBox and DigitalOcean, both images wil have little to no difference, and finally with Vagrant we can manage those boxes.</p><p>To use Packer.io I had to fork an existing example on Github by, but that didn't work because it was build for an older version of Packer, so I had to change a few things. You can get the files I'll be working on from <a href="https://github.com/luiselizondo/packer-example">https://github.com/luiselizondo/packer-example</a></p><p>In packer.json we define properties that packer will take to build the box, in the packer directory I have bash scripts that will be executed by packer when building the box, you have commands to install software that you need your box to have; you'll see a preseed.cfg file too, this file will be used by packer to send instructions to the installer. The Vagrantfile will be loaded everytime you provision a box and you can use it to configure your network or even to run scripts that need to be run.</p><p>Let's analyse the packer.json file:</p><p> </p>
<code>
{
  "provisioners": [
    {
      "type": "shell",
      "scripts": [
        "packer/base.sh",
        "packer/vagrant.sh",
        "packer/virtualbox.sh",
        "packer/cleanup.sh",
        "packer/zerodisk.sh"
      ],
      "override": {
        "virtualbox-iso": {
          "execute_command": "echo 'vagrant'|sudo -S sh '{{.Path}}'"
        }
      }
    }
  ],
  "builders": [
    {
      "type": "virtualbox-iso",
      "boot_command": [
        "<esc><esc><enter><wait>",
        "/install/vmlinuz noapic preseed/url=http://{{ .HTTPIP }}:{{ .HTTPPort }}/preseed.cfg <wait>",
        "debian-installer=en_US auto locale=en_US kbd-chooser/method=us <wait>",
        "hostname=vagrant <wait>",
        "fb=false debconf/frontend=noninteractive <wait>",
        "keyboard-configuration/modelcode=SKIP keyboard-configuration/layout=USA keyboard-configuration/variant=USA console-setup/ask_detect=false <wait>",
        "initrd=/install/initrd.gz -- <enter><wait>"
      ],
      "boot_wait": "4s",
      "disk_size": 10140,
      "guest_os_type": "Ubuntu_64",
      "http_directory": "packer",
      "iso_checksum": "2cbe868812a871242cdcdd8f2fd6feb9",
      "iso_checksum_type": "md5",
      "iso_url": "http://releases.ubuntu.com/12.04/ubuntu-12.04.3-server-amd64.iso",
      "ssh_username": "vagrant",
      "ssh_password": "vagrant",
      "ssh_port": 22,
      "ssh_wait_timeout": "10m",
      "shutdown_command": "echo 'shutdown -P now' > shutdown.sh; echo 'vagrant'|sudo -S sh 'shutdown.sh'",
      "guest_additions_path": "VBoxGuestAdditions_{{.Version}}.iso",
      "virtualbox_version_file": ".vbox_version",
      "vboxmanage": [
        [
          "modifyvm",
          "{{.Name}}",
          "--memory",
          "512"
        ],
        [
          "modifyvm",
          "{{.Name}}",
          "--cpus",
          "2"
        ]
      ]
    },
    {
      "type": "digitalocean",
      "api_key": "YOUR_API_KEY",
      "client_id": "YOUR_CLIENT_ID",
      "private_networking": false,
      "snapshot_name": "mybox-{{timestamp}}",
      "ssh_username": "vagrant"
    }
  ],
  "post-processors": ["vagrant"]
}
</code>

<p>This is just a JSON file with some things Packer needs. Provisioners are tasks that will execute after the installer finishes but before the image gets created, meaning that if you install Apache and configure it, your image will come with Apache installed and configured. This tasks can be shell scripts or you can use Puppet/Chef/Salt if you want.</p>

<p>On the Builders section you specify what kind of image you want to create, in my case, I need two boxes, one to use it with VirtualBox and another one living on Digital Ocean. We also set some properties for each builder that are unique depending on the type of Builder</p>

<p>Finally, in the Post Processors section you specify if you'll manage this using Vagrant or vSphere</p>

To run Packer, just do

<code>
packer build packer.json
</code>

<p>This command will create a file for you to use with Virtualbox on your local machine and a new Digital Ocean box and we'll use vagrant to manage both.</p>

To use vagrant, you'll need to add the new file to the list of boxes vagrant can use. To do it just do:

<code>
vagrant box add mycoolbox file_packer_created.box
</code>

You can see a list of boxes vagrant knows about by issuing the command:

<code>
vagrant box list
</code>

<p>
The Vagrantfile is also easy to read, you set configurations for each provider or general configurations that will be used for all providers. Since the Vagrantfile accepts comments, I'll use the same file to explain each line
</p>

<code>
# -*- mode: ruby -*-
# vi: set ft=ruby :

# Vagrantfile API/syntax version. Don't touch unless you know what you're doing!
VAGRANTFILE_API_VERSION = "2"

# Tell vagrant to use the Version 2 of the API
Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  
  # Vagrant will use "mycoolbox" to boot the machine, you can choose any box you've added or listed with the vagrant box list command
  config.vm.box = "mycoolbox"
  # Run this script each time we provision this box
  config.vm.provision :shell, :path => "vagrant/bootstrap-stage.sh"
  # Forward the port 80 on the VM machine, to the port 8080 on the host machine
  config.vm.network :forwarded_port, host: 8080, guest: 80
  # Run this command too
  config.vm.provision :shell, :inline => "/etc/init.d/networking restart"

  # Forward the ssh agent
  config.ssh.forward_agent = true

  # The path to my ssh key
  config.ssh.private_key_path = "~/.ssh/id_rsa"
  
  config.vm.provider :digital_ocean do |provider, override|    
    override.ssh.private_key_path = '~/.ssh/id_rsa'
    override.vm.box = 'digital_ocean'
    override.vm.box_url = "https://github.com/smdahlen/vagrant-digitalocean/raw/master/box/digital_ocean.box"

    provider.image = "Ubuntu 12.04.3 x64"
    provider.client_id = 'YOUR_CLIENT_ID'
    provider.api_key = 'YOUR_API_KEY'
  end
end
</code>

<p>To run your new box, just run <strong>vagrant up</strong> and you'll be able to ssh using <strong>vagrant ssh</strong></p>

<h3>Integrate everything</h3>
I added the packer and vagrant files to my application source code, this way, any developer can build the same machine exactly to use it on Virtualbox and Digital ocean. The box will have a symlink to between /vagrant and /var/www. This is important since /vagrant is a shared folder between the guest and the host machine.

<h3>Disclaimer</h3>
Remember that digital ocean will charge you for each box you create, it's cheap and only charges by hour but if you forget to turn off your machine I will not be responsible and most importantly I will not help you pay your bill. Amazon Web Services offers a Free tier plan that you can use for free for a limited amount of hours.



