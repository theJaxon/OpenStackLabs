# OpenStack Labs:

[![forthebadge](https://forthebadge.com/images/featured/featured-made-with-crayons.svg)](https://forthebadge.com)


### Lab 1:
#### Installing OpenStack Rocky on Ubuntu Bionic:

#### Steps:
- Configure Vagrant
- Configure [Ubuntu Bionic box](https://app.vagrantup.com/ubuntu/boxes/bionic64) to install Devstack
- Modify `local.conf` file
- Modify `openrc` file

## Warning: DevStack doesn't survive **reboots** so all it takes is `vagrant reload` and you completely mess your setup
* Take a snapshot after the installation

<details><summary>Configure Vagrant</summary>
<p>

```Ruby

Vagrant.configure("2") do |config|

  config.vm.box = "ubuntu/bionic64"
  config.vm.hostname = "DevStack"
  config.vm.synced_folder ".", "/vagrant", type: "rsync"
  config.vm.network "public_network", ip: "192.168.1.10"
  config.vm.network "forwarded_port", guest: 80, host: 8060 # Horizon
  config.vm.network "forwarded_port", guest: 5000, host: 5000 # Authentication

  config.vm.provider "virtualbox" do |vb|

  # Customize the amount of memory on the VM:
      #vb.memory = "4096" # 4 Gigs 
      #vb.memory = "6144" # 6 Gigs 
      vb.memory = "8192" # 8 Gigs 
      vb.cpus = 4 
  end
end

```

</p>
</details>


<details><summary>Configure Ubuntu bionic box to install Devstack</summary>
<p>


```bash

# Create a DevStack user 
sudo useradd -s /bin/bash -d /opt/stack -m stack

# Give sudo permissions to the user
echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack

# Login as that user
sudo su - stack

# Clone the OpenStack Rocky release from the /stable branch
git clone https://github.com/openstack-dev/devstack.git -b stable/rocky devstack/

# cd into the cloned directory
cd devstack

# Copy the configuration file from /samples/local.conf into the current directory
cp samples/local.conf .

```

</p>
</details>

<details><summary>Modify local.conf file</summary>
<p>

```json
[[local|localrc]]
ADMIN_PASSWORD=vagrant
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD
HOST_IP=192.168.1.10

```

</p>
</details>

<details><summary>Modify openrc file</summary>
<p>


```bash

#!/usr/bin/env bash
#
# source openrc [username] [projectname]
#
# Configure a set of credentials for $PROJECT/$USERNAME:
#   Set OS_PROJECT_NAME to override the default project 'demo'
#   Set OS_USERNAME to override the default user name 'demo'
#   Set ADMIN_PASSWORD to set the password for 'admin' and 'demo'

# NOTE: support for the old NOVA_* novaclient environment variables has
# been removed.

if [[ -n "$1" ]]; then
    OS_USERNAME=$1
fi
if [[ -n "$2" ]]; then
    OS_PROJECT_NAME=$2
fi

# Find the other rc files
RC_DIR=$(cd $(dirname "${BASH_SOURCE:-$0}") && pwd)

# Import common functions
source $RC_DIR/functions

# Load local configuration
source $RC_DIR/stackrc

# Load the last env variables if available
if [[ -r $RC_DIR/.stackenv ]]; then
    source $RC_DIR/.stackenv
    export OS_CACERT
fi

# Get some necessary configuration
source $RC_DIR/lib/tls

# The OpenStack ecosystem has standardized the term **project** as the
# entity that owns resources.  In some places **tenant** remains
# referenced, but in all cases this just means **project**.  We will
# warn if we need to turn on legacy **tenant** support to have a
# working environment.
export OS_PROJECT_NAME=${OS_PROJECT_NAME:-admin}

export OS_TENANT_NAME=$OS_PROJECT_NAME

# In addition to the owning entity (project), nova stores the entity performing
# the action as the **user**.
export OS_USERNAME=${OS_USERNAME:-admin}

# With Keystone you pass the keystone password instead of an api key.
# Recent versions of novaclient use OS_PASSWORD instead of NOVA_API_KEYs
# or NOVA_PASSWORD.
export OS_PASSWORD=${ADMIN_PASSWORD:-vagrant}

# Region
export OS_REGION_NAME=${REGION_NAME:-RegionOne}

# Set the host API endpoint. This will default to HOST_IP if SERVICE_IP_VERSION
# is 4, else HOST_IPV6 if it's 6. SERVICE_HOST may also be used to specify the
# endpoint, which is convenient for some localrc configurations. Additionally,
# some exercises call Glance directly. On a single-node installation, Glance
# should be listening on a local IP address, depending on the setting of
# SERVICE_IP_VERSION. If its running elsewhere, it can be set here.
if [[ $SERVICE_IP_VERSION == 6 ]]; then
    HOST_IPV6=${HOST_IPV6:-::1}
    SERVICE_HOST=${SERVICE_HOST:-[$HOST_IPV6]}
    GLANCE_HOST=${GLANCE_HOST:-[$HOST_IPV6]}
else
    HOST_IP=${HOST_IP:-192.168.1.10}
    SERVICE_HOST=${SERVICE_HOST:-$HOST_IP}
    GLANCE_HOST=${GLANCE_HOST:-$HOST_IP}
fi

# Identity API version
export OS_IDENTITY_API_VERSION=${IDENTITY_API_VERSION:-3}

# Ask keystoneauth1 to use keystone
export OS_AUTH_TYPE=password

# Authenticating against an OpenStack cloud using Keystone returns a **Token**
# and **Service Catalog**.  The catalog contains the endpoints for all services
# the user/project has access to - including nova, glance, keystone, swift, ...
# We currently recommend using the version 3 *identity api*.
#

# If you don't have a working .stackenv, this is the backup position
KEYSTONE_BACKUP=$SERVICE_PROTOCOL://$SERVICE_HOST:5000
KEYSTONE_AUTH_URI=${KEYSTONE_AUTH_URI:-$KEYSTONE_BACKUP}

export OS_AUTH_URL=${OS_AUTH_URL:-$KEYSTONE_AUTH_URI}

# Currently, in order to use openstackclient with Identity API v3,
# we need to set the domain which the user and project belong to.
if [ "$OS_IDENTITY_API_VERSION" = "3" ]; then
    export OS_USER_DOMAIN_ID=${OS_USER_DOMAIN_ID:-"default"}
    export OS_PROJECT_DOMAIN_ID=${OS_PROJECT_DOMAIN_ID:-"default"}
fi

# Set OS_CACERT to a default CA certificate chain if it exists.
if [[ ! -v OS_CACERT ]] ; then
    DEFAULT_OS_CACERT=$INT_CA_DIR/ca-chain.pem
    # If the file does not exist, this may confuse preflight sanity checks
    if [ -e $DEFAULT_OS_CACERT ] ; then
        export OS_CACERT=$DEFAULT_OS_CACERT
    fi
fi

# Currently cinderclient needs you to specify the *volume api* version. This
# needs to match the config of your catalog returned by Keystone.
export CINDER_VERSION=${CINDER_VERSION:-3}
export OS_VOLUME_API_VERSION=${OS_VOLUME_API_VERSION:-$CINDER_VERSION}


```

</p>
</details>

Now run `./stack.sh` and let [DevStack](https://github.com/openstack/devstack/tree/stable/rocky) installer do its magic .. this takes a bit of time as it downloads and installs all the components of OpenStack so **be patient** ..
* There was an issue when launching cerros instance and it was fixed by running `sudo systemctl start iscsid`
#### Previews:
<details><summary>Terminal after successfull installation</summary>
<p>

 <img src="https://github.com/theJaxon/OpenStackLabs/blob/master/etc/Lab%201/FinalResult.jpeg">

</p>
</details>

<details><summary>OpenStack Dashboard [Keystone auth]</summary>
<p>

<img src="https://github.com/theJaxon/OpenStackLabs/blob/master/etc/Lab%201/LoginPage.jpg">


</p>
</details>

<details><summary>Horizon homepage</summary>
<p>

<img src="https://github.com/theJaxon/OpenStackLabs/blob/master/etc/Lab%201/Dashboard.jpg">


</p>
</details>

---

### Lab 2:
### Part 1 [Horizon dashboard tasks]:
1- Launch cirros instance using tiny flavor and launch instance console
    - From instances tab inside Compute click "Launch instance"
    - Remaining steps are demonstrated using the images below
    
<details><summary>Cirros instance steps</summary>
<p>

<img src="https://github.com/theJaxon/OpenStackLabs/blob/master/etc/Lab%202/Q1/1.jpg">
<img src="https://github.com/theJaxon/OpenStackLabs/blob/master/etc/Lab%202/Q1/2.jpg">
<img src="https://github.com/theJaxon/OpenStackLabs/blob/master/etc/Lab%202/Q1/3.jpg">
<img src="https://github.com/theJaxon/OpenStackLabs/blob/master/etc/Lab%202/Q1/4.jpg">
<img src="https://github.com/theJaxon/OpenStackLabs/blob/master/etc/Lab%202/Q1/5.jpg">




</p>
</details>

2- Create new flavor  1024 MB Ram 5 GB HD 1 vcpus
    - From the `admin` project in the admin sidebar > Compute > Flavours
    
<details><summary>Flavor creation</summary>
<p>

<img src="https://github.com/theJaxon/OpenStackLabs/blob/master/etc/Lab%202/Q2/1.jpg">
<img src="https://github.com/theJaxon/OpenStackLabs/blob/master/etc/Lab%202/Q2/2.jpg">

</p>
</details>

3- Create new image based on ubuntu/centos image (make sure to use minimal image due to limited resources)
    - From `admin` project > admin > Compute > Images
    
<details><summary>Image creation</summary>
<p>

<img src="https://github.com/theJaxon/OpenStackLabs/blob/master/etc/Lab%202/Q3/1.jpg">
<img src="https://github.com/theJaxon/OpenStackLabs/blob/master/etc/Lab%202/Q3/2.jpg">

</p>
</details>

4- Stop cirros instance and delete it

<details><summary>Stopping and deleting cirros</summary>
<p>

<img src="https://github.com/theJaxon/OpenStackLabs/blob/master/etc/Lab%202/Q4/1.jpg">
<img src="https://github.com/theJaxon/OpenStackLabs/blob/master/etc/Lab%202/Q4/2.jpg">

</p>
</details>


5- Create new instance using the created flavor and image

<details><summary>new instance creation</summary>
<p>

<img src="https://github.com/theJaxon/OpenStackLabs/blob/master/etc/Lab%202/Q5/1.jpg">
<img src="https://github.com/theJaxon/OpenStackLabs/blob/master/etc/Lab%202/Q5/2.jpg">

</p>
</details>

6- stop/start the instance

<details><summary>new instance creation</summary>
<p>

<img src="https://github.com/theJaxon/OpenStackLabs/blob/master/etc/Lab%202/Q6/1.jpg">
<img src="https://github.com/theJaxon/OpenStackLabs/blob/master/etc/Lab%202/Q6/2.jpg">
<img src="https://github.com/theJaxon/OpenStackLabs/blob/master/etc/Lab%202/Q6/3.jpg">

</p>
</details>

7- Export any light vm from your system – import it in openstack as new image (Virtual box or vmware) then launch an instance form this new image 

***

### Part 2 [Using CLI]:
1- [List](https://docs.openstack.org/python-openstackclient/pike/cli/command-objects/project.html) all projects

`openstack project list`

<details><summary>Project list output</summary>
<p>

```

+----------------------------------+--------------------+
| ID                               | Name               |
+----------------------------------+--------------------+
| 01bffc0d7ade4fe39eaa1352f80abf8a | admin              |
| 604d0274aa6045ada6ea0c2d1f6fabff | invisible_to_admin |
| db71ce84b7d447069481a449b6f91db2 | alt_demo           |
| f003a3672da345ff860da82c05a792a1 | demo               |
| f1c6afc9e16748d09a43ef982e2b0b16 | service            |
+----------------------------------+--------------------+

```

</p>
</details>

2- Create new project 

`openstack project create theJaxon --description "Testing the project create command"`

<details><summary>create project output</summary>
<p>

```

+-------------+------------------------------------+
| Field       | Value                              |
+-------------+------------------------------------+
| description | Testing the project create command |
| domain_id   | default                            |
| enabled     | True                               |
| id          | 72079d2e51dc4c1ea74e10a1c90e7aa1   |
| is_domain   | False                              |
| name        | theJaxon                           |
| parent_id   | default                            |
| tags        | []                                 |
+-------------+------------------------------------+


```

</p>
</details>

3- set [quota](https://docs.openstack.org/python-openstackclient/pike/cli/command-objects/quota.html) for the new project to `10 instances – 20 core – 80400 ram`
* By default the number of instances is set to 10 and the core to 20 so the only needed modification was the ram

`openstack quota set theJaxon --ram 80400`

`openstack quota show theJaxon`

<details><summary>quota show</summary>
<p>

```

+-----------------------+----------------------------------+
| Field                 | Value                            |
+-----------------------+----------------------------------+
| backup-gigabytes      | 1000                             |
| backups               | 10                               |
| cores                 | 20                               |
| fixed-ips             | -1                               |
| floating-ips          | 50                               |
| gigabytes             | 1000                             |
| gigabytes_lvmdriver-1 | -1                               |
| groups                | 10                               |
| health_monitors       | None                             |
| injected-file-size    | 10240                            |
| injected-files        | 5                                |
| injected-path-size    | 255                              |
| instances             | 10                               |
| key-pairs             | 100                              |
| l7_policies           | None                             |
| listeners             | None                             |
| load_balancers        | None                             |
| location              | None                             |
| name                  | None                             |
| networks              | 100                              |
| per-volume-gigabytes  | -1                               |
| pools                 | None                             |
| ports                 | 500                              |
| project               | 72079d2e51dc4c1ea74e10a1c90e7aa1 |
| project_name          | theJaxon                         |
| properties            | 128                              |
| ram                   | 80400                            |
| rbac_policies         | 10                               |
| routers               | 10                               |
| secgroup-rules        | 100                              |
| secgroups             | 10                               |
| server-group-members  | 10                               |
| server-groups         | 10                               |
| snapshots             | 10                               |
| snapshots_lvmdriver-1 | -1                               |
| subnet_pools          | -1                               |
| subnets               | 100                              |
| volumes               | 10                               |
| volumes_lvmdriver-1   | -1                               |
+-----------------------+----------------------------------+


```

</p>
</details>

4- Create new [flavor](https://docs.openstack.org/python-openstackclient/pike/cli/command-objects/flavor.html) `1024 MB Ram 5 GB HD 1 vcpus`

`openstack flavor create --ram 1024 --disk 5 --vcpus 1 theJaxon`

<details><summary>flavor create output</summary>
<p>

```

+----------------------------+--------------------------------------+
| Field                      | Value                                |
+----------------------------+--------------------------------------+
| OS-FLV-DISABLED:disabled   | False                                |
| OS-FLV-EXT-DATA:ephemeral  | 0                                    |
| disk                       | 5                                    |
| id                         | 5dac01ac-1417-4c09-bf84-7832d1252c4b |
| name                       | theJaxon                             |
| os-flavor-access:is_public | True                                 |
| properties                 |                                      |
| ram                        | 1024                                 |
| rxtx_factor                | 1.0                                  |
| swap                       |                                      |
| vcpus                      | 1                                    |
+----------------------------+--------------------------------------+


```

</p>
</details>

5- Verify flavor creation

I went to the dashboard and found that it was listed there 

<details><summary>Flavor dashboard preview</summary>
<p>

```



```

</p>
</details>

6- Create new instance using the created flavor

`openstack server create --flavor theJaxon --image cirros-0.3.5-x86_64-disk cli-instance`

<details><summary>server create output</summary>
<p>

```

+-------------------------------------+-----------------------------------------------------------------+
| Field                               | Value                                                           |
+-------------------------------------+-----------------------------------------------------------------+
| OS-DCF:diskConfig                   | MANUAL                                                          |
| OS-EXT-AZ:availability_zone         |                                                                 |
| OS-EXT-SRV-ATTR:host                | None                                                            |
| OS-EXT-SRV-ATTR:hypervisor_hostname | None                                                            |
| OS-EXT-SRV-ATTR:instance_name       |                                                                 |
| OS-EXT-STS:power_state              | NOSTATE                                                         |
| OS-EXT-STS:task_state               | scheduling                                                      |
| OS-EXT-STS:vm_state                 | building                                                        |
| OS-SRV-USG:launched_at              | None                                                            |
| OS-SRV-USG:terminated_at            | None                                                            |
| accessIPv4                          |                                                                 |
| accessIPv6                          |                                                                 |
| addresses                           |                                                                 |
| adminPass                           | vFtkwoo62BjB                                                    |
| config_drive                        |                                                                 |
| created                             | 2020-04-12T10:31:36Z                                            |
| flavor                              | theJaxon (5dac01ac-1417-4c09-bf84-7832d1252c4b)                 |
| hostId                              |                                                                 |
| id                                  | b120c9d5-f7f8-441d-a0c3-6da429c19c59                            |
| image                               | cirros-0.3.5-x86_64-disk (6192282d-3cb0-42c4-8231-02bd27a30520) |
| key_name                            | None                                                            |
| name                                | cli-instance                                                    |
| progress                            | 0                                                               |
| project_id                          | 01bffc0d7ade4fe39eaa1352f80abf8a                                |
| properties                          |                                                                 |
| security_groups                     | name='default'                                                  |
| status                              | BUILD                                                           |
| updated                             | 2020-04-12T10:31:36Z                                            |
| user_id                             | 67078083cc2f4abfa0b1ae5985138f8b                                |
| volumes_attached                    |                                                                 |
+-------------------------------------+-----------------------------------------------------------------+


```

</p>
</details>

7- View [console log](https://docs.openstack.org/python-openstackclient/pike/cli/command-objects/console-log.html) of the instance

`openstack console log show cli-instance`

<details><summary>cli-instance logs</summary>
<p>

```python

[    0.000000] Initializing cgroup subsys cpuset
[    0.000000] Initializing cgroup subsys cpu
[    0.000000] Linux version 3.2.0-80-virtual (buildd@batsu) (gcc version 4.6.3 (Ubuntu/Linaro 4.6.3-1ubuntu5) ) #116-Ubuntu SMP Mon Mar 23 17:28:52 UTC 2015 (Ubuntu 3.2.0-80.116-virtual 3.2.68)
[    0.000000] Command line: LABEL=cirros-rootfs ro console=tty1 console=ttyS0
[    0.000000] KERNEL supported cpus:
[    0.000000]   Intel GenuineIntel
[    0.000000]   AMD AuthenticAMD
[    0.000000]   Centaur CentaurHauls
[    0.000000] BIOS-provided physical RAM map:
[    0.000000]  BIOS-e820: 0000000000000000 - 000000000009fc00 (usable)
[    0.000000]  BIOS-e820: 000000000009fc00 - 00000000000a0000 (reserved)
[    0.000000]  BIOS-e820: 00000000000f0000 - 0000000000100000 (reserved)
[    0.000000]  BIOS-e820: 0000000000100000 - 000000003ffdc000 (usable)
[    0.000000]  BIOS-e820: 000000003ffdc000 - 0000000040000000 (reserved)
[    0.000000]  BIOS-e820: 00000000fffc0000 - 0000000100000000 (reserved)
[    0.000000] NX (Execute Disable) protection: active
[    0.000000] SMBIOS 2.8 present.
[    0.000000] No AGP bridge found
[    0.000000] last_pfn = 0x3ffdc max_arch_pfn = 0x400000000
[    0.000000] x86 PAT enabled: cpu 0, old 0x7040600070406, new 0x7010600070106
[    0.000000] found SMP MP-table at [ffff8800000f6a80] f6a80
[    0.000000] init_memory_mapping: 0000000000000000-000000003ffdc000
[    0.000000] RAMDISK: 37c91000 - 37ff0000
[    0.000000] ACPI: RSDP 00000000000f6880 00014 (v00 BOCHS )
[    0.000000] ACPI: RSDT 000000003ffe15c9 00030 (v01 BOCHS  BXPCRSDT 00000001 BXPC 00000001)
[    0.000000] ACPI: FACP 000000003ffe1425 00074 (v01 BOCHS  BXPCFACP 00000001 BXPC 00000001)
[    0.000000] ACPI: DSDT 000000003ffe0040 013E5 (v01 BOCHS  BXPCDSDT 00000001 BXPC 00000001)
[    0.000000] ACPI: FACS 000000003ffe0000 00040
[    0.000000] ACPI: APIC 000000003ffe1519 00078 (v01 BOCHS  BXPCAPIC 00000001 BXPC 00000001)
[    0.000000] ACPI: HPET 000000003ffe1591 00038 (v01 BOCHS  BXPCHPET 00000001 BXPC 00000001)
[    0.000000] No NUMA configuration found
[    0.000000] Faking a node at 0000000000000000-000000003ffdc000
[    0.000000] Initmem setup node 0 0000000000000000-000000003ffdc000
[    0.000000]   NODE_DATA [000000003ffd7000 - 000000003ffdbfff]
[    0.000000] Zone PFN ranges:
[    0.000000]   DMA      0x00000010 -> 0x00001000
[    0.000000]   DMA32    0x00001000 -> 0x00100000
[    0.000000]   Normal   empty
[    0.000000] Movable zone start PFN for each node
[    0.000000] early_node_map[2] active PFN ranges
[    0.000000]     0: 0x00000010 -> 0x0000009f
[    0.000000]     0: 0x00000100 -> 0x0003ffdc
[    0.000000] ACPI: PM-Timer IO Port: 0x608
[    0.000000] ACPI: LAPIC (acpi_id[0x00] lapic_id[0x00] enabled)
[    0.000000] ACPI: LAPIC_NMI (acpi_id[0xff] dfl dfl lint[0x1])
[    0.000000] ACPI: IOAPIC (id[0x00] address[0xfec00000] gsi_base[0])
[    0.000000] IOAPIC[0]: apic_id 0, version 32, address 0xfec00000, GSI 0-23
[    0.000000] ACPI: INT_SRC_OVR (bus 0 bus_irq 0 global_irq 2 dfl dfl)
[    0.000000] ACPI: INT_SRC_OVR (bus 0 bus_irq 5 global_irq 5 high level)
[    0.000000] ACPI: INT_SRC_OVR (bus 0 bus_irq 9 global_irq 9 high level)
[    0.000000] ACPI: INT_SRC_OVR (bus 0 bus_irq 10 global_irq 10 high level)
[    0.000000] ACPI: INT_SRC_OVR (bus 0 bus_irq 11 global_irq 11 high level)
[    0.000000] Using ACPI (MADT) for SMP configuration information
[    0.000000] ACPI: HPET id: 0x8086a201 base: 0xfed00000
[    0.000000] SMP: Allowing 1 CPUs, 0 hotplug CPUs
[    0.000000] PM: Registered nosave memory: 000000000009f000 - 00000000000a0000
[    0.000000] PM: Registered nosave memory: 00000000000a0000 - 00000000000f0000
[    0.000000] PM: Registered nosave memory: 00000000000f0000 - 0000000000100000
[    0.000000] Allocating PCI resources starting at 40000000 (gap: 40000000:bffc0000)
[    0.000000] Booting paravirtualized kernel on bare hardware
[    0.000000] setup_percpu: NR_CPUS:64 nr_cpumask_bits:64 nr_cpu_ids:1 nr_node_ids:1
[    0.000000] PERCPU: Embedded 27 pages/cpu @ffff88003fc00000 s78848 r8192 d23552 u2097152
[    0.000000] Built 1 zonelists in Node order, mobility grouping on.  Total pages: 257894
[    0.000000] Policy zone: DMA32
[    0.000000] Kernel command line: LABEL=cirros-rootfs ro console=tty1 console=ttyS0
[    0.000000] PID hash table entries: 4096 (order: 3, 32768 bytes)
[    0.000000] Checking aperture...
[    0.000000] No AGP bridge found
[    0.000000] Memory: 1012228k/1048432k available (6576k kernel code, 452k absent, 35752k reserved, 6620k data, 928k init)
[    0.000000] SLUB: Genslabs=15, HWalign=64, Order=0-3, MinObjects=0, CPUs=1, Nodes=1
[    0.000000] Hierarchical RCU implementation.
[    0.000000]  RCU dyntick-idle grace-period acceleration is enabled.
[    0.000000] NR_IRQS:4352 nr_irqs:256 16
[    0.000000] Console: colour VGA+ 80x25
[    0.000000] console [tty1] enabled
[    0.000000] console [ttyS0] enabled
[    0.000000] allocated 8388608 bytes of page_cgroup
[    0.000000] please try 'cgroup_disable=memory' option if you don't want memory cgroups
[    0.000000] Fast TSC calibration failed
[    0.000000] TSC: Unable to calibrate against PIT
[    0.000000] TSC: using HPET reference calibration
[    0.000000] Detected 3406.301 MHz processor.
[    0.004860] Calibrating delay loop (skipped), value calculated using timer frequency.. 6812.60 BogoMIPS (lpj=13625204)
[    0.006083] pid_max: default: 32768 minimum: 301
[    0.008000] Security Framework initialized
[    0.008000] AppArmor: AppArmor initialized
[    0.008000] Yama: becoming mindful.
[    0.008000] Dentry cache hash table entries: 131072 (order: 8, 1048576 bytes)
[    0.008000] Inode-cache hash table entries: 65536 (order: 7, 524288 bytes)
[    0.008000] Mount-cache hash table entries: 256
[    0.008000] Initializing cgroup subsys cpuacct
[    0.008000] Initializing cgroup subsys memory
[    0.008000] Initializing cgroup subsys devices
[    0.008000] Initializing cgroup subsys freezer
[    0.008000] Initializing cgroup subsys blkio
[    0.008000] Initializing cgroup subsys perf_event
[    0.008000] mce: CPU supports 10 MCE banks
[    0.008000] SMP alternatives: switching to UP code
[    0.174444] Freeing SMP alternatives: 24k freed
[    0.176591] ACPI: Core revision 20110623
[    0.210111] ftrace: allocating 26610 entries in 105 pages
[    0.239385] ..TIMER: vector=0x30 apic1=0 pin1=2 apic2=-1 pin2=-1
[    0.282790] CPU0: AMD QEMU Virtual CPU version 2.5+ stepping 03
[    0.288017] Performance Events: Broken PMU hardware detected, using software events only.
[    0.295330] NMI watchdog disabled (cpu0): hardware events not enabled
[    0.296826] Brought up 1 CPUs
[    0.300166] Total of 1 processors activated (6812.60 BogoMIPS).
[    0.321890] devtmpfs: initialized
[    0.358993] EVM: security.selinux
[    0.360041] EVM: security.SMACK64
[    0.361003] EVM: security.capability
[    0.378225] print_constraints: dummy:
[    0.381810] RTC time: 10:31:43, date: 04/12/20
[    0.383864] NET: Registered protocol family 16
[    0.388967] ACPI: bus type pci registered
[    0.392344] PCI: Using configuration type 1 for base access
[    0.408968] bio: create slab <bio-0> at 0
[    0.412517] ACPI: Added _OSI(Module Device)
[    0.413104] ACPI: Added _OSI(Processor Device)
[    0.413547] ACPI: Added _OSI(3.0 _SCP Extensions)
[    0.414103] ACPI: Added _OSI(Processor Aggregator Device)
[    0.445980] ACPI: Interpreter enabled
[    0.446515] ACPI: (supports S0 S3 S4 S5)
[    0.448210] ACPI: Using IOAPIC for interrupt routing
[    0.491624] ACPI: No dock devices found.
[    0.492104] HEST: Table not found.
[    0.492537] PCI: Using host bridge windows from ACPI; if necessary, use "pci=nocrs" and report a bug
[    0.495707] ACPI: PCI Root Bridge [PCI0] (domain 0000 [bus 00-ff])
[    0.498098] pci_root PNP0A03:00: host bridge window [io  0x0000-0x0cf7]
[    0.498791] pci_root PNP0A03:00: host bridge window [io  0x0d00-0xffff]
[    0.500107] pci_root PNP0A03:00: host bridge window [mem 0x000a0000-0x000bffff]
[    0.500900] pci_root PNP0A03:00: host bridge window [mem 0x40000000-0xfebfffff]
[    0.501788] pci_root PNP0A03:00: host bridge window [mem 0x100000000-0x17fffffff]
[    0.512216] pci 0000:00:01.3: quirk: [io  0x0600-0x063f] claimed by PIIX4 ACPI
[    0.513311] pci 0000:00:01.3: quirk: [io  0x0700-0x070f] claimed by PIIX4 SMB
[    0.959753]  pci0000:00: Unable to request _OSC control (_OSC support mask: 0x1e)
[    0.992906] ACPI: PCI Interrupt Link [LNKA] (IRQs 5 *10 11)
[    0.996194] ACPI: PCI Interrupt Link [LNKB] (IRQs 5 *10 11)
[    0.997566] ACPI: PCI Interrupt Link [LNKC] (IRQs 5 10 *11)
[    0.999067] ACPI: PCI Interrupt Link [LNKD] (IRQs 5 10 *11)
[    1.000518] ACPI: PCI Interrupt Link [LNKS] (IRQs *9)
[    1.005613] vgaarb: device added: PCI:0000:00:02.0,decodes=io+mem,owns=io+mem,locks=none
[    1.006615] vgaarb: loaded
[    1.007000] vgaarb: bridge control possible 0000:00:02.0
[    1.009512] i2c-core: driver [aat2870] using legacy suspend method
[    1.010091] i2c-core: driver [aat2870] using legacy resume method
[    1.012203] SCSI subsystem initialized
[    1.014562] usbcore: registered new interface driver usbfs
[    1.015562] usbcore: registered new interface driver hub
[    1.016581] usbcore: registered new device driver usb
[    1.019062] PCI: Using ACPI for IRQ routing
[    1.026631] NetLabel: Initializing
[    1.027158] NetLabel:  domain hash size = 128
[    1.028096] NetLabel:  protocols = UNLABELED CIPSOv4
[    1.029738] NetLabel:  unlabeled traffic allowed by default
[    1.031239] HPET: 3 timers in total, 0 timers will be used for per-cpu timer
[    1.032351] hpet0: at MMIO 0xfed00000, IRQs 2, 8, 0
[    1.033335] hpet0: 3 comparators, 64-bit 100.000000 MHz counter
[    1.042112] Switching to clocksource hpet
[    1.150314] AppArmor: AppArmor Filesystem Enabled
[    1.151538] pnp: PnP ACPI init
[    1.152442] ACPI: bus type pnp registered
[    1.163761] pnp: PnP ACPI: found 10 devices
[    1.164288] ACPI: ACPI bus type pnp unregistered
[    1.209134] NET: Registered protocol family 2
[    1.237032] IP route cache hash table entries: 32768 (order: 6, 262144 bytes)
[    1.244839] TCP established hash table entries: 131072 (order: 9, 2097152 bytes)
[    1.247633] TCP bind hash table entries: 65536 (order: 8, 1048576 bytes)
[    1.249391] TCP: Hash tables configured (established 131072 bind 65536)
[    1.250110] TCP reno registered
[    1.250819] UDP hash table entries: 512 (order: 2, 16384 bytes)
[    1.251663] UDP-Lite hash table entries: 512 (order: 2, 16384 bytes)
[    1.253834] NET: Registered protocol family 1
[    1.254682] pci 0000:00:00.0: Limiting direct PCI/PCI transfers
[    1.255720] pci 0000:00:01.0: PIIX3: Enabling Passive Release
[    1.256757] pci 0000:00:01.0: Activating ISA DMA hang workarounds
[    1.260098] ACPI: PCI Interrupt Link [LNKD] enabled at IRQ 11
[    1.261265] pci 0000:00:01.2: PCI INT D -> Link[LNKD] -> GSI 11 (level, high) -> IRQ 11
[    1.263600] pci 0000:00:01.2: PCI INT D disabled
[    1.276704] Trying to unpack rootfs image as initramfs...
[    1.293009] audit: initializing netlink socket (disabled)
[    1.294944] type=2000 audit(1586687503.292:1): initialized
[    1.422352] HugeTLB registered 2 MB page size, pre-allocated 0 pages
[    1.467953] VFS: Disk quotas dquot_6.5.2
[    1.469013] Dquot-cache hash table entries: 512 (order 0, 4096 bytes)
[    1.485888] fuse init (API version 7.17)
[    1.487711] msgmni has been set to 1977
[    1.518919] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 253)
[    1.520246] io scheduler noop registered
[    1.521238] io scheduler deadline registered (default)
[    1.522068] io scheduler cfq registered
[    1.525526] pci_hotplug: PCI Hot Plug PCI Core version: 0.5
[    1.527043] pciehp: PCI Express Hot Plug Controller Driver version: 0.4
[    1.530777] input: Power Button as /devices/LNXSYSTM:00/LNXPWRBN:00/input/input0
[    1.532148] ACPI: Power Button [PWRF]
[    1.564296] ERST: Table is not found!
[    1.564918] GHES: HEST is not enabled!
[    1.573724] ACPI: PCI Interrupt Link [LNKC] enabled at IRQ 10
[    1.574343] virtio-pci 0000:00:03.0: PCI INT A -> Link[LNKC] -> GSI 10 (level, high) -> IRQ 10
[    1.577049] virtio-pci 0000:00:04.0: PCI INT A -> Link[LNKD] -> GSI 11 (level, high) -> IRQ 11
[    1.579185] ACPI: PCI Interrupt Link [LNKA] enabled at IRQ 10
[    1.579729] virtio-pci 0000:00:05.0: PCI INT A -> Link[LNKA] -> GSI 10 (level, high) -> IRQ 10
[    1.591270] Serial: 8250/16550 driver, 32 ports, IRQ sharing enabled
[    1.614716] serial8250: ttyS0 at I/O 0x3f8 (irq = 4) is a 16550A
[    1.689566] 00:05: ttyS0 at I/O 0x3f8 (irq = 4) is a 16550A
[    1.717040] Linux agpgart interface v0.103
[    1.758471] brd: module loaded
[    1.774927] loop: module loaded
[    1.798052]  vda: vda1
[    1.829162] scsi0 : ata_piix
[    1.838581] Freeing initrd memory: 3452k freed
[    1.842453] scsi1 : ata_piix
[    1.843708] ata1: PATA max MWDMA2 cmd 0x1f0 ctl 0x3f6 bmdma 0xc0c0 irq 14
[    1.844730] ata2: PATA max MWDMA2 cmd 0x170 ctl 0x376 bmdma 0xc0c8 irq 15
[    1.851810] Fixed MDIO Bus: probed
[    1.852573] tun: Universal TUN/TAP device driver, 1.6
[    1.853183] tun: (C) 1999-2004 Max Krasnyansky <maxk@qualcomm.com>
[    1.860949] PPP generic driver version 2.4.2
[    1.868600] ehci_hcd: USB 2.0 'Enhanced' Host Controller (EHCI) Driver
[    1.870101] ohci_hcd: USB 1.1 'Open' Host Controller (OHCI) Driver
[    1.871038] uhci_hcd: USB Universal Host Controller Interface driver
[    1.871932] uhci_hcd 0000:00:01.2: PCI INT D -> Link[LNKD] -> GSI 11 (level, high) -> IRQ 11
[    1.873475] uhci_hcd 0000:00:01.2: UHCI Host Controller
[    1.875279] uhci_hcd 0000:00:01.2: new USB bus registered, assigned bus number 1
[    1.876892] uhci_hcd 0000:00:01.2: irq 11, io base 0x0000c080
[    1.885012] hub 1-0:1.0: USB hub found
[    1.885825] hub 1-0:1.0: 2 ports detected
[    1.889757] usbcore: registered new interface driver libusual
[    1.891407] i8042: PNP: PS/2 Controller [PNP0303:KBD,PNP0f13:MOU] at 0x60,0x64 irq 1,12
[    1.895277] serio: i8042 KBD port at 0x60,0x64 irq 1
[    1.895978] serio: i8042 AUX port at 0x60,0x64 irq 12
[    1.898926] mousedev: PS/2 mouse device common for all mice
[    1.903204] input: AT Translated Set 2 keyboard as /devices/platform/i8042/serio0/input/input1
[    1.905577] rtc_cmos 00:01: RTC can wake from S4
[    1.909327] rtc_cmos 00:01: rtc core: registered rtc_cmos as rtc0
[    1.910691] rtc0: alarms up to one day, y3k, 114 bytes nvram, hpet irqs
[    1.912457] device-mapper: uevent: version 1.0.3
[    1.915897] device-mapper: ioctl: 4.22.0-ioctl (2011-10-19) initialised: dm-devel@redhat.com
[    1.917397] cpuidle: using governor ladder
[    1.918092] cpuidle: using governor menu
[    1.918622] EFI Variables Facility v0.08 2004-May-17
[    1.922232] TCP cubic registered
[    1.923818] NET: Registered protocol family 10
[    1.935588] NET: Registered protocol family 17
[    1.936455] Registering the dns_resolver key type
[    1.940316] registered taskstats version 1
[    2.107324]   Magic number: 0:426:527
[    2.108836] rtc_cmos 00:01: setting system clock to 2020-04-12 10:31:45 UTC (1586687505)
[    2.109760] powernow-k8: Processor cpuid 663 not supported
[    2.111870] BIOS EDD facility v0.16 2004-Jun-25, 0 devices found
[    2.112627] EDD information not available.
[    2.136059] Freeing unused kernel memory: 928k freed
[    2.157812] Write protecting the kernel read-only data: 12288k
[    2.200870] Freeing unused kernel memory: 1596k freed
[    2.234007] Freeing unused kernel memory: 1184k freed
[    2.265240] Refined TSC clocksource calibration: 3406.284 MHz.
[    2.266034] Switching to clocksource tsc

info: initramfs: up at 2.34
GROWROOT: CHANGED: partition=1 start=16065 old: size=64260 end=80325 new: size=10458315,end=10474380
info: initramfs loading root from /dev/vda1
info: /etc/init.d/rc.sysinit: up at 3.56
info: container: none
Starting logging: OK
modprobe: module virtio_blk not found in modules.dep
modprobe: module virtio_net not found in modules.dep
WARN: /etc/rc3.d/S10-load-modules failed
Initializing random number generator... done.
Starting acpid: OK
cirros-ds 'local' up at 5.13
no results found for mode=local. up 5.50. searched: nocloud configdrive ec2
Starting network...
udhcpc (v1.20.1) started
Sending discover...
Sending discover...
Sending discover...

```

</p>
</details>

8- Create [keypair](https://docs.openstack.org/python-openstackclient/pike/cli/command-objects/keypair.html)

`` openstack keypair create  --private-key  daskey.pem daskey ``
<details><summary>keypair output</summary>
<p>

```
+-------------+-------------------------------------------------+
| Field       | Value                                           |
+-------------+-------------------------------------------------+
| fingerprint | 23:6e:de:e7:a2:0d:46:33:36:25:09:12:a4:0e:df:5a |
| name        | daskey                                          |
| user_id     | 67078083cc2f4abfa0b1ae5985138f8b                |
+-------------+-------------------------------------------------+

```

</p>
</details>


``openstack keypair show --public-key daskey``

<details><summary>Reveal the public key</summary>
<p>


```

ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCr2KxJKPQ9YvsbNebP7zHKZcqxIJZHZ23ppJZJqJ0S/IbGHIkN37LFEQva1ItmDOcYGBLAgMZmAfZI/uHTfViTxaQjhDyrvolkVIv5hhUCMPc/GB8hnxPuL+Y5TqD2T4AAdFbcCB7OUaeco5dpECr43y5lO4HfK8EY/4hj0NRJxmEYMiMIIKYFV6hGJ2tfWuMS3Eg3QT6Ic4G9tbZyUZ86kwmLPOnaPbQW3VL9VxedhDIfKUSPNNwBEQRzBA2LqHzcIqPmOp4vG15IAyC+Vs8nP6zourxC/ixWgwKnW3tP+27GVC0WjCh8FMx/3eg7aJO4ilT7eWVHGpgTDtkNerxd Generated-by-Nova

```

</p>
</details>

9- Access the instance using the Key pair [i couldn't do it although i've created a security group, allowed port 22]

`ssh -i daskey.pem cirros@172.24.4.71`

ssh: connect to host 172.24.4.71 port 22: No route to host

10- Create new [network](https://docs.openstack.org/ocata/user-guide/cli-create-and-manage-networks.html)

`openstack network create yan`

<details><summary>network output</summary>
<p>


```python

+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        |                                      |
| created_at                | 2020-04-12T11:38:57Z                 |
| description               |                                      |
| dns_domain                | None                                 |
| id                        | 7d46621b-d51b-497a-bd5b-626c391b5a8a |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | False                                |
| is_vlan_transparent       | None                                 |
| mtu                       | 1450                                 |
| name                      | yan                                  |
| port_security_enabled     | True                                 |
| project_id                | 01bffc0d7ade4fe39eaa1352f80abf8a     |
| provider:network_type     | vxlan                                |
| provider:physical_network | None                                 |
| provider:segmentation_id  | 47                                   |
| qos_policy_id             | None                                 |
| revision_number           | 1                                    |
| router:external           | Internal                             |
| segments                  | None                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tags                      |                                      |
| updated_at                | 2020-04-12T11:38:58Z                 |
+---------------------------+--------------------------------------+


```

</p>
</details>

11-	Create new subnet for the network for ex. 10.110.152.0/24

`openstack subnet create subnet1 --network yan --subnet-range 10.110.152.0/24`

<details><summary>subnet output</summary>
<p>


```

+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| allocation_pools  | 10.110.152.2-10.110.152.254          |
| cidr              | 10.110.152.0/24                      |
| created_at        | 2020-04-12T11:41:22Z                 |
| description       |                                      |
| dns_nameservers   |                                      |
| enable_dhcp       | True                                 |
| gateway_ip        | 10.110.152.1                         |
| host_routes       |                                      |
| id                | b75e38c1-b9dc-4256-a319-77134f2726b3 |
| ip_version        | 4                                    |
| ipv6_address_mode | None                                 |
| ipv6_ra_mode      | None                                 |
| name              | subnet1                              |
| network_id        | 7d46621b-d51b-497a-bd5b-626c391b5a8a |
| project_id        | 01bffc0d7ade4fe39eaa1352f80abf8a     |
| revision_number   | 0                                    |
| segment_id        | None                                 |
| service_types     |                                      |
| subnetpool_id     | None                                 |
| tags              |                                      |
| updated_at        | 2020-04-12T11:41:22Z                 |
+-------------------+--------------------------------------+

```

</p>
</details>

12-	List all [volumes](https://docs.openstack.org/python-openstackclient/pike/cli/command-objects/volume.html) (cinder)

`openstack volume list`

<details><summary>Volume output</summary>
<p>


```

+--------------------------------------+------+-----------+------+----------------------------------------+
| ID                                   | Name | Status    | Size | Attached to                            |
+--------------------------------------+------+-----------+------+----------------------------------------+
| e8c6d850-4d88-4243-936a-9b42cd4cbdbf |      | in-use    |    5 | Attached to cli-instance2 on /dev/vda  |
| 6c166815-3dcd-4db1-95cb-216050af46b3 |      | in-use    |    5 | Attached to CentOS on /dev/vda         |
| 901b5033-8166-4e40-8679-1bbde548c693 |      | available |    1 |                                        |
+--------------------------------------+------+-----------+------+----------------------------------------+

```

</p>
</details>

13-	Create new volume of 1 GB

`openstack volume create --size 1 cervol`

<details><summary>volume output</summary>
<p>

```

+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| attachments         | []                                   |
| availability_zone   | nova                                 |
| bootable            | false                                |
| consistencygroup_id | None                                 |
| created_at          | 2020-04-12T11:45:22.000000           |
| description         | None                                 |
| encrypted           | False                                |
| id                  | c2f62c57-a131-4fdc-bb82-f5a74af11196 |
| migration_status    | None                                 |
| multiattach         | False                                |
| name                | cervol                               |
| properties          |                                      |
| replication_status  | None                                 |
| size                | 1                                    |
| snapshot_id         | None                                 |
| source_volid        | None                                 |
| status              | creating                             |
| type                | lvmdriver-1                          |
| updated_at          | None                                 |
| user_id             | 67078083cc2f4abfa0b1ae5985138f8b     |
+---------------------+--------------------------------------+

```

</p>
</details>

14- List all containers (swift) [Fails]

`openstack container list`

public endpoint for object-store service in RegionOne region not found
