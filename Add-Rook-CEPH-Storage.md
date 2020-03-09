# Add Rook/CEPH Storage

This guide assumes that you already have a Openshift cluster up and running with nodes provisioned as VMs using KVM and that you are executing this guide in the same machine that was used to install the cluster.

## Provision the additional VMs needed

This process can be done either at setup or after the initial installation process.

1. Create 3 JSON files for matchbox to assign the worker ignition file when the VMs boot and save them inside the folder `/var/lib/matchbox/groups/`:
    1. storage1.json
    1. storage2.json
    1. storage3.json

    Sample file:

    ```json
    {
        "id": "storage1",
        "name": "storage1",
        "profile": "worker",
        "selector": {
            "mac": "52:54:00:02:88:01"
        }
    }
    ```

1. Create 3 VMs each of them with an extra drive that will be used by CEPH as storage.

    ```bash
    # VM Creation
    $ virsh vol-create-as uvtool storage1.qcow2 120G
    $ virt-install --name=storage1 --ram=32768 --vcpus=8 --mac=52:54:00:02:88:01 \
    --disk path=/var/lib/uvtool/libvirt/images/storage1.qcow2,bus=virtio \
    --pxe --noautoconsole --graphics=vnc --hvm \
    --network network=ocp,model=virtio --boot hd,network
    # Create storage volume and attach it to the VM
    $ virsh vol-create-as uvtool storage11.qcow2 500G
    $ virsh attach-disk storage1 --source /var/lib/uvtool/libvirt/images/storage11.qcow2 --target vdb --persistent
    ```

    The VMs will start and shutdown automatically.

    > Note the MAC addresses assigned match the ones defined in the JSON files in the prior step.

1. Once the VMs are created and the extra storage volume has been attached get the assigned IP addresses.

    ```bash
    $ cat /var/lib/misc/dnsmasq.leases
    ...
    1583806088 52:54:00:02:88:01 192.168.10.214 * 01:52:54:00:02:88:01
    1583823028 52:54:00:02:88:02 192.168.10.215 * 01:52:54:00:02:88:02
    1583805896 52:54:00:02:88:03 192.168.10.216 * 01:52:54:00:02:88:03
    ```

    For reference, in this installation the assigned IP addresses were:

    ```txt
    192.168.10.214
    192.168.10.215
    192.168.10.216
    ```

1. Update the load balancer with the new worker nodes information. The 2 elements that needs updates are `servers.http.discovery` and `servers.https.discovery`. Make the appropriate port is appended for each section. After the edits, restart the service.

    ```bash
    vim /etc/gobetween/gobetween.toml
    systemctl restart gobetween
    ```

1. Add the DNS entries to dnsmasq and restart the service.

    ```bash
    vim /etc/dnsmasq.d/cluster.conf
    ```

    ```txt
    # Storage1
    dhcp-host=52:54:00:02:88:01,192.168.10.214
    address=/storage1.domain.com/192.168.10.214
    ptr-record=214.10.168.192.in-addr.arpa,storage1.domain.com

    # Storage2
    dhcp-host=52:54:00:02:88:02,192.168.10.215
    address=/storage2.domain.com/192.168.10.215
    ptr-record=215.10.168.192.in-addr.arpa,storage2.domain.com

    # Storage3
    dhcp-host=52:54:00:02:88:03,192.168.10.216
    address=/storage3.domain.com/192.168.10.216
    ptr-record=216.10.168.192.in-addr.arpa,storage3.domain.com

    $ systemctl restart dnsmasq
    NO_OUTPUT
    ```

1. Start the VMs. Each VM will boot and install CoreOS using the same images that were used by the installation process.

    ```bash
    virsh start storage1
    virsh start storage2
    virsh start storage3
    ```

1. Depending on how soon you add this nodes to your cluster it is possible that the certificates generated to install CoreOS are now expired (the current expiration time is 60 minutes). To validate that the installation is not hang by expired certificates use `oc get csr`. If there are pending certificates to be approved use the command `oc adm certificate approve all`.

1. Go to the cluster admin console nodes and verify that the new worker nodes have been added or from the terminal use the command `oc get nodes` and make sure all nodes show `Ready` status.

## Install Rook/CEPH in the new nodes

1. Label the worker nodes dedicated to storage appropiately.

    ```bash
    oc label node storage1 role=storage-node
    oc label node storage2 role=storage-node
    oc label node storage3 role=storage-node
    ```

1. Get the Rook installation files.

    ```bash
    cd opt
    git clone https://github.com/rook/rook.git
    cd rook
    # Temporary step that was needed at the time of install
    git checkout 3d5776f
    ```

1. Create the `common` and `operator` objects.

    ```bash
    cd /opt/rook/cluster/examples/kubernetes/ceph
    oc create -f common.yaml
    oc create -f operator-openshift.yaml
    ```

1. Wait for all the pods to start running.

    ```bash
    watch -n5 "oc get pods -n rook-ceph"
    ```

1. Edit the file `cluster.yaml` with your configuration. You can check a sample of the file [here](resources/cluster.yaml).

1. Create the CEPH cluster

    ```bash
    oc create -f cluster.yaml
    ```

1. Wait for the cluster to be created. Check the progress of the pods.

    ```bash
    watch -n5 "oc get pods -n rook-ceph"
    ```

1. Once the cluster is deploy install the CEPH toolbox pod to validate the health of the cluster.

    ```bash
    oc create -f toolbox.yaml
    ```

1. Check that the cluster is healthy by validating that the status is `HEALTH_OK`.

    ```bash
    $ export CEPH_TOOLS_POD=$(oc get pods -n rook-ceph | grep rook-ceph-tools | awk '{print $1}')
    $ oc -n rook-ceph exec -it $CEPH_TOOLS_POD -- /usr/bin/ceph -s
        cluster:
        id:     f6d09f9f-05fd-4aea-9bc6-21dfc16f7c99
        health: HEALTH_OK

        services:
        mon: 3 daemons, quorum a,b,c (age 4d)
        mgr: a(active, since 4d)
        osd: 3 osds: 3 up (since 4d), 3 in (since 4d)

        data:
        pools:   1 pools, 64 pgs
        objects: 6.15k objects, 21 GiB
        usage:   55 GiB used, 542 GiB / 597 GiB avail
        pgs:     64 active+clean

        io:
        client:   74 KiB/s wr, 0 op/s rd, 4 op/s wr
    ```

1. If for some reason your cluster has a warning `Dealing with the Too FEW PGs warning` you can increase the placement groups by doing the following:

    ```bash
    oc exec -it -n rook-ceph $CEPH_TOOLS_POD -- bash
    export POOL_NAME=$(ceph osd lspools | awk '{print $2}')
    ceph osd pool set $POOL_NAME pg_num 64
    ceph osd pool set $POOL_NAME pgp_num 64
    exit
    ```

1. Create the storage class.

    ```bash
    cd /opt/rook/cluster/examples/kubernetes/ceph/csi/rbd
    oc create -f storageclass.yaml
    ```

1. CEPH is now ready to be used in your cluster.
