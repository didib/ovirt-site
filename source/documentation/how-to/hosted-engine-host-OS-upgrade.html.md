---
title: Hosted Engine host operating system upgrade Howto
category: howto
---

### Hosted Engine host operating system upgrade Howto

### Summary

How to upgrade a hosted-engine setup on el6 hosts to el7 hosts.

### Background

oVirt 3.6 does not support installing new hosts as el6.

oVirt 3.5 did allow that, including for hosted-engine.

Upgrading hosts from el6 to el7 should be done when using 3.5.

Trying to "upgrade" a hosted-engine cluster from el6 to el7 by installing a new hosted-engine host as el7 and deploy it as additional hosted-engine host currently fails. It might be possible to make it work but that is not planned for now.

### Assumptions

As existing oVirt 3.5 hosted-engine setup, consisting of:

*   At least two hypervisors (hosts) with el6 (RHEL 6 or CentOS 6), one of which can be evacuated during the process
*   A hosted-engine engine vm with whatever OS (irrelevant for current document)

### Upgrade process

#### 1. Upgrade hosts to el7

1. Move one host to maintenance in the web admin interface.
Wait until all the VMs there were on it were migrated to other hosts and it's displayed as being in local maintenance in:

         # hosted-engine --vm-status

2. Reinstall the host as el7

3. Create a new cluster in the engine named 'el7'

4. Add relevant 3.5 repos

5. Install and deploy:

         # yum install -y ovirt-hosted-engine-setup
         # hosted-engine --deploy

You will not be able to supply host id '1' even if reinstalling the actual first host. 

TODO

Assuming you're using ovirt RPMs, you should start with install and deploy:

         # yum install ovirt-hosted-engine-setup
         # hosted-engine --deploy

During the deployment you'll be asked for input on host name, storage path and other relevant information. The installer will configure the system and run an empty VM. Access the VM and install an OS:

         [ INFO  ] Creating VM
                 ...
                 ...
                 Please install the OS on the VM.
                 When the installation is completed reboot or shutdown the VM: the system will wait until then
                 Has the OS installation been completed successfully?

After completing the OS installation on the VM, return to the host and continue. The installer on the host will sync with the VM and ask for the engine to be installed on the new VM:

        [ INFO  ] Creating VM
                 ...
                 ...
                 Please install the engine in the VM, hit enter when finished.

On the VM:

         # yum install ovirt-engine
         # engine-setup

When the engine-setup has completed on the VM, return to the host and complete the configuration. Your hosted engine VM is up and running!

**Notes:**

*   Remember to setup the same hostname you specified as FQDN during deploy while you're setting up the engine on the VM.
*   Although hosted-engine and engine-setup use different wording for the admin password ("'admin@internal' user password" vs "Engine admin password"), they are asking for the same thing. If you enter different passwords, the hosted-engine setup will fail.
*   If you want to install ovirt-engine-dwh and ovirt-engine-reports, or update the engine after the deployment is completed, remember that you need to set the system in global maintenance using
        # hosted-engine --set-maintenance --mode=global

    because the engine service must be stopped during setup / upgrade operations.

#### **Restarting form a partially deployed system**

If, for any reason, the deployment process breaks before its end, you can try to continue from where it got interrupted without the need to restart from scratch.

*   Closing up, hosted-engine --deploy always generates an answerfile. You could simply try restart the deployment process with that answerfile:

      hosted-engine --deploy --config-append=/var/lib/ovirt-hosted-engine-setup/answers/answers-20150402165233.conf

*   it should start the VM from CD-ROM using the same storage device for it, but if you have already installed the OS you could simply poweroff it and select: (1) Continue setup - VM installation is complete
*   at that point it should boot the previously engine VM from the storage device and you are ready to conclude it
*   if this doesn't work you have to cleanup the storage device and restart from scratch

### **Migrate existing setup to a VM**

Moving an existing setup into a VM is similar to a fresh install, but instead of running a fresh engine-setup inside the VM, we restore there a backup of the existing engine. For full details see [Migrate_to_Hosted_Engine](Migrate_to_Hosted_Engine)

