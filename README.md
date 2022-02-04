# sw-raid configuration with RHOSP-16.2.1.GA


```diff
- scenario 1: Deploy all nodes with sw-raid
```

* Deploy undercloud as per official documentation. [1] https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/16.2/html-single/director_installation_and_usage/index
* Registering nodes for the overcloud

```
(undercloud) [stack@dell430-33-undercloud-0-16-2 ~]$ cat instackenv.json
```

```
{
"nodes": [
{
"pm_user": "admin",
"mac": ["52:54:00:d2:f7:5c"],
"pm_type": "pxe_ipmitool",
"pm_port": "6230",
"pm_password": "redhat",
"pm_addr": "192.168.24.250",
"capabilities" : "node:ctrl-0,boot_option:local",
"name": "overcloud-controller-0"
},

{
"pm_user": "admin",
"mac": ["52:54:00:82:08:75"],
"pm_type": "pxe_ipmitool",
"pm_port": "6231",
"pm_password": "redhat",
"pm_addr": "192.168.24.250",
"capabilities" : "node:cmpt-0,boot_option:local",
"name": "overcloud-compute-0"
}

]
}
```

* Import the node

```
openstack overcloud node import /home/stack/instackenv.json
```

* Introspect the node

```
openstack overcloud node introspect --all-manageable --provide
```

```
+--------------------------------------+-----------------------------+--------------------------------------+-------------+--------------------+-------------+
| UUID                                 | Name                        | Instance UUID                        | Power State | Provisioning State | Maintenance |
+--------------------------------------+-----------------------------+--------------------------------------+-------------+--------------------+-------------+
| 49fa80a6-64a5-4f5b-b4da-857d6be5a1a0 | overcloud-controller-0      | 										| power off    | available         | False       |
| 8b6a0233-7c62-4794-be6d-d6c8d6dfa94f | overcloud-compute-0         |   									| power off    | available         | False       |
+--------------------------------------+-----------------------------+--------------------------------------+-------------+--------------------+-------------+
```

* Create a customize whole disk for sw-raid [Mendatory Steps]

```
[root@localhost ~]# cat hardened_image_gen.sh

export DIB_LOCAL_IMAGE=/root/rhel8.4-guest.qcow2
export REG_METHOD=portal
export REG_USER=rhn-support-nchandek
export REG_PASSWORD=**********
export REG_RELEASE="8.4"
export REG_POOL_ID=8a85f9a17af8e616017b0d59c6476885
export REG_REPOS="rhel-8-for-x86_64-baseos-eus-rpms \
  rhel-8-for-x86_64-appstream-eus-rpms \
  rhel-8-for-x86_64-highavailability-eus-rpms \
  ansible-2.9-for-rhel-8-x86_64-rpms \
  openstack-16.2-for-rhel-8-x86_64-rpms \
  fast-datapath-for-rhel-8-x86_64-rpms \
  rhceph-4-tools-for-rhel-8-x86_64-rpms "
export DIB_BLOCK_DEVICE_CONFIG='''
- local_loop:
    name: image0
- partitioning:
    base: image0
    label: mbr
    partitions:
      - name: root
        flags: [ boot,primary ]
        size: 10G
        mkfs:
          type: xfs
          mount:
            mount_point: /
            fstab:
              options: "defaults"
              fsck-passno: 1
      - name: var
        flags: [primary ]
        size: 10G
        mkfs:
          type: xfs
          mount:
            mount_point: /var
            fstab:
              options: "defaults"
              fsck-passno: 1
'''
[root@localhost ~]#
```

* Export above variable in the shell and then create a image out of it,
* Note size of the disk is important.

```
cp /usr/share/openstack-tripleo-common/image-yaml/overcloud-hardened-images-python3.yaml .
sed -i 's/40/26/g' overcloud-hardened-images-python3.yaml
```
```
openstack overcloud image build \
--image-name overcloud-hardened-full \
--config-file overcloud-hardened-images-python3.yaml \
--config-file /usr/share/openstack-tripleo-common/image-yaml/overcloud-hardened-images-rhel8.yaml
```

```
cd images/harden/
cp ~/images/ironic-python-agent.initramfs .
cp ~/images/ironic-python-agent.kernel .
mv overcloud-hardened-full.qcow2 overcloud-full.qcow2
```

