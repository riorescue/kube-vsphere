# Intro
This is an ongoing configuration guide to deploy kubernetes on vSphere leveraging the vSphere cloud provider container storage interface. This is a work in progress.

## Version History
| Version          | Description |
| ---------------- | ----------- |
| 0.1 (02/17/2020) | Initial     |

# Prerequisites
You will need a linux desktop to proceed with many of the items in this configuration guide. You can also leverage MacOS with <code>brew</code> to perform the same.

# Build Virtual Machine Template
We will be using the Ubuntu cloud image as a reference image. By default this ova leverages cloud-init which we will be disabling and instead using VMware vSphere VM Customization Specifications.