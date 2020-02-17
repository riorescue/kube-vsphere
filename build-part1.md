# Kubernetes with vSphere Integration (Part 1)
This is an ongoing configuration guide to deploy kubernetes on vSphere leveraging the vSphere cloud provider container storage interface. This is a work in progress.

## Version History
| Version          | Description                                |
| ---------------- | ------------------------------------------ |
| 0.1 (02/17/2020) | Initial (Kubernetes 1.17.3, Docker 19.3.6) |

## Prerequisites
You will need a linux desktop to proceed with many of the items in this configuration guide. You can also leverage MacOS with `brew` to perform the same.

## Build Virtual Machine Template
### Install and Configure govc
We will be using the Ubuntu Server 18.04 LTS (Bionic Beaver) official Cloud Image as a reference to create our template virtual machine. By default this OVA leverages cloud-init which we will be disabling and instead using VMware vSphere VM Customization Specifications (created later).

https://cloud-images.ubuntu.com/releases/bionic/release/ubuntu-18.04-server-cloudimg-amd64.ova

After downloading the OVA above we will need to use `govc` to connect to vCenter and create the template and VM Customization Specifications that will eventually be used to deploy kubernetes control-plane and worker nodes.

Install `govc` from github: https://github.com/vmware/govmomi/tree/master/govc

`govc` will require specific configuration paramters to connect and authenticate with vCenter. Instead of providing this information everytime you want to connect to vCenter we can use a file and export the parameters into our current shell. I am creating a file called `govcparms.sh` to store the required parameters. Execute the following to create the file and edit as needed to fit your vCenter environment.

```shell
sudo tee ${PWD}/govcparms.sh >/dev/null <<EOF
export GOVC_INSECURE=1 # Don't verify SSL certs on vCenter
export GOVC_URL=10.0.200.41 # vCenter IP/FQDN
export GOVC_USERNAME=administrator@vsphere.local # vCenter username
export GOVC_PASSWORD=P@ssw0rd110 # vCenter password
export GOVC_DATASTORE=SATA # Default datastore to deploy to
export GOVC_NETWORK="vlan200-labnet" # Default network to deploy to
export GOVC_RESOURCE_POOL='*/Resources' # Default resource pool to deploy to
EOF
```

Next, load the above variables into your current shell:

```shell
source ${PWD}/govcparms.sh
```

We should now be able to test if `govc` is able to connect to vCenter:

```shell
$ govc about
Name:         VMware vCenter Server
Vendor:       VMware, Inc.
Version:      6.7.0
Build:        15129973
OS type:      linux-x64
API type:     VirtualCenter
API version:  6.7.3
Product ID:   vpx
UUID:         2666c667-84f8-42b4-b2ce-10e12fb9c63a
```

### Extract and Customize the OVF Specification
Using `govc` extract the OVF specification file from the Ubuntu OVA (downloaded earlier). This will create the spec file to the file in your present directory called `ubuntuspec.json`:

```shell
govc import.spec ~/Downloads/ubuntu-18.04-server-cloudimg-amd64.ova | python3 -m json.tool > ubuntuspec.json
```

> Note: You may need to modify the above command to meet the requirements of your installed python version. In this case I am running python3 only but you can change the command to "python -m json" if you are running python2.x.

Edit the `ubuntuspec.json` file updating the following fields:
* Change `DiskProvisioning` from "flat" to "thin"
* Change `hostname` value to `Ubuntu1804CloudTemplate`
* Change `public-keys` value to (your SSH public key) (see below for example)
* Change `password` to a *temporary* value `VMware9!` (you will be required to change this later)
* Change `VM Network` to your specified network portgroup name. In my case it's `vlan200-labnet`
* Change `name` from null to the same value you used for `hostname` above

Below is an example of my configuration:

```json
{
    "DiskProvisioning": "thin",
    "IPAllocationPolicy": "dhcpPolicy",
    "IPProtocol": "IPv4",
    "PropertyMapping": [
        {
            "Key": "instance-id",
            "Value": "id-ovf"
        },
        {
            "Key": "hostname",
            "Value": "Ubuntu1804CloudTemplate"
        },
        {
            "Key": "seedfrom",
            "Value": ""
        },
        {
            "Key": "public-keys",
            "Value": "ssh-rsa AAAAB3NzaC1yc2EA[truncated] lnxcfg@ubuntu1804cloudtemplate"
        },
        {
            "Key": "user-data",
            "Value": ""
        },
        {
            "Key": "password",
            "Value": "VMware9!"
        }
    ],
    "NetworkMapping": [
        {
            "Name": "VM Network",
            "Network": "vlan200-labnet"
        }
    ],
    "MarkAsTemplate": false,
    "PowerOn": false,
    "InjectOvfEnv": false,
    "WaitForIP": false,
    "Name": "Ubuntu1804CloudTemplate"
}
```