### **Installing additional nodes**

Here is an example of a deployment on an additional host:

         # yum install ovirt-hosted-engine-setup
         # hosted-engine --deploy

Once storage path is given, the installer will identify this is an additional host, and will change the flow accordingly:

         The specified storage location already contains a data domain. Is this an additional host setup (Yes, No)[Yes]? yes
         [ INFO  ] Installing on additional host
                 Please specify the Host ID [Must be integer, default: 2]:

As with the first node, this will take you to the process completion.

**Notes**

*   Remember to use the same storage path you used on first host.

### **Maintaining the setup**

The HA services have two maintenance types for different tasks.

#### **Global maintenance**

Main use is to allow the administrator to start/stop/modify the engine VM without any worry of interference from the HA agents.
In order to maintain the engine VM, use:

         # hosted-engine --set-maintenance --mode=global

To resume HA functionality, use:

         # hosted-engine --set-maintenance --mode=none

#### **Local maintenance**

Main use is to allow the administrator to maintain one or more hosts. Note that if you have only 2 nodes and one is in maintenance,
there is only one host available to run the engine VM. The way to maintain a host is by using:

         # hosted-engine --set-maintenance --mode=local

To resume HA functionality, use:

         # hosted-engine --set-maintenance --mode=none

### **Upgrade Hosted Engine**

Assuming you have already deployed Hosted Engine on your hosts and running the Hosted Engine VM, having the same oVirt version both on hosts and Hosted Engine VM.