* Upload the image to glance

```
openstack overcloud image upload --image-path /home/stack/images/harden/ --whole-disk
```

```diff
(undercloud) [stack@dell430-33-undercloud-0-16-2 ~]$ openstack image list
|--------------------------------------+---------------------------------+--------+
| ID                                   | Name                            | Status |
|--------------------------------------+---------------------------------+--------+
+ 66742683-9d7c-42a4-bb30-df4601912247 | overcloud-full                  | active |
| 05f0270b-8199-4b18-ad7e-b200c64285c2 | overcloud-full-initrd           | active |
| 2e052cb4-66a4-429f-81fb-fabc17040120 | overcloud-full-vmlinuz          | active |
|--------------------------------------+---------------------------------+--------+
(undercloud) [stack@dell430-33-undercloud-0-16-2 ~]$
```


```
for node in 49fa80a6-64a5-4f5b-b4da-857d6be5a1a0 8b6a0233-7c62-4794-be6d-d6c8d6dfa94f ; \
do openstack overcloud node configure $node ; \
done
```


##  Sw-Raid** configuration started from here:

* Required ironic configuration changes on **undercloud** node.

```
grep  enabled_raid_interfaces /var/lib/config-data/puppet-generated/ironic/etc/ironic/ironic.conf


enabled_raid_interfaces=**agent**,idrac,no-raid
```

* Restart ironic_conductor container:

```
sudo podman restart ironic_conductor
```

* Set sw-raid interface on the node.

```
openstack baremetal node set --raid-interface agent 49fa80a6-64a5-4f5b-b4da-857d6be5a1a0
openstack baremetal node set --raid-interface agent 8b6a0233-7c62-4794-be6d-d6c8d6dfa94f
```

* Manage the node, for further configuration.

```
openstack baremetal node manage  49fa80a6-64a5-4f5b-b4da-857d6be5a1a0
openstack baremetal node manage  8b6a0233-7c62-4794-be6d-d6c8d6dfa94f
```

* Set the target configuration [sw raid]

```
openstack baremetal node set   --target-raid-config /home/stack/sw-raid/target.yaml 49fa80a6-64a5-4f5b-b4da-857d6be5a1a0
openstack baremetal node set   --target-raid-config /home/stack/sw-raid/target.yaml 8b6a0233-7c62-4794-be6d-d6c8d6dfa94f
```

* cat  /home/stack/sw-raid/target.yaml

```
{
   "logical_disks": [
     {
       "size_gb": "MAX",
       "raid_level": "1",
       "controller": "software",
       "is_root_volume": true,
       "disk_type": "hdd"
     }
   ]
}
```

* Clean the node before deployment,

```
openstack baremetal node clean  --clean-steps /home/stack/sw-raid/cleanstate.yaml 49fa80a6-64a5-4f5b-b4da-857d6be5a1a0
openstack baremetal node clean  --clean-steps /home/stack/sw-raid/cleanstate.yaml 8b6a0233-7c62-4794-be6d-d6c8d6dfa94f
```

* Move node from manage to available. [provide]

```
openstack baremetal node provide 49fa80a6-64a5-4f5b-b4da-857d6be5a1a0
openstack baremetal node provide 8b6a0233-7c62-4794-be6d-d6c8d6dfa94f
```

* Set the flavor as well,

```
openstack flavor set \
--property "capabilities:raid_level"="1"  \
0f8deb91-2a64-4025-a75e-f70e28e89491
```

* Now at this moment the nodes should be in **available** state. [note down in the previous step we manually moved nodes to manage state]

```
+--------------------------------------+-----------------------------+--------------------------------------+-------------+--------------------+-------------+
| UUID                                 | Name                        | Instance UUID                        | Power State | Provisioning State | Maintenance |
+--------------------------------------+-----------------------------+--------------------------------------+-------------+--------------------+-------------+
| 49fa80a6-64a5-4f5b-b4da-857d6be5a1a0 | overcloud-controller-0      | 										| power off    | available         | False       |
| 8b6a0233-7c62-4794-be6d-d6c8d6dfa94f | overcloud-compute-0         |   									| power off    | available         | False       |
+--------------------------------------+-----------------------------+--------------------------------------+-------------+--------------------+-------------+
```

