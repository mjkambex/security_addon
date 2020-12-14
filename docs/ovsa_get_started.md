# OpenVINO™ Security Add-on  {#ovsa_get_started}

This guide provides instructions for people who use the OpenVINO™ Security Add-on to create, distribute, and use models that are created with the OpenVINO™ toolkit:

* **Model Developer**: The Model Developer interacts with the Independent Software Vendor to control the User access to models. This document shows you how to setup hardware and virtual machines to use the OpenVINO™ Security Add-on to define access control to your OpenVINO™ models and then provide the access controlled models to the users. 
* **Independent Software Vendor**: Use this guide for instructions to use the OpenVINO™ Security Add-on to validate license for access controlled models that are provided to your customers (users). 
* **User**: This document includes instructions for end users who need to access and run access controlled models through the OpenVINO™ Security Add-on.  

In this release, one person performs the role of both the Model Developer and the Independent Software Vendor. Therefore, this document provides instructions to configure one system for these two roles and one system for the User role. This document also provides a way for the same person to play the role of the Model Developer, Independent Software Vendor, and User to let you see how the OpenVINO™ Security Add-on functions from the User perspective.


## Overview

The OpenVINO™ Security Add-on works with the [OpenVINO™ Model Server](@ref openvino_docs_ovms) on Intel® architecture. Together, the OpenVINO™ Security Add-on and the OpenVINO™ Model Server provide a way for Model Developers and Independent Software Vendors to use secure packaging and secure model execution to enable access control to the OpenVINO™ models, and for model Users to run inference within assigned limits.

The OpenVINO™ Security Add-on consists of three components that run in Kernel-based Virtual Machines (KVMs). These components provide a way to run security-sensitive operations in an isolated environment. A brief description of the three components are as follows. Click each triangled line for more information about each. 

<details>
    <summary><strong>OpenVINO™ Security Add-on Tool</strong>: As a Model Developer or Independent Software Vendor, you use the OpenVINO™ Security Add-on Tool(`ovsatool`) to generate a access controlled model and master license. </summary>

- The Model Developer generates a access controlled model from the OpenVINO™ toolkit output. The access controlled model uses the model's Intermediate Representation (IR) files to create a access controlled output file archive that are distributed to Model Users. The Developer can also put the archive file in long-term storage or back it up without additional security. 

- The Model Developer uses the OpenVINO™ Security Add-on Tool(`ovsatool`) to generate and manage cryptographic keys and related collateral for the access controlled models. Cryptographic material is only available in a virtual machine (VM) environment. The OpenVINO™ Security Add-on key management system lets the Model Developer to get external Certificate Authorities to generate certificates to add to a key-store. 

- The Model Developer generates user-specific licenses in a JSON format file for the access controlled model. The Model Developer can define global or user-specific licenses and attach licensing policies to the licenses. For example, the Model Developer can add a time limit for a model or limit the number of times a user can run a model. 

</details>

<details>
    <summary><strong>OpenVINO™ Security Add-on License Service</strong>: Use the OpenVINO™ Security Add-on License Service to verify user parameters.</summary>

- The Independent Software Vendor hosts the OpenVINO™ Security Add-on License Service, which responds to license validation requests when a user attempts to load a access controlled model in a model server. The licenses are registered with the OpenVINO™ Security Add-on License Service.

- When a user loads the model, the OpenVINO™ Security Add-on Runtime contacts the License Service to make sure the license is valid and within the parameters that the Model Developer defined with the OpenVINO™ Security Add-on Tool(`ovsatool`). The user must be able to reach the Independent Software Vendor's License Service over the Internet. 

</details>

<details>
    <summary><strong>OpenVINO™ Security Add-on Runtime</strong>: Users install and use the OpenVINO™ Security Add-on Runtime on a virtual machine. </summary>

Users host the OpenVINO™ Security Add-on Runtime component in a virtual machine. 

Externally from the OpenVINO™ Security Add-on, the User adds the access controlled model to the OpenVINO™ Model Server config file. The OpenVINO™ Model Server attempts to load the model in memory. At this time, the OpenVINO™ Security Add-on Runtime component validates the user's license for the access controlled model against information stored in the License Service provided by the Independent Software Vendor. 

After the license is successfully validated, the OpenVINO™ Model Server loads the model and services the inference requests. 

</details> 

<br>
**Where the OpenVINO™ Security Add-on Fits into Model Development and Deployment**

![Security Add-on Diagram](ovsa_diagram.png)

## About the Installation
The Model Developer, Independent Software Vendor, and User each must prepare one physical hardware machine and one Kernel-based Virtual Machine (KVM). In addition, each person must prepare a Guest Virtual Machine (Guest VM) for each role that person plays. 

For example:
* If one person acts as both the Model Developer and as the Independent Software Vendor, that person must prepare two Guest VMs. Both Guest VMs can be on the same physical hardware (Host Machine) and under the same KVM on that Host Machine.
* If one person acts as all three roles, that person must prepare three Guest VMs. All three Guest VMs can be on the same Host Machine and under the same KVM on that Host Machine.

**Purpose of Each Machine**

| Machine      | Purpose |
| ----------- | ----------- |
| Host Machine      | Physical hardware on which the KVM and Guest VM share set up.      |
| Kernel-based Virtual Machine (KVM)    | The OpenVINO™ Security Add-on runs in this virtual machine because it provides an isolated environment for security sensitive operations.        |
|  Guest VM    |    The Model Developer uses the Guest VM to enable access control to the completed model. <br>The Independent Software Provider uses the Guest VM to host the License Service.<br>The User uses the Guest VM to contact the License Service and run the access controlled model.   |