### Deploy the OVA
We will now deploy the Ubuntu cloud image OVA to vCenter using our customized OVF specification (above). You will pass the `ubuntuspec.json` file as an option to the `govc` command:

```shell
govc import.ova -options=ubuntuspec.json ~/Downloads/ubuntu-18.04-server-cloudimg-amd64.ova
```

Modify the VM hardware properties changing the VM configuration to **4 vCPU, 4GB of RAM, and 60GB hard disk**. Recall we configured thin provisioned disks so all 60GB will not be used. Also, kubernetes vSphere integration requires that `disk.enableUUID=1` be set on all VMware-based virtual machines. Finally, power on the virtual machine to proceed to post-configuration steps:

```shell
govc vm.change -vm Ubuntu1804CloudTemplate -c 4 -m 4096 -e="disk.enableUUID=1"
govc vm.disk.change -vm Ubuntu1804CloudTemplate -disk.label "Hard disk 1" -size 60G
govc vm.power -on=true Ubuntu1804CloudTemplate
```

### Prepare the VM for Templating
To proceed, you will need to obtain the Ubuntu cloud  VM's IP address to that you can SSH to it. This can either be done through vCenter or using `govc` at the command-line:

```shell
$ watch -n 10 govc vm.info Ubuntu1804CloudTemplate
Name:           Ubuntu1804CloudTemplate
  Path:         /Datacenter/vm/Ubuntu1804CloudTemplate
  UUID:         4238b5ba-ac62-6d0a-1ac7-c4764e8274be
  Guest name:   Ubuntu Linux (64-bit)
  Memory:       4096MB
  CPU:          4 vCPU(s)
  Power state:  poweredOn
  Boot time:    2020-02-17 17:18:55.320752 +0000 UTC
  IP address:   10.0.200.109
  Host:         esx.lab.vstable.com
```

SSH to the new virtual machine. You will be logged in automatically through key-based authentication because you included your SSH public key in the `ubuntuspec.json` file ealier. You will be required to change the password of the `ubuntu` account at first login (do not try to re-use the previous password). Recall the *current password* was set in the ubuntuspec.json file and is **VMware9!**

```shell
$ ssh ubuntu@10.0.200.109
```

> After changing the password for user `ubuntu` you will be automatically logged out. 

SSH back to the virtual machine and update using apt:

```shell
ssh ubuntu@10.0.200.109
sudo apt update
sudo apt install open-vm-tools -y
sudo apt upgrade -y
sudo apt autoremove -y
```

Because we are no longer going to use the in-built/configured `cloud-init` system we are going to cleanup and disable it:

```shell
sudo cloud-init clean --logs
sudo touch /etc/cloud/cloud-init.disabled
sudo rm -rf /etc/netplan/50-cloud-init.yaml
sudo apt purge cloud-init -y
sudo apt autoremove -y
```

Configure systemd to start open-vm-tools which will be used with the VMware Guest Customization process:

```shell
sudo sed -i 's/D \/tmp 1777 root root -/#D \/tmp 1777 root root -/g' /usr/lib/tmpfiles.d/tmp.conf
sudo sed -i 's/Before=cloud-init-local.service/After=dbus.service/g' /lib/systemd/system/open-vm-tools.service
```

Perform pre-templating cleanup (*provided by blah.cloud*):