* Deployment command:

```
(undercloud) [stack@dell430-33-undercloud-0-16-2 ~]$ cat deploy1.sh
#!/bin/bash
time openstack overcloud deploy --templates /home/stack/templates/rendered/ \
-r /home/stack/templates/rendered/roles_data.yaml \
-e /home/stack/templates/rendered/environments/network-environment.yaml \
-e /home/stack/templates/rendered/environments/network-isolation.yaml \
-e /home/stack/containers-prepare-parameter.yaml \
-e /home/stack/templates/extra/scheduler_hints_env.yaml \
-e /home/stack/templates/extra/node-info.yaml \
--libvirt-type qemu --debug --log-file /tmp/install_overcloud.log --timeout 90

(undercloud) [stack@dell430-33-undercloud-0-16-2 ~]$
```


* Deployed environment:

```
+--------------------------------------+-------------------------+--------+------------------------+---------------------------------+-----------+
| ID                                   | Name                    | Status | Networks               | Image                           | Flavor    |
+--------------------------------------+-------------------------+--------+------------------------+---------------------------------+-----------+
| e1b17fac-19e7-4487-a5b3-9540f6f68f3b | overcloud-controller-0  | ACTIVE | ctlplane=192.168.24.13 | overcloud-full 				 | baremetal |
| 78a4605c-3ab3-49a8-91ac-c9ebccaeeb19 | overcloud-novacompute-0 | ACTIVE | ctlplane=192.168.24.21 | overcloud-full 				 | baremetal |
+--------------------------------------+-------------------------+--------+------------------------+---------------------------------+-----------+
```


* Verification:

```diff
cat /proc/mdstat
Personalities : [raid1]
+ md127 : active raid1 vdb1[1] vda1[0]
      62878720 blocks super 1.2 [2/2] [UU]

unused devices: <none>
```

```diff
mdadm --detail /dev/md127
/dev/md127:
           Version : 1.2
     Creation Time : Tue Feb  1 14:35:03 2022
+        Raid Level : raid1
        Array Size : 62878720 (59.97 GiB 64.39 GB)
     Used Dev Size : 62878720 (59.97 GiB 64.39 GB)
      Raid Devices : 2
     Total Devices : 2
       Persistence : Superblock is persistent

       Update Time : Tue Feb  1 16:34:26 2022
            + State : clean
    + Active Devices : 2
   Working Devices : 2
    Failed Devices : 0
     Spare Devices : 0

+ Consistency Policy : resync

              Name : host-192-168-24-24:0
              UUID : b0ba6b96:bcdba009:3288f603:7f8c8625
            Events : 175

    Number   Major   Minor   RaidDevice State
+        0     252        1        0      active sync   /dev/vda1
+        1     252       17        1      active sync   /dev/vdb1
```

```
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        3.8G     0  3.8G   0% /dev
tmpfs           3.8G   84K  3.8G   1% /dev/shm
tmpfs           3.8G  1.8M  3.8G   1% /run
tmpfs           3.8G     0  3.8G   0% /sys/fs/cgroup
/dev/md127p1    9.4G  2.4G  7.0G  26% /
/dev/md127p2    9.4G  4.8G  4.6G  51% /var
tmpfs           777M     0  777M   0% /run/user/1000
```

```
NAME          MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
vda           252:0    0   60G  0 disk
└─vda1        252:1    0   60G  0 part
  └─md127       9:127  0   60G  0 raid1
    ├─md127p1 259:0    0  9.3G  0 md    /
    ├─md127p2 259:1    0  9.3G  0 md    /var
    └─md127p3 259:2    0   64M  0 md
vdb           252:16   0   60G  0 disk
└─vdb1        252:17   0   60G  0 part
  └─md127       9:127  0   60G  0 raid1
    ├─md127p1 259:0    0  9.3G  0 md    /
    ├─md127p2 259:1    0  9.3G  0 md    /var
    └─md127p3 259:2    0   64M  0 md
```


```diff
- scenario 2: Enable sw-raid on already deployed environment. [with new role] - SCALEING COMPUTE NODE WITH SW-RAID.
```

