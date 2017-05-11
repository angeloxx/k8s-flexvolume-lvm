# Introduction

This set of script is intended to be used in Kubernetes to manage a shared volume between nodes without the need of an external provider like Ceph. These scripts are based on:
- Kubernetes flexVolume volume adapter that supports a plugin architecture based on a shell script. The standard LVM adapter is extended to manage simple lock (see https://github.com/kubernetes/kubernetes/tree/master/examples/volumes/flexvolume)
- lvmlockd and sanlock that is used to manage the shared LVM volume without the complexity of a cluster (pacemaker and clvmd)
- a shared volume provided by a shared storage or a shared VMDK file in VMware (my solution)

# Installation

If you want use a shared volume provided by VMware you can follow this guide:
https://communities.vmware.com/blogs/Abhilash_hb/2013/08/25/clustering-using-sharing-of-vmdks-between-virtual-machines

Once configured/installed the shared volume, you need to set-up a cluster-based LVM volume so you DON'T NEED to create the phisical volume but you need to use vgcreate's feature that creates a shared volume group with sanlock support:

    CentOS
    yum install lvmlockd sanlock

    cat <<\EOF | tee -a /etc/lvm/lvm.local.conf

    EOF
    vgcreate --shared sharevg /dev/sdx
    lvcreate -L10G -n sharedlv /dev/sharevg
    mkfs.ext4 /dev/sharevg/sharedlv

Once cloned the repo you can:

    mkdir -p /usr/libexec/kubernetes/kubelet-plugins/volume/exec/angeloxx~lvm
    cp lvm /usr/libexec/kubernetes/kubelet-plugins/volume/exec/angeloxx~lvm/
    cp lvmflexvolumed /usr/local/bin/
    cp lvmflexvolumed.service /etc/systemd/system
    chmod +x /usr/local/bin/lvmflexvolumed
    chmod +x /usr/libexec/kubernetes/kubelet-plugins/volume/exec/angeloxx~lvm/lvm
    systemctl daemon-reload
    systemctl start lvmflexvolumed

# Usage

In a Kubernetes deployment configuration you can use the lvmflexvolumed plugin to attach a previously created volume. The lvm plugin will:
- check if the volume exists
- check if is locked by another node and if the lock is fresh (use the sysconfig/default environment file to set LOCK_TIMEOUT timeout in seconds)
- remove the lock if is old, return an error if the volume is in use on another node or attach the volume
- mount the volume and start the container

The lvmflexvolumed script daemon will refresh the lock every LOCK_SECONDS