# Automated Netskope Publisher Deployment using Cloud-Init
**Version: 0.2.0**

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

# How do I use this?
1. [Access the configuration file here](https://github.com/nathancatania/publisher-cloud-init/blob/main/user-data.yaml) and copy the text. Note where it says to paste in the Publisher enrollment token.
2. Access your Netskope Admin Console and create a new Publisher. Generate an enrollment token and paste it into the appropriate field in the configuration file.
3. Create an Ubuntu VM within your IaaS environment. This was validated on Ubuntu Server 20.04 LTS.
4. During the VM creation workflow, paste in the configuration text (including your Publisher enrollment token) into a field marked as cloud-init/custom-data/user-data. This is different for every platform (see specific instructions below).
5. Wait a few minutes. After some time, you should see the Publisher be marked as `Connected` in the Netskope Admin Console.
6. If you specified an SSH key in the config, you should now be able to SSH to the Publisher VM using the `npa` user. Eg: `ssh -i <key> npa@<publisher-ip>`

## AWS / EC2
* For AWS EC2, when creating a VM instance, scroll down and under **Advanced Details**, simply paste the configuration into the **User-Data**.
* Don't worry about explicitly setting an SSH identity/key when configuring the VM details - the configuration data can do this for you under the `npa` user.
* Don't forget to add the Publisher enrollment key.

![aws](https://i.imgur.com/vc7POtl.png)

## Azure
* For Azure, when creating a VM instance, select the **Advanced** tab, and paste the configuration into the **Custom data** field under the **Custom data and cloud init** sub-heading.
* Don't worry about explicitly setting an SSH identity/key when configuring the VM details - the configuration data can do this for you under the `npa` user.
* Don't forget to add the Publisher enrollment key.
![azure](https://i.imgur.com/2GetPUq.png)

## GCP
* For Google Cloud, when creating a VM instance, open the **Management** tab, and add a new item under **Metadata**.
* Set the **key** to be `user-data` exactly! For the value, paste in the configuration.
* Don't forget to add the Publisher enrollment key.
![gcp](https://i.imgur.com/6YLrF9f.png)

# Who are you?
I'm a Netskope Solutions Engineer that created this to make everyone's lives easier! This script is NOT officially supported by Netskope, and by using it, **you agree to use it at your own risk**. Additionally, any of my own thoughts posted here are my own and do not reflect those of Netskope.

# Validation / Troubleshooting Steps
If you're not sure whether this has worked properly, do the following:

## Connect to the VM
SSH to the VM using the `npa` user, eg: `ssh -i <ssh-key> npa@<vm-ipaddress>`

* Note: You'll only be able to SSH to the VM if you also specified an SSH key or identity in the configuration.
  
## Check the status of cloud-init
Check the status of the cloud-init service: `cloud-init status`

```
npa@i-097b1b0c65d4ef774:~$ cloud-init status
status: done
```

If the status shows `status: running`, cloud-init is still configuring the VM. Either wait a few more minutes or proceed below.

## Check the cloud-init logs

  * `/var/log/cloud-init-output.log` is the live output as the configuration is being applied. Very useful to observe this file if your cloud-init status is still `status: running`. You can also run `sudo tail -f /var/log/cloud-init-output.log` if cloud-init is still running to see the live output as this log is written to.
  * `/var/log/cloud-init.log` is the log of the overarching cloud-init process itself.
 
```
npa@i-097b1b0c65d4ef774:~$ sudo cat /var/log/cloud-init-output.log
[...snip...]
Reading package lists...
Building dependency tree...
Reading state information...
The following packages will be REMOVED:
  libfwupdplugin1
0 upgraded, 0 newly installed, 1 to remove and 0 not upgraded.
After this operation, 459 kB disk space will be freed.
(Reading database ... 92928 files and directories currently installed.)
Removing libfwupdplugin1:amd64 (1.5.11-0ubuntu1~20.04.2) ...
Processing triggers for libc-bin (2.31-0ubuntu9.7) ...
Registering with your Netskope address: ns-[REDACTED].us-sjc1.npa.goskope.com
Publisher certificate CN: [REDACTED]
Attempt 1 to register publisher.
Publisher registered successfully.

Verifying connectivity to the Netskope Dataplane...
Connectivity to the Netskope Dataplane was successfully verified.
Cloud-init v. 21.4-0ubuntu1~20.04.1 running 'modules:final' at Tue, 26 Apr 2022 01:57:49 +0000. Up 18.20 seconds.
Cloud-init v. 21.4-0ubuntu1~20.04.1 finished at Tue, 26 Apr 2022 02:01:43 +0000. Datasource DataSourceEc2Local.  Up 252.19 seconds
```

## Check the applied configuration
`/var/lib/cloud/instance/user-data.txt` is a copy of the configuration template you pasted in during VM creation. You can see what this looks like in a rendered/finished state (ie: the *exact* configuration that cloud-init applies) through the following command: `sudo cloud-init devel render /var/lib/cloud/instance/user-data.txt`

Unrendered template example:
```
npa@i-097b1b0c65d4ef774:~$ sudo cat /var/lib/cloud/instance/user-data.txt
 - systemctl restart ssh
 - 'apt autoremove -y'
 - 'apt-get clean -y'
{% if (token != "TOKEN") or (not token) %}
 - "/home/npa/npa_publisher_wizard -token {{ token }}"
{% endif %}
```

Rendered template example:
```
npa@i-097b1b0c65d4ef774:~$ sudo cloud-init devel render /var/lib/cloud/instance/user-data.txt
[...snip...]
 - systemctl restart ssh
 - 'apt autoremove -y'
 - 'apt-get clean -y'
 - "/home/npa/npa_publisher_wizard -token eyJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJucy0xMzY1Ni51cy1zamMxLm5w[REDACTED]fGXxg"
```

---

# TODO
* ~~Add logic to apply authorized SSH keys if present.~~
* Simplify the existing `runcmd` commands to use native cloud-init modules where possible.
* Possibly simplify even further to pull most of the logic/commands from a remote script (hosted by Netskope) to execute - similar to how the existing bootstrap script works today.
* ~~Test on AWS (Azure and GCP are done and working)~~
* Update documentation to add more detailed instructions (and screenshots) for ~~AWS, Azure, GCP~~, and DO.

# Known Issues
* The NPA Publisher Wizard does not auto-start when you SSH to the VM. Currently it can be manaully run by running the command: `sudo /home/npa/npa_publisher_wizard`
* Logic to grab SSH Public Key from GitHub might not be working. Need to test further, but recommend you manually define an SSH key using the 2nd method/option for now.
* Note an issue per-se, but when SSH'ing to the VM, you will need to SSH as the `npa` default user. Eg: `ssh npa@publisher-ip`