* RHOSP-16.2.1.GA is already deployed with some set of controller's and compute's

```
Red Hat OpenStack Platform release 16.2.1 GA (Train)
```


```
+--------------------------------------+-----------------------------+--------------------------------------+-------------+--------------------+-------------+
| UUID                                 | Name                        | Instance UUID                        | Power State | Provisioning State | Maintenance |
+--------------------------------------+-----------------------------+--------------------------------------+-------------+--------------------+-------------+
| 49fa80a6-64a5-4f5b-b4da-857d6be5a1a0 | overcloud-controller-0      | d8e6e27b-baf6-4b32-8cb7-88184de3f290 | power on    | active             | False       |
| 8b6a0233-7c62-4794-be6d-d6c8d6dfa94f | overcloud-compute-0         | 7a888f19-1bce-445b-a76f-ab4bde684d0c | power on    | active             | False       |

+--------------------------------------+---------------------------+--------+------------------------+-------------------------+-----------+
| ID                                   | Name                      | Status | Networks               | Image                   | Flavor    |
+--------------------------------------+---------------------------+--------+------------------------+-------------------------+-----------+
| d8e6e27b-baf6-4b32-8cb7-88184de3f290 | overcloud-controller-0    | ACTIVE | ctlplane=192.168.24.9  | overcloud-full          | baremetal |
| 7a888f19-1bce-445b-a76f-ab4bde684d0c | overcloud-novacompute-0   | ACTIVE | ctlplane=192.168.24.17 | overcloud-full          | baremetal |
+--------------------------------------+---------------------------+--------+------------------------+-------------------------+-----------+
+--------------------------------------+------------+----------------------------------+-----------------+----------------------+----------------------+
| ID                                   | Stack Name | Project                          | Stack Status    | Creation Time        | Updated Time         |
+--------------------------------------+------------+----------------------------------+-----------------+----------------------+----------------------+
| 05375109-3772-4a6d-80c4-1e5072619a67 | overcloud  | 398ed9b97d9d424abe6a627dae1c61c0 | UPDATE_COMPLETE | 2022-02-02T07:23:25Z | 2022-02-03T16:11:48Z |
+--------------------------------------+------------+----------------------------------+-----------------+----------------------+----------------------+
```

### **sw-raid** configuration  starts here.

* Required ironical configuration changes on **undercloud** node.

```
grep  enabled_raid_interfaces /var/lib/config-data/puppet-generated/ironic/etc/ironic/ironic.conf


enabled_raid_interfaces=**agent**,idrac,no-raid
```

* Restart ironic_conductor container:

```
sudo podman restart ironic_conductor
```

* Import the node first:

```diff
(undercloud) [stack@dell430-33-undercloud-0-16-2 ~]$ cat instackenv2.json
{
"nodes": [

{
"pm_user": "admin",
"mac": ["52:54:00:96:47:ef"],
"pm_type": "pxe_ipmitool",
"pm_port": "6232",
"pm_password": "redhat",
"pm_addr": "192.168.24.250",
+ "capabilities" : "node:swcmpt-0,boot_option:local",
"name": "overcloud-sw-raid-compute-0"
}

]
}
(undercloud) [stack@dell430-33-undercloud-0-16-2 ~]$
```

* Import the node

```
openstack overcloud node import /home/stack/instackenv2.json
```

* Introspect the node.

```
openstack overcloud node introspect --provide 5a6342b6-4386-4633-b558-ba4900789879
```

* Set sw-raid agent on the node.

```
openstack baremetal node set --raid-interface agent  5a6342b6-4386-4633-b558-ba4900789879
```

* move the node into manage, for further configuration.

```
openstack baremetal node manage  5a6342b6-4386-4633-b558-ba4900789879
```

* Set the target configuration [sw raid]

```
openstack baremetal node set   --target-raid-config /home/stack/sw-raid/target.yaml  5a6342b6-4386-4633-b558-ba4900789879
```

* cat  /home/stack/sw-raid/target.yaml

```
{
   "logical_disks": [
     {
       "size_gb": "MAX",
       "raid_level": "1",
       "controller": "software",
       "is_root_volume": true,
       "disk_type": "hdd"
     }
   ]
}
```

