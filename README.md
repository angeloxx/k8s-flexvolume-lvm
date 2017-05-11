# Introduction

This set of script is intended to be used in Kubernetes to manage a shared volume between nodes without the need of an external provider like Ceph. These scripts are based on:
- Kubernetes flexVolume volume adapter that supports a plugin architecture based on a shell script. The standard LVM adapter is extended to manage simple lock (see https://github.com/kubernetes/kubernetes/tree/master/examples/volumes/flexvolume)
- lvmlockd and sanlock that is used to manage the shared LVM volume without the complexity of a cluster (pacemaker and clvmd)
- a shared volume provided by a physically shared storage or a shared VMDK file in VMware (my preferred solutions)

The script uses the tag feature of the LVM2 to lock the volume between multiple Kubernetes workers nodes so only one node can activate and mount the volume and other nodes needs to wait a timeout in case of node failure; the lvmlockd/sanlock stack avoids race condition on volume metadata area.

# Installation

If you want use a shared volume provided by VMware you can follow this guide:
https://communities.vmware.com/blogs/Abhilash_hb/2013/08/25/clustering-using-sharing-of-vmdks-between-virtual-machines

Once configured/installed the shared volume, you need to set-up a cluster-based LVM volume so you DON'T NEED to create the phisical volume but you need to use vgcreate's feature that creates a shared volume group with sanlock support:

    CentOS

    # All nodes
    yum install lvmlockd sanlock jq

    cat <<\EOF | tee -a /etc/lvm/lvm.local.conf

    EOF

    # Single node
    vgcreate --shared sharevg /dev/sdx
    lvcreate -L10G -n sharedlv /dev/sharevg
    mkfs.ext4 /dev/sharevg/sharedlv

    # All nodes

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

In a Kubernetes Pod configuration you can use the lvmflexvolumed plugin to attach a previously created volume. The lvm plugin will:
- check if the volume exists
- check if is locked by another node and if the lock is fresh (use the sysconfig/default environment file to set LOCK_TIMEOUT timeout in seconds)
- remove the lock if is old, return an error if the volume is in use on another node or attach the volume
- mount the volume and start the container

The Pod sample configuration is:

    apiVersion: v1
    kind: Pod
    metadata:
    name: nginx
    namespace: default
    spec:
    containers:
    - name: nginx
        image: nginx
        volumeMounts:
        - name: test
        mountPath: /data
        ports:
        - containerPort: 80
    volumes:
    - name: test
        flexVolume:
        driver: "angeloxx/lvm"
        fsType: "ext4"
        options:
            volumeID: "sharelv"
            volumegroup: "sharevg"


The lvmflexvolumed script daemon will refresh the lock every LOCK_SECONDS.