# Intro
This is an ongoing configuration guide to deploy kubernetes on vSphere leveraging the vSphere cloud provider container storage interface. This is a work in progress.

## Version History
| Version          | Description |
| ---------------- | ----------- |
| 0.1 (02/17/2020) | Initial     |

## Prerequisites
You will need a linux desktop to proceed with many of the items in this configuration guide. You can also leverage MacOS with `brew` to perform the same.

## Build Virtual Machine Template
We will be using the Ubuntu Server 18.04 LTS (Bionic Beaver) official Cloud Image as a reference to create our template virtual machine. By default this OVA leverages cloud-init which we will be disabling and instead using VMware vSphere VM Customization Specifications (created later).

https://cloud-images.ubuntu.com/releases/bionic/release/ubuntu-18.04-server-cloudimg-amd64.ova

After downloading the OVA above we will need to use `govc` to connect to vCenter and create the template and VM Customization Specifications that will eventually be used to deploy kubernetes control-place and worker nodes.

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