* Clean the node before deployment, [During this step the nodes will turn on and will start cleaning o the disk]

```
openstack baremetal node clean  --clean-steps /home/stack/sw-raid/cleanstate.yaml 5a6342b6-4386-4633-b558-ba4900789879
```

* Move node from manage to available. [provide]

```
openstack baremetal node provide 5a6342b6-4386-4633-b558-ba4900789879
```

* Set the flavor as well,

```
openstack flavor set \
--property "capabilities:raid_level"="1"  \
baremetal
```

* Upload the image: [New role]

```diff
(undercloud) [stack@dell430-33-undercloud-0-16-2 harden]$ ll -lhrt
total 1.7G
-rw-r--r--. 1 stack stack 426M Jan 27 13:44 ironic-python-agent.initramfs
-rwxr-xr-x. 1 stack stack 9.6M Jan 27 13:44 ironic-python-agent.kernel
+ -rw-rw-r--. 1 stack stack 1.3G Feb  1 06:23 overcloud-full-hardened.qcow2
(undercloud) [stack@dell430-33-undercloud-0-16-2 harden]$
```

* Upload the image ot the glance.

```
openstack overcloud image upload --os-image-name overcloud-full-hardened.qcow2 --whole-disk
```

```diff
(undercloud) [stack@dell430-33-undercloud-0-16-2 harden]$ openstack image list
|--------------------------------------+-------------------------+--------+
| ID                                   | Name                    | Status |
|--------------------------------------+-------------------------+--------+
| 31e4c942-e143-4039-b1de-f1da588c64ad | overcloud-full          | active |
+ 9c9d1321-2a3a-4c6c-9f14-27f29f4885c5 | overcloud-full-hardened | active |
| 05f0270b-8199-4b18-ad7e-b200c64285c2 | overcloud-full-initrd   | active |
| 2e052cb4-66a4-429f-81fb-fabc17040120 | overcloud-full-vmlinuz  | active |
|--------------------------------------+-------------------------+--------+
(undercloud) [stack@dell430-33-undercloud-0-16-2 harden]$
```

* Deployment command:

```diff
(undercloud) [stack@dell430-33-undercloud-0-16-2 ~]$ cat deploy1.sh
#!/bin/bash
time openstack overcloud deploy --templates /usr/share/openstack-tripleo-heat-templates/ \
+-r /home/stack/templates/rendered/roles_data.yaml \
-e /home/stack/templates/rendered/environments/network-environment.yaml \
-e /usr/share/openstack-tripleo-heat-templates/environments/network-isolation.yaml \
-e /home/stack/containers-prepare-parameter.yaml \
-e /home/stack/templates/extra/scheduler_hints_env.yaml \
-e /home/stack/templates/extra/node-info.yaml \
--libvirt-type qemu --debug --log-file /tmp/install_overcloud.log --timeout 90

(undercloud) [stack@dell430-33-undercloud-0-16-2 ~]$
```