1.  Set hosted engine maintenance mode to global (now ha agent stop monitoring engine-vm, you can see above how to activate it)
2.  Access to engine-vm and upgrade oVirt to latest version using the same procedure used for non hosted engine setups.
3.  Select one of the hosted-engine nodes (hypervisors) and put it into maintenance mode from the engine. Note that the host must be in maintenance to allow upgrade to run.
4.  Upgrade that host with new packages (changes repository to latest version and run yum update -y) on this stage may appear vdsm-tool exception <https://bugzilla.redhat.com/show_bug.cgi?id=1088805>
5.  Restart vdsmd (# service vdsmd restart)
6.  Restart ha-agent and broker services (# systemctl restart ovirt-ha-broker && systemctl restart ovirt-ha-agent)
7.  Exit the global maintenance mode: in a few minutes the engine VM should migrate to the fresh upgraded host cause it will get an higher score
8.  When the migration has been completed re-enter into global maintenance mode
9.  Repeat step 3-6 for all the other hosted-engine hosts
10. Enter for example via UI to engine and change 'Default' cluster (where all your hosted hosts seats) compatibility version to current version (for example 3.6 and activate your hosts (to get features of the new version)
11. Change hosted-engine maintenance to none, starting from 3.4 you can do it via UI(right click on engine vm, and 'Disable Global HA Maintenance Mode')

### **Hosted Engine Backup and Restore**

Please refer to [oVirt Hosted Engine Backup and Restore](oVirt Hosted Engine Backup and Restore) guide

### **Lockspace corrupted recovery procedure**

If you end up with corrupted sanlock lockspace due to power outage, hw failure or so, you can fix it using the following procedure:

1.  Move HE to global maintenance
2.  Stop all HE agents on all hosts (keep the local broker running)
3.  Run hosted-engine --reinitialize-lockspace from the host with running broker

You might need to use --force if something is still running, corrupted or did not report proper shutdown. But it should not be necessary for the "best" case of shutting everything down properly before the reinitialize command is issued.

### **Remove old host from the metadata whiteboard**

It is possible to remove an old host from the hosted-engine --vm-status report by using the hosted-engine --clean-metadata command. The agent has to be stopped first. You can force cleaning of a specific ID In the case when the host does not exist anymore by adding --host-id=<ID> argument.

### **More info**

Additional information is available in the feature page [Features/Self_Hosted_Engine](Features/Self_Hosted_Engine)

# **FAQ**

### What is the expected downtime in case of Datacenter / Host / VM failure?

The VM should be up and running in less than 5 minutes if everything works properly. We did test three scenarios with four hosts:

1.  Kill (forced poweroff) of host A at time T
    -   T + 2 minutes - other hosts noticed and tried to start the VM (EngineStarting state)
    -   T + 3 minutes - the engine VM started responding to pings
    -   T + 5 minutes - EngineUp (good health)

2.  Complete forced poweroff of the whole cluster, first machine booting kernel at time T
    -   T + 3 minutes - EngineStarting on the first host
    -   T + 5 minutes - engine VM responding to pings

3.  Engine VM killed with kill -9
    -   T + 0 minutes (matter of seconds) - EngineStarting on other hosts
    -   T + 1 minute - engine VM responding to pings

The measured times assume the network is fine and the VM either crashed or responded to the shutdown command. There is 5 minute grace period when the VM is still running but the ovirt-engine is not responding. It can also take additional five minutes to stop the engine VM when it gets stuck and then additional five minutes to start it (if the engine is not Up after 5 minutes, we kill it and try elsewhere).

### EngineUnexpectedlyDown

#### Failed to acquire lock

When the hosted engine VM is down for some reason the agent(s) will try to start it again. There is no synchronization between agents while starting the VM, so it might happen that more than one agent will try to start the VM at the same time. This is intended behavior because only one host can actually acquire the lock and run the VM. The host which failed the acquire the log will print an error to the vdsm.log: 'Failed to acquire lock: error -243'. The agent will move to the EngineUnexpectedlyDown state, because it failed to start the VM, but it will sync in a while once the timeout expires (you can grep the agent.log for "Timeout" to get the specific time when it should sync).

### Recoving from failed install

If your hosted engine install fails, you have to manually clean up before you can reinstall. Exactly what needs to be done depends on how far the install got before failing. Here are the steps I've used, base on this [thread from the mailing list](http://lists.ovirt.org/pipermail/users/2014-May/024423.html):

*   clean up hosted engine storage. This will vary depending on your storage setup. I logged into my NFS server and purged the directory used during the hoste-engine install.

      # ls  /export/ovirt/hosted-engine
      __DIRECT_IO_TEST__  ce61789b-4291-47d6-a2a6-01263d6b4f5b
      # rm -fR /export/ovirt/hosted-engine/*

*   clean up host files

<!-- -->

    #!/bin/bash

    echo "stopping services"
    service vdsmd stop 2>/dev/null
    service supervdsmd stop 2>/dev/null
    initctl stop libvirtd 2>/dev/null

    echo "removing packages"
    yum remove \*ovirt\* \*vdsm\* \*libvirt\*

    rm -fR /etc/*ovirt* /etc/*vdsm* /etc/*libvirt* /etc/pki/vdsm

    FILES=" /etc/init/libvirtd.conf"
    FILES+=" /etc/libvirt/nwfilter/vdsm-no-mac-spoofing.xml"
    FILES+=" /etc/ovirt-hosted-engine/answers.conf"
    FILES+=" etc/vdsm/vdsm.conf"
    FILES+=" etc/pki/vdsm/*/*.pem"
    FILES+=" etc/pki/CA/cacert.pem"
    FILES+=" etc/pki/libvirt/*.pem"
    FILES+=" etc/pki/libvirt/private/*.pem"
    for f in $FILES
    do
       [ ! -e $f ] && echo "? $f already missing" && continue
       echo "- removing $f"
       rm -f $f && continue
       echo "! error removing $f"
       exit 1
    done

    DIRS="/etc/ovirt-hosted-engine /var/lib/libvirt/ /var/lib/vdsm/ /var/lib/ovirt-hosted-engine-* /var/log/ovirt-hosted-engine-setup/ /var/cache/libvirt/"
    for d in $DIRS
    do
       [ ! -d $f ] && echo "? $d already missing" && continue
       echo "- removing $d"
       rm -fR $d && continue
       echo "! error removing $d"
       exit 1
    done

<Category:SLA>