## Prerequisites <a name="prerequisites"></a>

**Hardware**
* Intel® Core™ or Xeon® processor<br>

**Operating system, firmware, and software**
* Ubuntu* Linux* 18.04 on the Host Machine.<br>
* TPM version 2.0-conformant Discrete Trusted Platform Module (dTPM) or Firmware Trusted Platform Module (fTPM)
* Secure boot is enabled.<br>

**Other**
* The Independent Software Vendor must have access to a Certificate Authority (CA) that implements the Online Certificate Status Protocol (OCSP), supporting Elliptic Curve Cryptography (ECC) certificates for deployment.
* The example in this document uses self-signed certificates.

## How to Prepare a Host Machine <a name="setup-host"></a>

This section is for the combined role of Model Developer and Independent Software Vendor, and the separate User role.

### Step 1: Set up Packages on the Host Machine<a name="setup-packages"></a>

Begin this step on the Intel® Core™ or Xeon® processor machine that meets the <a href="#prerequisites">prerequisites</a>.

**NOTE**: As an alternative to manually following steps 1 - 11, you can run the script `install_host_deps.sh` in the `Scripts/reference directory` under the OpenVINO™ Security Add-on repository. The script stops with an error message if it identifies any issues. If the script halts due to an error, correct the issue that caused the error and restart the script. The script runs for several minutes and provides progress information.

1. Test for Trusted Platform Module (TPM) support:
   ```sh
   dmesg | grep -i TPM 
   ```	
   The output indicates TPM availability in the kernel boot logs. Look for presence of the following devices to indicate TPM support is available:
   * `/dev/tpm0`
   * `/dev/tpmrm0`
   
   If you do not see this information, your system does not meet the <a href="#prerequisites">prerequisites</a>  to use the OpenVINO™ Security Add-on.
2. Make sure hardware virtualization support is enabled in the BIOS:
   ```sh
   kvm-ok 
   ```
   The output should show: <br>
   `INFO: /dev/kvm exists` <br>
   `KVM acceleration can be used`
	
   If your output is different, modify your BIOS settings to enable hardware virtualization.
   
   If the `kvm-ok` command is not present, install it:
   ```sh
   sudo apt install -y cpu-checker
   ```
3. Install the Kernel-based Virtual Machine (KVM) and QEMU packages. 
	```sh	
	sudo apt install qemu qemu-kvm libvirt-bin  bridge-utils  virt-manager 
	```	