```diff
###############################################################################
# Role: ComputeSwRaid                                                         #
###############################################################################
- name: ComputeSwRaid
  description: |
    Basic Compute Node role
  CountDefault: 0
  # Create external Neutron bridge (unset if using ML2/OVS without DVR)
  tags:
    - external_bridge
  networks:
    InternalApi:
      subnet: internal_api_subnet
    Tenant:
      subnet: tenant_subnet
    Storage:
      subnet: storage_subnet
  HostnameFormatDefault: '%stackname%-swnovacompute-%index%'
+  ImageDefault: overcloud-full-hardened
  RoleParametersDefault:
    TunedProfileName: "virtual-host"
  # Deprecated & backward-compatible values (FIXME: Make parameters consistent)
  # Set uses_deprecated_params to True if any deprecated params are used.
  # These deprecated_params only need to be used for existing roles and not for
  # composable roles.
  uses_deprecated_params: True
  deprecated_param_image: 'NovaImage'
  deprecated_param_extraconfig: 'NovaComputeExtraConfig'
  deprecated_param_metadata: 'NovaComputeServerMetadata'
  deprecated_param_scheduler_hints: 'NovaComputeSchedulerHints'
  deprecated_param_ips: 'NovaComputeIPs'
  deprecated_server_resource_name: 'NovaCompute'
  deprecated_nic_config_name: 'compute.yaml'
  update_serial: 25
  ServicesDefault:
    - OS::TripleO::Services::Aide
    - OS::TripleO::Services::AuditD
    - OS::TripleO::Services::BootParams
    - OS::TripleO::Services::CACerts
    - OS::TripleO::Services::CephClient
    - OS::TripleO::Services::CephExternal
    - OS::TripleO::Services::CertmongerUser
    - OS::TripleO::Services::Collectd
    - OS::TripleO::Services::ComputeCeilometerAgent
    - OS::TripleO::Services::ComputeNeutronCorePlugin
    - OS::TripleO::Services::ComputeNeutronL3Agent
    - OS::TripleO::Services::ComputeNeutronMetadataAgent
    - OS::TripleO::Services::ComputeNeutronOvsAgent
    - OS::TripleO::Services::Docker
    - OS::TripleO::Services::IpaClient
    - OS::TripleO::Services::Ipsec
    - OS::TripleO::Services::Iscsid
    - OS::TripleO::Services::Kernel
    - OS::TripleO::Services::LoginDefs
    - OS::TripleO::Services::MetricsQdr
    - OS::TripleO::Services::Multipathd
    - OS::TripleO::Services::MySQLClient
    - OS::TripleO::Services::NeutronBgpVpnBagpipe
    - OS::TripleO::Services::NeutronLinuxbridgeAgent
    - OS::TripleO::Services::NeutronVppAgent
    - OS::TripleO::Services::NovaAZConfig
    - OS::TripleO::Services::NovaCompute
    - OS::TripleO::Services::NovaLibvirt
    - OS::TripleO::Services::NovaLibvirtGuests
    - OS::TripleO::Services::NovaMigrationTarget
    - OS::TripleO::Services::ContainersLogrotateCrond
    - OS::TripleO::Services::Podman
    - OS::TripleO::Services::Rear
    - OS::TripleO::Services::Rhsm
    - OS::TripleO::Services::Rsyslog
    - OS::TripleO::Services::RsyslogSidecar
    - OS::TripleO::Services::Securetty
    - OS::TripleO::Services::Snmp
    - OS::TripleO::Services::Sshd
    - OS::TripleO::Services::Timesync
    - OS::TripleO::Services::Timezone
    - OS::TripleO::Services::TripleoFirewall
    - OS::TripleO::Services::TripleoPackages
    - OS::TripleO::Services::Tuned
    - OS::TripleO::Services::Vpp
    - OS::TripleO::Services::OVNController
    - OS::TripleO::Services::OVNMetadataAgent
###############################################################################
```

* Scheduler hints:

```diff
(undercloud) [stack@dell430-33-undercloud-0-16-2 ~]$ cat /home/stack/templates/extra/scheduler_hints_env.yaml
parameter_defaults:
  ControllerSchedulerHints:
    'capabilities:node': 'ctrl-%index%'

  ComputeSchedulerHints:
    'capabilities:node': 'cmpt-%index%'

+  ComputeSwRaidSchedulerHints:
+    'capabilities:node': 'swcmpt-%index%'
(undercloud) [stack@dell430-33-undercloud-0-16-2 ~]$
```

* Node info :

```diff
(undercloud) [stack@dell430-33-undercloud-0-16-2 ~]$ cat /home/stack/templates/extra/node-info.yaml
parameter_defaults:
  OvercloudControllerFlavor: baremetal
  OvercloudComputeFlavor: baremetal
+  OvercloudComputeSwRaidFlavor: baremetal

  ControllerCount: 1
  ComputeCount: 1
+  ComputeSwRaidCount: 1

  NtpServer: 192.168.24.1
  NeutronEnableDVR: false
(undercloud) [stack@dell430-33-undercloud-0-16-2 ~]$
```

* Deployed one:

```diff
|--------------------------------------+---------------------------+--------+------------------------+-------------------------+-----------+
| ID                                   | Name                      | Status | Networks               | Image                   | Flavor    |
--------------------------------------+---------------------------+--------+------------------------+-------------------------+-----------+
+ 4835c407-445c-41d4-9506-cdc6030ba12f | overcloud-swnovacompute-0 | ACTIVE | ctlplane=192.168.24.23 | overcloud-full-hardened | baremetal |
| d8e6e27b-baf6-4b32-8cb7-88184de3f290 | overcloud-controller-0    | ACTIVE | ctlplane=192.168.24.9  | overcloud-full          | baremetal |
| 7a888f19-1bce-445b-a76f-ab4bde684d0c | overcloud-novacompute-0   | ACTIVE | ctlplane=192.168.24.17 | overcloud-full          | baremetal |
|--------------------------------------+---------------------------+--------+------------------------+-------------------------+-----------+
```

```
[root@overcloud-swnovacompute-0 ~]# cat /proc/mdstat
Personalities : [raid1]
md127 : active raid1 vdb1[1] vda1[0]
      62878720 blocks super 1.2 [2/2] [UU]

unused devices: <none>
[root@overcloud-swnovacompute-0 ~]#
```

```
[root@overcloud-swnovacompute-0 ~]# mdadm --detail /dev/md127
/dev/md127:
           Version : 1.2
     Creation Time : Wed Feb  2 09:35:23 2022
        Raid Level : raid1
        Array Size : 62878720 (59.97 GiB 64.39 GB)
     Used Dev Size : 62878720 (59.97 GiB 64.39 GB)
      Raid Devices : 2
     Total Devices : 2
       Persistence : Superblock is persistent

       Update Time : Thu Feb  3 17:03:32 2022
             State : clean
    Active Devices : 2
   Working Devices : 2
    Failed Devices : 0
     Spare Devices : 0

Consistency Policy : resync

              Name : localhost.localdomain:0
              UUID : 078b1d94:543da54c:df879980:a0c81fa8
            Events : 257

    Number   Major   Minor   RaidDevice State
       0     252        1        0      active sync   /dev/vda1
       1     252       17        1      active sync   /dev/vdb1
[root@overcloud-swnovacompute-0 ~]#
```

```
[root@overcloud-swnovacompute-0 ~]# lsblk
NAME          MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
vda           252:0    0   60G  0 disk
└─vda1        252:1    0   60G  0 part
  └─md127       9:127  0   60G  0 raid1
    ├─md127p1 259:0    0  9.3G  0 md    /
    ├─md127p2 259:1    0  9.3G  0 md    /var
    └─md127p3 259:2    0   64M  0 md
vdb           252:16   0   60G  0 disk
└─vdb1        252:17   0   60G  0 part
  └─md127       9:127  0   60G  0 raid1
    ├─md127p1 259:0    0  9.3G  0 md    /
    ├─md127p2 259:1    0  9.3G  0 md    /var
    └─md127p3 259:2    0   64M  0 md
[root@overcloud-swnovacompute-0 ~]#
```


```
[root@overcloud-swnovacompute-0 ~]# blkid
/dev/vda1: UUID="078b1d94-543d-a54c-df87-9980a0c81fa8" UUID_SUB="963b0a43-163c-9ac7-43ac-02f7ff953d45" LABEL="localhost.localdomain:0" TYPE="linux_raid_member" PARTUUID="248342fe-01"
/dev/vdb1: UUID="078b1d94-543d-a54c-df87-9980a0c81fa8" UUID_SUB="de1ecb8d-fb4c-1dfc-8189-eb723088bc03" LABEL="localhost.localdomain:0" TYPE="linux_raid_member" PARTUUID="5cc87a30-01"
/dev/md127p1: LABEL="img-rootfs" UUID="82e663f1-7482-4fd2-86ef-d9592f314c02" BLOCK_SIZE="512" TYPE="xfs" PARTUUID="df2e5cf4-01"
/dev/md127p2: LABEL="mkfs_var" UUID="5a8c29c7-3911-4ecd-82dd-2cbf98ce7c01" BLOCK_SIZE="512" TYPE="xfs" PARTUUID="df2e5cf4-02"
/dev/md127p3: BLOCK_SIZE="2048" UUID="2022-02-02-05-09-28-00" LABEL="config-2" TYPE="iso9660" PARTUUID="df2e5cf4-03"
/dev/md127: PTUUID="df2e5cf4" PTTYPE="dos"
[root@overcloud-swnovacompute-0 ~]#
```