```shell
# cleanup current ssh keys so templated VMs get fresh key
sudo rm -f /etc/ssh/ssh_host_*

# add check for ssh keys on reboot...regenerate if neccessary
sudo tee /etc/rc.local >/dev/null <<EOL
#!/bin/sh -e
#
# rc.local
#
# This script is executed at the end of each multiuser runlevel.
# Make sure that the script will "" on success or any other
# value on error.
#
# In order to enable or disable this script just change the execution
# bits.
#

# By default this script does nothing.
test -f /etc/ssh/ssh_host_dsa_key || dpkg-reconfigure openssh-server
exit 0
EOL

# make the script executable
sudo chmod +x /etc/rc.local

# cleanup apt
sudo apt clean

# reset the machine-id (DHCP leases in 18.04 are generated based on this... not MAC...)
echo "" | sudo tee /etc/machine-id >/dev/null

# disable swap for K8s
sudo swapoff --all
sudo sed -ri '/\sswap\s/s/^#?/#/' /etc/fstab

# cleanup shell history and shutdown for templating
history -c
history -w
sudo shutdown -h now
```

Convert the virtual machine to a template:

```shell
govc vm.markastemplate Ubuntu1804CloudTemplate
```

### Create VMware Guest Customization Specification in vCenter
Unfortunately, `govc` cannot currently be used to create VMware Guest Customization Specifications and therefore you either need to do this manually or use PowerShell. I will leverage PowerShell because it's easier to document:

> Linux or MacOS can now run PowerShell and Windows is not a requirement to proceed. The easiest way to get PowerShell on Unbutu is via a Snap.

```$ sudo snap install powershell –classic```

Once PowerShell is installed you can start if from the command-line:

```$ pwsh```

Install the VMware PowerCLI (if you haven't already done so previously):

```powershell
> Install-Module -Name VMware.PowerCLI
```

Connect to your Virtual Center server:

```powershell
> Connect-VIServer 10.0.200.41 -User administrator@vsphere.local -Password P@ssw0rd110
```

A successfull connection should return the following:
```shell
Name                           Port  User
----                           ----  ----
10.0.200.41                    443   VSPHERE.LOCAL\Administrator
```

Create a new VMware Guest Customization Specification. Note that you may specify multiple DNS servers separating each with a comma (ex. 1.1.1.1,2.2.2.2):
```powershell
New-OSCustomizationSpec -Name Ubuntu1804 -OSType Linux -DnsServer 10.0.200.1 -DnsSuffix lab.vstable.com -Domain lab.vstable.com -NamingScheme vm
```

> If your vCenter is using self-signed certifcates (common in lab scenarios) you may need to configure PowerCLI to ignore SSL certificate warnings (see below)
```powershell
> Set-PowerCLIConfiguration -InvalidCertificateAction Ignore -Confirm:$false
```

### Deploy Virtual Machines for Control-Plane and Worker Nodes
Deploy new virtual machines to be used later in this tutorial. We will be deploying a single control-plane virtual machine with 3 worker nodes. These deployments will leverage the template vm and VMware Customization Specification created above.

```shell
govc vm.clone -vm Ubuntu1804CloudTemplate -customization=Ubuntu1804  k8s-master
govc vm.clone -vm Ubuntu1804CloudTemplate -customization=Ubuntu1804  k8s-worker1
govc vm.clone -vm Ubuntu1804CloudTemplate -customization=Ubuntu1804  k8s-worker2
govc vm.clone -vm Ubuntu1804CloudTemplate -customization=Ubuntu1804  k8s-worker3
```

> Be sure to exit the PowerShell (pwsh) command-shell prior to executing the above commands using `govc`

Once all 4 virtual machines have been deployed you can leverage `govc` to check the IP addresses of each machine:

```shell
$ watch -n 10 "govc find / -type m -name 'k8s*' | xargs govc vm.info | grep 'Name:\|IP'"

Name:           k8s-master
  IP address:   10.0.200.123
Name:           k8s-worker3
  IP address:   10.0.200.135
Name:           k8s-worker1
  IP address:   10.0.200.207
Name:           k8s-worker2
  IP address:   10.0.200.129
```

### Cleanup vCenter Inventory
Create a new Virtual Machine folder named `k8s` to store your Kubernetes virtual machine nodes. Be sure to replace `Datacenter` below with the name of **your** vCenter Datacenter.

```shell
govc folder.create /Datacenter/vm/k8s
govc object.mv /Datacenter/vm/k8s-\* /Datacenter/vm/k8s
```

### Continue to Part 2
We are now ready to configure general requirements for our Kubernetes build.
[Continue to part 2](build-part2.md)

> Optional: You may want to take snapshots of each of the four virtual machines if this is your first time configuring Kubernetes. This will allow a more easy rollback and start if something goes wrong.