4. Check the QEMU version:
   ```sh	
   qemu-system-x86_64 --version 
   ```	
   If the response indicates a QEMU version lower than 2.12.0 download, compile and install the latest QEMU version from [https://www.qemu.org/download](https://www.qemu.org/download).
5.  Build and install the [`libtpm` package](https://github.com/stefanberger/libtpms/). 
6.  Build and install the [`swtpm` package](https://github.com/stefanberger/swtpm/).
7.  Add the `swtpm` package to the `$PATH` environment variable.
8.  Install the software tool [`tpm2-tss`]( https://github.com/tpm2-software/tpm2-tss/releases/download/2.4.4/tpm2-tss-2.4.4.tar.gz).<br>
    Installation information is at https://github.com/tpm2-software/tpm2-tss/blob/master/INSTALL.md
9.  Install the software tool [`tpm2-abmrd`](https://github.com/tpm2-software/tpm2-abrmd/releases/download/2.3.3/tpm2-abrmd-2.3.3.tar.gz).<br>
    Installation information is at https://github.com/tpm2-software/tpm2-abrmd/blob/master/INSTALL.md
10. Install the [`tpm2-tools`](https://github.com/tpm2-software/tpm2-tools/releases/download/4.3.0/tpm2-tools-4.3.0.tar.gz).<br>
    Installation information is at https://github.com/tpm2-software/tpm2-tools/blob/master/INSTALL.md
11. Install the [Docker packages](https://docs.docker.com/engine/install/ubuntu/).	

**NOTE**: Regardless of whether you used the `install_host_deps.sh` script, complete step 12 to finish setting up the packages on the Host Machine.

12. If you are running behind a proxy, [set up a proxy for Docker](https://docs.docker.com/config/daemon/systemd/). 

The following are installed and ready to use:
* Kernel-based Virtual Machine (KVM)
* QEMU
* SW-TPM
* HW-TPM support
* Docker<br>
	
You're ready to configure the Host Machine for networking. 

### Step 2: Set up Networking on the Host Machine<a name="setup-networking"></a>

This step is for the combined Model Developer and Independent Software Vendor roles. If Model User VM is running on different physical host, repeat the following steps for that host also.

In this step you prepare two network bridges:
* A global IP address that a KVM can access across the Internet. This is the address that the OpenVINO™ Security Add-on Run-time software on a user's machine uses to verify they have a valid license.
* A host-only local address to provide communication between the Guest VM and the QEMU host operating system.

This example in this step uses the following names. Your configuration might use different names:
* `50-cloud-init.yaml` as an example configuration file name.
* `eno1` as an example network interface name. 
* `br0` as an example bridge name.
* `virbr0` as an example bridge name.

1. Open the network configuration file for editing. This file is in `/etc/netplan` with a name like `50-cloud-init.yaml`
2. Look for these lines in the file:
   ```sh	
   network:
     ethernets:
        eno1:
          dhcp4: true
          dhcp-identifier: mac
     version: 2
   ```
3. Change the existing lines and add the `br0` network bridge. These changes enable external network access:
   ```sh	
   network:
     ethernets:
        eno1:
          dhcp4: false
     bridges:
        br0:
          interfaces: [eno1]
          dhcp4: yes
		  dhcp-identifier: mac
     version: 2
   ```
4. Save and close the network configuration file.
5. Run two commands to activate the updated network configuration file. If you use ssh, you might lose network connectivity when issuing these commands. If so, reconnect to the network.
   ```sh
   sudo netplan generate
   ```
   
   ```sh
   sudo netplan apply
   ```	
   A bridge is created and an IP address is assigned to the new bridge.
6. Verify the new bridge:
   ```sh
   ip a | grep br0
   ```	
   The output looks similar to this and shows valid IP addresses:
   ```sh	
   4: br0:<br><BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000<br>inet 123.123.123.123/<mask> brd 321.321.321.321 scope global dynamic br0
   ```	
7. Create a script named `br0-qemu-ifup` to bring up the `br0` interface. Add the following script contents:
   ```sh
   #!/bin/sh
   nic=$1
   if [ -f /etc/default/qemu-kvm ]; then
   	. /etc/default/qemu-kvm
   fi
   switch=br0
   ifconfig $nic 0.0.0.0 up
   brctl addif ${switch} $nic
   ```
8. Create a script named `br0-qemu-ifdown` to bring down the `br0` interface. Add the following script contents:
   ```sh
   #!/bin/sh
   nic=$1
   if [ -f /etc/default/qemu-kvm ]; then
   	. /etc/default/qemu-kvm
   fi
   switch=br0
   brctl delif $switch $nic
   ifconfig $nic 0.0.0.0 down
   ```
9. Create a script named `virbr0-qemu-ifup` to bring up the `virbr0` interface. Add the following script contents:
   ```sh
   #!/bin/sh
   nic=$1
   if [ -f /etc/default/qemu-kvm ]; then
   	. /etc/default/qemu-kvm
   fi
   switch=virbr0
   ifconfig $nic 0.0.0.0 up
   brctl addif ${switch} $nic
   ```
10. Create a script named `virbr0-qemu-ifdown` to bring down the `virbr0` interface. Add the following script contents:
	```sh
	#!/bin/sh
	nic=$1
	if [ -f /etc/default/qemu-kvm ]; then
	. /etc/default/qemu-kvm
	fi
	switch=virbr0
	brctl delif $switch $nic
	ifconfig $nic 0.0.0.0 down
	```

See the QEMU documentation for more information about the QEMU network configuration.

Networking is set up on the Host Machine. Continue to the Step 3 to prepare a Guest VM for the combined role of Model Developer and Independent Software Vendor.

	
### Step 3: Set Up one Guest VM for the combined roles of Model Developer and Independent Software Vendor<a name="dev-isv-vm"></a>

For each separate role you play, you must prepare a virtual machine, called a Guest VM. Because in this release, the Model Developer and Independent Software Vendor roles are combined, these instructions guide you to set up one Guest VM, named `ovsa_isv`.

Begin these steps on the Host Machine. 

As an option, you can use `virsh` and the virtual machine manager to create and bring up a Guest VM. See the `libvirtd` documentation for instructions if you'd like to do this.

1. Download the [Ubuntu 18.04 server ISO image](https://releases.ubuntu.com/18.04/ubuntu-18.04.5-live-server-amd64.iso)

2. Create an empty virtual disk image to serve as the Guest VM for your role as Model Developer and Independent Software Vendor:
   ```sh
   sudo qemu-img create -f qcow2 <path>/ovsa_isv_dev_vm_disk.qcow2 20G
   ```
3. Install Ubuntu 18.04 on the Guest VM. Name the Guest VM `ovsa_isv`:
	```sh
	sudo qemu-system-x86_64 -m 8192 -enable-kvm \
	-cpu host \
	-drive if=virtio,file=<path-to-disk-image>/ovsa_isv_dev_vm_disk.qcow2,cache=none \
	-cdrom <path-to-iso-image>/ubuntu-18.04.5-live-server-amd64.iso \
	-device e1000,netdev=hostnet1,mac=52:54:00:d1:66:5f \
	-netdev tap,id=hostnet1,script=<path-to-scripts>/virbr0-qemu-ifup,downscript=<path-to-scripts>/virbr0-qemu-ifdown \
	-vnc :1
	```
4. Connect a VNC client with `<host-ip-address>:1`
5. Follow the prompts on the screen to finish installing the Guest VM. Name the VM as `ovsa_isv_dev`
6. Shut down the Guest VM. 
7. Restart the Guest VM after removing the option of cdrom image:
   ```sh
   sudo qemu-system-x86_64 -m 8192 -enable-kvm \
   -cpu host \
   -drive if=virtio,file=<path-to-disk-image>/ovsa_isv_dev_vm_disk.qcow2,cache=none \
   -device e1000,netdev=hostnet1,mac=52:54:00:d1:66:5f \
   -netdev tap,id=hostnet1,script=<path-to-scripts>/virbr0-qemu-ifup,downscript=<path-to-scripts>/virbr0-qemu-ifdown \
   -vnc :1
   ```
8. Choose ONE of these options to install additional required software:
<details><summary>Option 1: Use a script to install additional software</summary>
	a. Copy the script `install_guest_deps.sh` from the Scripts/reference directory of the OVSA repository to the Guest VM<br>
	b. Run the script.<br>
	c. Shut down the Guest VM.<br><br>
	Click the triangled line to close Option 1
</details>

<details><summary>Option 2: Manually install additional software</summary>
	a.  Install the software tool [`tpm2-tss`](https://github.com/tpm2-software/tpm2-tss/releases/download/2.4.4/tpm2-tss-2.4.4.tar.gz)<br>
    Installation information is at https://github.com/tpm2-software/tpm2-tss/blob/master/INSTALL.md<br>
	b.  Install the software tool [`tpm2-abmrd`](https://github.com/tpm2-software/tpm2-abrmd/releases/download/2.3.3/tpm2-abrmd-2.3.3.tar.gz)<br>
    Installation information is at https://github.com/tpm2-software/tpm2-abrmd/blob/master/INSTALL.md<br>
	c. Install the [`tpm2-tools`](https://github.com/tpm2-software/tpm2-tools/releases/download/4.3.0/tpm2-tools-4.3.0.tar.gz)<br>
    Installation information is at https://github.com/tpm2-software/tpm2-tools/blob/master/INSTALL.md<br>
	d. Install the [Docker packages](https://docs.docker.com/engine/install/ubuntu/)
	e. Shut down the Guest VM.<br><br>
	Click the triangled line to close Option 2
</details>

9. On the host, create a directory to support the virtual TPM device. Only `root` should have read/write permission to this directory:
   ```sh
   sudo mkdir -p /var/OVSA/
   sudo mkdir /var/OVSA/vtpm
   sudo mkdir /var/OVSA/vtpm/vtpm_isv_dev
   ```

**Note**: For steps 10 and 11, you can copy and edit the script named `start_ovsa_isv_dev_vm.sh` in the `Scripts/reference` directory in the OpenVINO™ Security Add-on repository instead of manually running the commands. If using the script, select the script with `isv` in the file name regardless of whether you are playing the role of the Model Developer or the role of the Independent Software Vendor. Edit the script to point to the correct directory locations and increment `vnc` for each Guest VM.
 
10. Start the vTPM on Host:
	```sh
	swtpm socket --tpmstate dir=/var/OVSA/vtpm/vtpm_isv_dev \
	 --tpm2 \
     --ctrl type=unixio,path=/var/OVSA/vtpm/vtpm_isv_dev/swtpm-sock \
     --log level=20
	```
	
11. Start the Guest VM:
	```sh
	sudo qemu-system-x86_64 \
	 -cpu host \
	 -enable-kvm \
	 -m 8192 \
	 -smp 8,sockets=1,cores=8,threads=1 \
	 -device e1000,netdev=hostnet0,mac=52:54:00:d1:66:6f \
	 -netdev tap,id=hostnet0,script=<path-to-scripts>/br0-qemu-ifup,downscript=<path-to-scripts>/br0-qemu-ifdown \
	 -device e1000,netdev=hostnet1,mac=52:54:00:d1:66:5f \
	 -netdev tap,id=hostnet1,script=<path-to-scripts>/virbr0-qemu-ifup,downscript=<path-to-scripts>/virbr0-qemu-ifdown \
	 -drive if=virtio,file=<path-to-disk-image>/ovsa_isv_dev_vm_disk.qcow2,cache=none \
	 -chardev socket,id=chrtpm,path=/var/OVSA/vtpm/vtpm_isv_dev/swtpm-sock \
	 -tpmdev emulator,id=tpm0,chardev=chrtpm \
	 -device tpm-tis,tpmdev=tpm0 \
	 -vnc :1
	```
   
	Use the QEMU runtime options in the command to change the memory amount or CPU assigned to this Guest VM.
   
12. Use a VNC client to log on to the Guest VM at `<host-ip-address>:1`

</details>

### Step 4: Set Up one Guest VM for the User role

1. Choose ONE of these options to create a Guest VM for the User role:

<details><summary>Option 1: Copy and Rename the `ovsa_isv_dev_vm_disk.qcow2` disk image</summary>
1. Copy the `ovsa_isv_dev_vm_disk.qcow2` disk image to a new image named `ovsa_runtime_vm_disk.qcow2`. You created the `ovsa_isv_dev_vm_disk.qcow2` disk image in <a  href="#prerequisites">Step 3</a>.

2. Boot the new image. 

3. Change the hostname from `ovsa_isv_dev` to `ovsa_runtime`.  
	```sh 
	sudo hostnamectl set-hostname ovsa_runtime
	```
	
4. Replace all instances of 'ovsa_isv_dev' to 'ovsa_runtime' in the new image.
	
	```sh 	
	sudo nano /etc/hosts
	```
5. Change the `/etc/machine-id`:
	```sh
	sudo rm /etc/machine-id
	systemd-machine-id-setup
	```
6. Shut down the Guest VM.<br><br>

Click the triangled line above to close Option 1.
</details>

<details><summary>Option 2: Manually create the Guest VM</summary>
	
1. Create an empty virtual disk image:
	```sh
	sudo qemu-img create -f qcow2 <path>/ovsa_ovsa_runtime_vm_disk.qcow2 20G
	```

2. Install Ubuntu 18.04 on the Guest VM. Name the Guest VM `ovsa_runtime`:
	```sh
	sudo qemu-system-x86_64 -m 8192 -enable-kvm \
	-cpu host \
	-drive if=virtio,file=<path-to-disk-image>/ovsa_ovsa_runtime_vm_disk.qcow2,cache=none \
	-cdrom <path-to-iso-image>/ubuntu-18.04.5-live-server-amd64.iso \
	-device e1000,netdev=hostnet1,mac=52:54:00:d1:66:5f \
	-netdev tap,id=hostnet1,script=<path-to-scripts>/virbr0-qemu-ifup,downscript=<path-to-scripts>/virbr0-qemu-ifdown \
	-vnc :2
	```
	
3. Connect a VNC client with `<host-ip-address>:2`.
	
4. Follow the prompts on the screen to finish installing the Guest VM. Name the Guest VM `ovsa_runtime`.
	
5. Shut down the Guest VM. 
	
6. Restart the Guest VM:
	```sh
	sudo qemu-system-x86_64 -m 8192 -enable-kvm \
	-cpu host \
	-drive if=virtio,file=<path-to-disk-image>/ovsa_ovsa_runtime_vm_disk.qcow2,cache=none \
	-device e1000,netdev=hostnet1,mac=52:54:00:d1:66:5f \
	-netdev tap,id=hostnet1,script=<path-to-scripts>/virbr0-qemu-ifup,downscript=<path-to-scripts>/virbr0-qemu-ifdown \
	-vnc :2
	```
	
7. Choose ONE of these options to install additional required software:
<details><summary>Option 1: Use a script to install additional software</summary>
	a. Copy the script `install_guest_deps.sh` from the Scripts/reference directory of the OVSA repository to the Guest VM
	b. Run the script.
	c. Shut down the Guest VM.<br><br>
	
Click the triangled line to close Option 2.
	
</details>

<details><summary>Option 2: Manually install additional software</summary>
	a.  Install the software tool [`tpm2-tss`](https://github.com/tpm2-software/tpm2-tss/releases/download/2.4.4/tpm2-tss-2.4.4.tar.gz) <br>
    Installation information is at https://github.com/tpm2-software/tpm2-tss/blob/master/INSTALL.md <br><br>
	b.  Install the software tool [`tpm2-abmrd`](https://github.com/tpm2-software/tpm2-abrmd/releases/download/2.3.3/tpm2-abrmd-2.3.3.tar.gz) <br>
    Installation information is at https://github.com/tpm2-software/tpm2-abrmd/blob/master/INSTALL.md <br><br>
	c. Install the [`tpm2-tools`](https://github.com/tpm2-software/tpm2-tools/releases/download/4.3.0/tpm2-tools-4.3.0.tar.gz) <br>
    Installation information is at https://github.com/tpm2-software/tpm2-tools/blob/master/INSTALL.md <br><br>
	d. Install the [Docker packages](https://docs.docker.com/engine/install/ubuntu/) <br><br>
	e. Shut down the Guest VM.<br><br>

Click the triangled line to close the option to manually install additional software.
</details>

</details>

2. Create a directory to support the virtual TPM device. Only `root` should have read/write permission to this directory:
	```sh
	sudo mkdir /var/OVSA/vtpm/vtpm_runtime
	```
**Note**: For steps 3 and 4, you can copy and edit the script named `start_ovsa_runtime_vm.sh` in the scripts directory in the OpenVINO™ Security Add-on repository instead of manually running the commands. Edit the script to point to the correct directory locations and increment `vnc` for each Guest VM. This means that if you are creating a third Guest VM on the same Host Machine, change `-vnc :2` to `-vnc :3`

3. Start the vTPM:
	```sh
	swtpm socket --tpmstate dir=/var/OVSA/vtpm/vtpm_runtime \
	--tpm2 \
	--ctrl type=unixio,path=/var/OVSA/vtpm/vtpm_runtime/swtpm-sock \
	--log level=20
	```
4. Start the Guest VM in a new terminal. To do so, either copy and edit the script named `start_ovsa_runtime_vm.sh` in the scripts directory in the OpenVINO™ Security Add-on repository or manually run the command:
	```sh
	sudo qemu-system-x86_64 \
	 -cpu host \
	 -enable-kvm \
	 -m 8192 \
	 -smp 8,sockets=1,cores=8,threads=1 \
	 -device e1000,netdev=hostnet2,mac=52:54:00:d1:67:6f \
	 -netdev tap,id=hostnet2,script=<path-to-scripts>/br0-qemu-ifup,downscript=<path-to-scripts>/br0-qemu-ifdown \
	 -device e1000,netdev=hostnet3,mac=52:54:00:d1:67:5f \
	 -netdev tap,id=hostnet3,script=<path-to-scripts>/virbr0-qemu-ifup,downscript=<path-to-scripts>/virbr0-qemu-ifdown \
	 -drive if=virtio,file=<path-to-disk-image>/ovsa_runtime_vm_disk.qcow2,cache=none \
	 -chardev socket,id=chrtpm,path=/var/OVSA/vtpm/vtpm_runtime/swtpm-sock \
	 -tpmdev emulator,id=tpm0,chardev=chrtpm \
	 -device tpm-tis,tpmdev=tpm0 \
	 -vnc :2
	```

   Use the QEMU runtime options in the command to change the memory amount or CPU assigned to this Guest VM.
   
5. Use a VNC client to log on to the Guest VM at `<host-ip-address>:<x>` where `<x>` corresponds to the vnc number in the `start_ovsa_isv_vm.sh` or in step 8.

 </details>  

## How to Build and Install the OpenVINO™ Security Add-on Software <a name="install-ovsa"></a>

Follow the below steps to build and Install OpenVINO™ Security Add-on on host and different VMs.

### Step 1: Build the OpenVINO™ Model Server image 
Building OpenVINO™ Security Add-on depends on OpenVINO™ Model Server docker containers. Download and build OpenVINO™ Model Server first on the host.

1. Download the [OpenVINO™ Model Server software](https://github.com/openvinotoolkit/model_server)
2. Build the [OpenVINO™ Model Server Docker images](https://github.com/openvinotoolkit/model_server/blob/main/docs/docker_container.md)
	```sh
	git clone https://github.com/openvinotoolkit/model_server.git
	cd model_server
	make docker_build
	```
### Step 2: Build the software required for all roles

This step is for the combined role of Model Developer and Independent Software Vendor, and the User

1. Download the [OpenVINO™ Security Add-on](https://github.com/openvinotoolkit/security_addon)

2. Go to the top-level OpenVINO™ Security Add-on source directory.
   ```sh
   cd security_addon
   ```
3. Build the OpenVINO™ Security Add-on:
   ```sh
   make clean all
   sudo make package
   ```
	The following packages are created under the `release_files` directory:
	- `ovsa-kvm-host.tar.gz`: Host Machine file
	- `ovsa-developer.tar.gz`: For the Model Developer and the Independent Software Developer
	- `ovsa-model-hosting.tar.gz`: For the User

### Step 3: Install the host software
This step is for the combined role of Model Developer and Independent Software Vendor, and the User. 

1. Go to the `release_files` directory:
	```sh
    cd release_files
2. Set up the path:
	```sh
	export OVSA_RELEASE_PATH=$PWD
     ```
3.  Install the OpenVINO™ Security Add-on Software on the Host Machine: 
     ```sh
     cd $OVSA_RELEASE_PATH
     tar xvfz ovsa-kvm-host.tar.gz
     cd ovsa-kvm-host
     ./install.sh
       ```

If you are using more than one Host Machine repeat Step 3 on each.

### Step 4: Set up packages on the Guest VM
This step is for the combined role of Model Developer and Independent Software Vendor. References to the Guest VM are to `ovsa_isv_dev`.
 
1. Log on to the Guest VM.
2. Create the OpenVINO™ Security Add-on directory in the home directory
    ```sh
    mkdir OVSA

3. Go to the Host Machine, outside of the Guest VM.

4. Copy `ovsa-developer.tar.gz` from `release_files` to the Guest VM:
    ```sh
    cd $OVSA_RELEASE_PATH
    scp ovsa-developer.tar.gz username@<isv-developer-vm-ip-address>:/<username-home-directory>/OVSA
    ```
5. Go to the Guest VM.

6. Install the software to the Guest VM:
     ```sh
     cd OVSA
     tar xvfz ovsa-developer.tar.gz
     cd ovsa-developer
     sudo -s
     ./install.sh
     ```
7. Create  a directory named `artefacts`. This directory will hold artefacts required to create licenses:
	```sh
	cd /<username-home-directory>/OVSA
	mkdir artefacts
	cd artefacts
	```

8. Start the license server on a separate terminal.
	```sh
	sudo -s
	source /opt/ovsa/scripts/setupvars.sh
	cd /opt/ovsa/bin
	./license_server
	```
	
### Step 5: Install the OpenVINO™ Security Add-on Model Hosting Component

This step is for the User. References to the Guest VM are to `ovsa_runtime`.

The Model Hosting components install the OpenVINO™ Security Add-on Runtime Docker container based on OpenVINO™ Model Server NGINX Docker to host a access controlled model. 
    
1. Log on to the Guest VM as `<user>`.
2. Create the OpenVINO™ Security Add-on directory in the home directory
    ```sh
    mkdir OVSA
    ```
3. While on the Host Machine copy the ovsa-model-hosting.tar.gz from release_files to the Guest VM:
	```sh
    cd $OVSA_RELEASE_PATH
    scp ovsa-model-hosting.tar.gz username@<isv-developer-vm-ip-address>:/<username-home-directory>/OVSA
    ```
4. Install the software to the Guest VM:
    ```sh
    cd OVSA
    tar xvfz ovsa-model-hosting.tar.gz
    cd ovsa-model-hosting
    sudo -s
    ./install.sh
    ```
5. Create a directory named `artefacts`:
	```sh
	cd /<username-home-directory>/OVSA
	mkdir artefacts
	cd artefacts
	```

## How to Use the OpenVINO™ Security Add-on

This section requires interactions between the Model Developer/Independent Software vendor and the User. All roles must complete all applicable <a href="#setup-host">set up steps</a> and <a href="#ovsa-install">installation steps</a> before beginning this section.

This document uses the face-detection-retail-0004 model as an example. 

The following figure describes the interactions between the Model Developer, Independent Software Vendor, and User.

**Remember**: The Model Developer/Independent Software Vendor and User roles are related to virtual machine use and one person might fill the tasks required by multiple roles. In this document the tasks of Model Developer and Independent Software Vendor are combined and use the Guest VM named `ovsa_isv`. It is possible to have all roles set up on the same Host Machine.

![OpenVINO™ Security Add-on Example Diagram](ovsa_example.png)

### Model Developer Instructions

The Model Developer creates model, defines access control and creates the user license. References to the Guest VM are to `ovsa_isv_dev`. After the model is created, access control enabled, and the license is ready, the Model Developer provides the license details to the Independent Software Vendor before sharing to the Model User.

#### Step 1: Create a key store and add a certificate to it

1. Set up a path to the artefacts directory:
	```sh
	sudo -s
	cd /<username-home-directory>/OVSA/artefacts
	export OVSA_RUNTIME_ARTEFACTS=$PWD
	source /opt/ovsa/scripts/setupvars.sh
	
2. Create files to request a certificate:<br>
	This example uses a self-signed certificate for demonstration purposes. In a production environment, use CSR files to request for a CA-signed certificate.
 
	```sh
	cd $OVSA_DEV_ARTEFACTS
	/opt/ovsa/bin/ovsatool keygen -storekey -t ECDSA -n Intel -k isv_keystore -r  isv_keystore.csr -e "/C=IN/CN=localhost"
 	```
	Two files are created:<br>
	- `isv_keystore.csr`- A Certificate Signing Request (CSR)  
	- `isv_keystore.csr.crt` - A self-signed certificate

	In a production environment, send `isv_keystore.csr` to a CA to request a CA-signed certificate.
	
3. Add the certificate to the key store
	```sh
	/opt/ovsa/bin/ovsatool keygen -storecert -c isv_keystore.csr.crt -k isv_keystore
	```	
	
#### Step 2: Create the model

This example uses `curl` to download the `face-detection-retail-004` model from the OpenVINO Model Zoo. If you are behind a firewall, check and set your proxy settings.

1. Log on to the Guest VM.
 
2. Download a model from the Model Zoo:
	```sh
	cd $OVSA_DEV_ARTEFACTS	
	curl --create-dirs https://download.01.org/opencv/2021/openvinotoolkit/2021.1/open_model_zoo/models_bin/1/face-detection-retail-0004/FP32/face-detection-retail-0004.xml https://download.01.org/opencv/2021/openvinotoolkit/2021.1/open_model_zoo/models_bin/1/face-detection-retail-0004/FP32/face-detection-retail-0004.bin -o model/face-detection-retail-0004.xml -o model/face-detection-retail-0004.bin
	```
	The model is downloaded to the `OVSA_DEV_ARTEFACTS/model` directory
	
#### Step 3: Define access control for  the model and create a master license for it

1. Go to the `artefacts` directory:
	```sh	
	cd $OVSA_DEV_ARTEFACTS
	```
2. 
	```sh	
	uuidgen
	```
3. Define and enable the model access control and master license:
	```sh	
	/opt/ovsa/bin/ovsatool protect -i model/face-detection-retail-0004.xml model/face-detection-retail-0004.bin -n "face detection" -d "face detection retail" -v 0004 -p face_detection_model.dat -m face_detection_model.masterlic -k isv_keystore -g <output-of-uuidgen>
	```
The Intermediate Representation files for the `face-detection-retail-0004` model are encrypted as `face_detection_model.dat` and a master license is generated as `face_detection_model.masterlic`

#### Step 4: Create a Runtime Reference TCB

Use the runtime reference TCB to create a customer license for the access controlled model and the specific runtime.

Generate the reference TCB for the runtime
```sh
cd $OVSA_DEV_ARTEFACTS
source /opt/ovsa/scripts/setupvars.sh
	/opt/ovsa/bin/ovsaruntime gen-tcb-signature -n "Face Detect @ Runtime VM" -v "1.0" -f face_detect_runtime_vm.tcb -k isv_keystore
```
	
#### Step 5: Publish the access controlled Model and Runtime Reference TCB
The access controlled model is ready to be shared with the User and the reference TCB is ready to perform license checks.

#### Step 6: Receive a User Request
1. Obtain artefacts from the User who needs access to a access controlled model:
	* Customer certificate from the customer's key store.
	* Other information that apply to your licensing practices, such as the length of time the user needs access to the model

2. Create a customer license configuration
	```sh
	cd $OVSA_DEV_ARTEFACTS
	/opt/ovsa/bin/ovsatool licgen -t TimeLimit -l30 -n "Time Limit License Config" -v 1.0 -u "<isv-developer-vm-ip-address>:<license_server-port>" -k isv_keystore -o 30daylicense.config
	```
3. Create the customer license
	```sh
	cd $OVSA_DEV_ARTEFACTS
	/opt/ovsa/bin/ovsatool sale -m face_detection_model.masterlic -k isv_keystore -l 30daylicense.config -t face_detect_runtime_vm.tcb -p custkeystore.csr.crt -c face_detection_model.lic
	```
	
4. Update the license server database with the license.
	```sh
	cd /opt/ovsa/DB
	python3 ovsa_store_customer_lic_cert_db.py ovsa.db $OVSA_DEV_ARTEFACTS/face_detection_model.lic $OVSA_DEV_ARTEFACTS/custkeystore.csr.crt
	```

5. Provide these files to the User:
	* `face_detection_model.dat`
	* `face_detection_model.lic`

### User Instructions
References to the Guest VM are to `ovsa_rumtime`.

#### Step 1: Add a CA-Signed Certificate to a Key Store

1. Set up a path to the artefacts directory:
	```sh
	sudo -s
	cd /<username-home-directory>/OVSA/artefacts
	export OVSA_RUNTIME_ARTEFACTS=$PWD
	source /opt/ovsa/scripts/setupvars.sh
	```
2. Generate a Customer key store file:
	```sh
	cd $OVSA_RUNTIME_ARTEFACTS
	/opt/ovsa/bin/ovsatool keygen -storekey -t ECDSA -n Intel -k custkeystore -r  custkeystore.csr -e "/C=IN/CN=localhost"
	```
	Two files are created:
	* `custkeystore.csr` - A Certificate Signing Request (CSR)
	* `custkeystore.csr.crt` - A self-signed certificate

3. Send  `custkeystore.csr` to the CA to request a CA-signed certificate.

4. Add the certificate to the key store:
	```sh
	/opt/ovsa/bin/ovsatool keygen -storecert -c custkeystore.csr.crt -k custkeystore
	```

#### Step 2: Request an access controlled Model from the Model Developer
This example uses scp to share data between the ovsa_runtime and ovsa_dev Guest VMs on the same Host Machine.

1. Communicate your need for a model to the Model Developer. The Developer will ask you to provide the certificate from your key store and other information. This example uses the length of time the model needs to be available. 
2. Generate an artefact file to provide to the Developer:
	```sh
	cd $OVSA_RUNTIME_ARTEFACTS
	scp custkeystore.csr.crt username@<developer-vm-ip-address>:/<username-home-directory>/OVSA/artefacts
	```
#### Step 3: Receive and load the access controlled model into the OpenVINO™ Model Server
1. Receive the model as files named
	* face_detection_model.dat
	* face_detection_model.lic

2. Prepare the environment:
	```sh
	cd $OVSA_RUNTIME_ARTEFACTS/..
	cp /opt/ovsa/example_runtime ovms -r
	cd ovms
	mkdir -vp model/fd/1
	```
The `$OVSA_RUNTIME_ARTEFACTS/../ovms` directory contains scripts and a sample configuration JSON file to start the model server. 

3. Copy the artefacts from the Model Developer:
	```sh
	cd $OVSA_RUNTIME_ARTEFACTS/../ovms
	cp $OVSA_RUNTIME_ARTEFACTS/face_detection_model.dat model/fd/1/.
	cp $OVSA_RUNTIME_ARTEFACTS/face_detection_model.lic model/fd/1/.
	cp $OVSA_RUNTIME_ARTEFACTS/custkeystore model/fd/1/.
	```
4. Rename and edit `sample.json` to include the names of the access controlled model artefacts you received from the Model Developer. The file looks like this:
	```sh
	{
	"custom_loader_config_list":[
		{
			"config":{
					"loader_name":"ovsa",
					"library_path": "/ovsa-runtime/lib/libovsaruntime.so"
			}
		}
	],
	"model_config_list":[
		{
		"config":{
			"name":"protected-model",
			"base_path":"/sampleloader/model/fd",
			"custom_loader_options": {"loader_name":  "ovsa", "keystore":  "custkeystore", "protected_file": "face_detection_model"}
		}
		}
	]
	}
	```
#### Step 4: Start the NGINX Model Server
The NGINX Model Server publishes the access controlled model.
	```sh
	./start_secure_ovsa_model_server.sh
	```
For information about the NGINX interface, see https://github.com/openvinotoolkit/model_server/blob/main/extras/nginx-mtls-auth/README.md

#### Step 5: Prepare to run Inference

1. Log on to the Guest VM from another terminal.

2. Install the Python dependencies for your set up. For example:
	```sh
	sudo apt install pip3
	pip3 install cmake
	pip3 install scikit-build
	pip3 install opencv-python
	pip3 install futures==3.1.1
	pip3 install tensorflow-serving-api==1.14.0
	```
3. Copy the `face_detection.py` from the example_client in `/opt/ovsa/example_client`
	```sh
	cd /home/intel/OVSA/ovms
	cp /opt/ovsa/example_client/* .
	```
4. Copy the sample images for inferencing. An image directory is created that includes a sample image for inferencing.
	```sh
	curl --create-dirs https://raw.githubusercontent.com/openvinotoolkit/model_server/master/example_client/images/people/people1.jpeg -o images/people1.jpeg
	```
#### Step 6: Run Inference

Run the `face_detection.py` script. 
```sh
python3 face_detection.py --grpc_port 3335 --batch_size 1 --width 300 --height 300 --input_images_dir images --output_dir results --tls --server_cert server.pem --client_cert client.pem --client_key client.key --model_name protected-model
```	

## Summary
You have completed these tasks:
- Set up one or more computers (Host Machines) with one KVM per machine and one or more virtual machines (Guest VMs) on the Host Machines
- Installed the OpenVINO™ Security Add-on 
- Used the OpenVINO™ Model Server to work with OpenVINO™ Security Add-on 
- As a Model Developer or Independent Software Vendor, you access controlled a model and prepared a license for it.
- As a Model Developer or Independent Software Vendor, you prepared and ran a License Server and used the License Server to verify a User had a valid license to use a access controlled model.
- As a User, you provided information to a Model Developer or Independent Software Vendor to get a access controlled model and the license for the model.
- As a User, you set up and launched a Host Server on which you can run licensed and access controlled models.
- As a User, you loaded a access controlled model, validated the license for the model, and used the model to run inference.

## References
Use these links for more information:
- [OpenVINO&trade; toolkit](https://software.intel.com/en-us/openvino-toolkit)
- [OpenVINO Model Server Quick Start Guide](https://github.com/openvinotoolkit/model_server/blob/main/docs/ovms_quickstart.md)
- [Model repository](https://github.com/openvinotoolkit/model_server/blob/main/docs/models_repository.md)