# Automated Netskope Publisher Deployment using Cloud-Init
**Version: 0.1.0**

## What is this?
This is a User-Data configuration file for Cloud-Init that will automatically deploy, configure and enrol a Netskope Publisher in AWS, Azure or GCP; eliminating the need for any manual configuration from the command-line.

This means that all you need to do is deploy a standard Ubuntu VM in any of these providers, and as long as this configuration is pasted into the VM config during creation, the Netskope Publisher will be downloaded, installed, and enrolled on the host as part of the VM deployment process.

## What is Cloud-Init?
[Cloud-Init](https://cloudinit.readthedocs.io/en/latest/index.html) is a piece of software included out of the box in most common Linux Distros (eg: Ubuntu, RHEL), and is used to automatically provision initialize required configuration on first boot/deployment of a VM.

On instance boot, cloud-init will identify the cloud it is running on, read any provided metadata from the cloud, and initialize the system accordingly. This may involve setting up the network and storage devices, configuring SSH access keys, and setting up many other aspects of a system. Later, cloud-init will parse and process any optional user or vendor data that was passed to the instance.

## Why is this useful?
The premise of ZTNA is that you provide access to internal resources and environments to validated users from anywhere, without requiring any inbound network exposure or connectivity (like with a VPN). Using this Cloud-Init configuration, you can deploy a Netskope Publisher into AWS/Azure/GCP without requiring inbound (SSH) access to access to configure the Publisher itself.

This means that in a closed off IaaS environment (as long as you have outbound access on TCP 443 towards the Netskope cloud), you can simply deploy an Ubuntu VM for the publisher to use, and have instant access to internal resources in that environment as soon as the deployment job for the VM is complete.

## What does the script/configuration do?
Essentially it adapts the existing Netskope bootstrapping script into something that works within Cloud-Init.

Normally to bootstrap and deploy a publisher on a fresh VM, you run the following from the command-line on the host VM:
```
curl https://s3-us-west-2.amazonaws.com/publisher.netskope.com/latest/generic/bootstrap.sh | sudo bash; sudo su - $USER; exit
```
This script (and the two child scripts that it calls):
1. Update software on the host
2. Install required packages for the Publisher (such as Docker)
3. Pulls the Publisher image from DockerHub.
4. Sets the Publisher container and management wizard to run on boot.
5. Applies firewall rules to the host as required by the Publisher
6. Hardens the host VM
7. Cleans up files from the installation process.

The admin then needs to manually apply the enrollment token/key to register the Publisher so that it can start forwarding traffic.

This Cloud-Init configuration actions the above steps from the bootstrapping script, in addition to the following:
* Installs the Netskope Root CA in case you wish to SSL inspect traffic coming from the VM using the Netskope service.
 * Note: Be sure to bypass the NPA domains if you do this as the publisher tunnel back to Netskope is certificate pinned and designed to break if SSL inspected.
* Applies the specified enrollment token/key to the publisher so that the Publisher can be used straight away with no further configuration required.
* [OPTIONAL/RECOMMENDED] If specified, applies a list of authorized public SSH keys so that you can actually connect to the host VM to manage the publisher.

## How do I use this?
1. [Access the configuration file here](https://github.com/nathancatania/publisher-cloud-init/blob/main/user-data.yaml) and copy the text. Note where it says to paste in the Publisher enrollment token.
2. Access your Netskope Admin Console and create a new Publisher. Generate an enrollment token and paste it into the appropriate field in the configuration file.
3. Create an Ubuntu VM within your IaaS environment. This was validated on Ubuntu Server 20.04 LTS.
4. During the VM creation workflow, paste in the configuration text (including your Publisher enrollment token) into a field marked as cloud-init/custom-data/user-data. This is different for every platform (see specific instructions below (TODO)).
5. Wait a few minutes. After some time, you should see the Publisher be marked as `Connected` in the Netskope Admin Console.

## Who are you?
I'm a Netskope Solutions Engineer that created this to make everyone's lives easier! This script is NOT officially supported by Netskope however (by using it, you agree to use it at your own risk). Additionally, any of my own thoughts are my own and do not reflect those of Netskope.

# TODO
* Add logic to apply authorized SSH keys if present.
* Simplify the existing `runcmd` commands to use native cloud-init modules where possible.
* Possibly simplify even further to pull most of the logic/commands from a remote script (hosted by Netskope) to execute - similar to how the existing bootstrap script works today.
* Test on AWS (Azure and GCP are done and working)
* Update documentation to add more detailed instructions (and screenshots) for AWS, Azure, GCP, and DO.

# Known Issues
* The NPA Publisher Wizard does not auto-start when you SSH to the VM. Currently it can be manaully run by running the command: `sudo /home/npa/npa_publisher_wizard`
* Note an issue per-se, but when SSH'ing to the VM, you will need to SSH as the `npa` default user. Eg: `ssh npa@publisher-ip`
