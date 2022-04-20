# OCP4 SNO Libvirt Bootstrap In-Place

This uses a single instance qemu+libvirt vm for installing OpenShift 4 - SNO in a home lab.

I'm using fc35, AMD gen3 ryzen 5 (6 physical cores), 96 GB RAM on the lab host.

## FC35

My fedora core35 has libvirt and hugepages configured based on this

- add huge pages - https://access.redhat.com/solutions/36741

## Setup

Create a place to work

```bash
mkdir sno-no-ai && cd sno-no-ai
```

Download the ocp install binaries

Cli and Installer

```bash
# the latest OpenShift 4.10 CLI binaries
wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/latest-4.10/openshift-client-linux.tar.gz
wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/latest-4.10/openshift-install-linux.tar.gz
```

Unpack

```bash
# unpackage them
tar xzf openshift-client-linux.tar.gz
tar xzf openshift-install-linux.tar.gz
# move to the path
chmod 755 kubectl oc openshift-install
VERSION=4.10.4
mv kubectl $HOME/bin/kubectl-$VERSION
mv oc $HOME/bin/oc-$VERSION
mv openshift-install $HOME/bin/openshift-install-$VERSION
ln -sf $HOME/bin/oc-$VERSION $HOME/bin/oc
ln -sf $HOME/kubectl-$VERSION $HOME/bin/kubectl
ln -sf $HOME/bin/openshift-install-$VERSION $HOME/bin/openshift-install
# clean up
rm *.gz README.md
```

Get the coreos-installer binary

```bash
wget https://mirror.openshift.com/pub/openshift-v4/clients/coreos-installer/latest/coreos-installer
chmod 755 coreos-installer
mv coreos-installer $HOME/bin
```

Get the RHCOS ISO from the installer and download

```bash
# get the RHCOS live ISO
ISO_URL=$(openshift-install coreos print-stream-json | grep location | grep x86_64 | grep iso | cut -d\" -f4)
curl -L ${ISO_URL} > rhcos-live.x86_64.iso
```

Create an install config for SNO, adjust ip's, hosts name, secrets to suit

```bash
cat << EOF > install-config.yaml
apiVersion: v1
baseDomain: eformat.me
metadata:
  name: sno
networking:
  networkType: OVNKubernetes
  machineNetwork:
  - cidr: 10.1.1.0/24
compute:
- name: worker
  replicas: 0
controlPlane:
  name: master
  replicas: 1
platform:
  none: {}
bootstrapInPlace:
  installationDisk: /dev/vda
pullSecret: '<your pull secret>'
sshKey: |
  <your ssh public key to access host>
EOF
```

As root, create a libvirt network, adjust to suit

```bash
cat <<EOF > /etc/libvirt/qemu/networks/sno.xml
<network>
  <name>sno</name>
  <uuid>fc43091e-de21-4bf5-974b-98711b9f3d9d</uuid>
  <forward mode='nat' size='1500'/>
  <bridge name='tt0' stp='on' delay='0'/>
  <mac address='52:54:00:29:4d:d7'/>
  <domain name='eformat.me'/>
  <dns enable='yes'>
    <host ip='10.1.1.10'>
      <hostname>api.sno.eformat.me</hostname>
      <hostname>api-int.sno.eformat.me</hostname>
      <hostname>console-openshift-console.apps.sno.eformat.me</hostname>
      <hostname>oauth-openshift.apps.sno.eformat.me</hostname>
      <hostname>canary-openshift-ingress-canary.apps.sno.eformat.me</hostname>
    </host>
  </dns>
  <ip family='ipv4' address='10.1.1.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='10.1.1.2' end='10.1.1.254'/>
      <host mac='52:54:00:29:1d:1a' name='sno' ip='10.1.1.10'/>
    </dhcp>
  </ip>
</network>
EOF
```

Auto start it

```bash
virsh net-define /etc/libvirt/qemu/networks/sno.xml
virsh net-start sno
virsh net-autostart sno
```

Set up dnsmasq pointing to gateway on libvirt network

```bash
echo server=/api.sno.eformat.me/10.1.1.1 | sudo tee /etc/NetworkManager/dnsmasq.d/openshift.conf
echo -e "[main]\ndns=dnsmasq" | sudo tee /etc/NetworkManager/conf.d/openshift.conf
sudo systemctl reload NetworkManager.service
```

Storage - choose between thin-lvm OR qcow2 using a storage pool. i use thin-lvm.

qcow2, setup a storage pool, adjust to suit

```bash
cat <<EOF > /root/sno.xml
<pool type='dir'>
  <name>sno</name>
  <uuid>821c97f0-e6f7-403a-8c84-9a67f7607e9f</uuid>
  <capacity unit='bytes'>999581282304</capacity>
  <allocation unit='bytes'>141831270400</allocation>
  <available unit='bytes'>857750011904</available>
  <source>
  </source>
  <target>
    <path>/var/lib/libvirt/openshift-images</path>
    <permissions>
      <mode>0755</mode>
      <owner>0</owner>
      <group>0</group>
      <label>unconfined_u:object_r:virt_image_t:s0</label>
    </permissions>
  </target>
</pool>
EOF
virsh pool-define /root/sno.xml
virsh vol-delete --pool sno sno
```

OR thin-lvm, create, change names to suit

```bash
lvcreate --size 200G --type thin-pool --thinpool thin_pool fedora_fedora
lvcreate --virtualsize 120G --name sno -T fedora_fedora/thin_pool
vgchange -ay -K fedora_fedora
```

Create all the OCP ignition and iso bits&pieces, set `OPENSHIFT_INSTALL_RELEASE_IMAGE_OVERRIDE` to the version you want to install

```bash
rm -rf cluster
mkdir cluster && cp install-config.yaml cluster/
export OPENSHIFT_INSTALL_RELEASE_IMAGE_OVERRIDE=quay.io/openshift-release-dev/ocp-release:4.10.4-x86_64
openshift-install create single-node-ignition-config --dir=cluster
sudo rm -f /var/lib/libvirt/images/rhcos-live.x86_64.iso
sudo coreos-installer iso ignition embed -fi cluster/bootstrap-in-place-for-live-iso.ign rhcos-live.x86_64.iso -o /var/lib/libvirt/images/rhcos-live.x86_64.iso
sudo chown qemu:qemu /var/lib/libvirt/images/rhcos-live.x86_64.iso
sudo restorecon -rv /var/lib/libvirt/images/rhcos-live.x86_64.iso
```

## Installation

As root, set these env vars, change to suit

```bash
VM_NAME=sno
NET_NAME=sno
POOL_NAME=sno
OS_VARIANT="fedora-coreos-stable"
RAM_MB="32768"
DISK_GB="120"
CPU_CORE="8"
RHCOS_ISO=/var/lib/libvirt/images/rhcos-live.x86_64.iso
MAC=52:54:00:29:1d:1a
```

Install OpenShift SNO

```bash
rm -f nohup.out
nohup virt-install \
    --virt-type kvm \
    --connect qemu:///system \
    -n "${VM_NAME}" \
    -r "${RAM_MB}" \
    --memorybacking hugepages=yes \
    --vcpus "${CPU_CORE}" \
    --os-type=linux \
    --os-variant="${OS_VARIANT}" \
    --cpu=host-passthrough,cache.mode=passthrough \
    --import \
    --network=network:${NET_NAME},mac=${MAC},driver.queues=4 \
    --events on_reboot=restart \
    --cdrom "${RHCOS_ISO}" \
    --disk path=/dev/fedora_fedora/sno,io=io_uring,cache='writeback',discard='unmap' \
    --boot hd,cdrom \
    --wait=-1 &
```

Use this disk line for qcow2 if you wanted to use that instead of thin-lvm

```bash
    --disk pool=sno,size="${DISK_GB},io=io_uring" \
```

Watch the installer

```bash
export KUBECONFIG=~/sno-no-ai/cluster/auth/kubeconfig
openshift-install --dir=cluster --log-level debug wait-for install-complete
```

and ```oc get co``` once installer has progressed.

All going well, OpenShift SNO will install, Profit !

## Thin LVM Snapshots, Rollbacks

Create snapshot lvms i.e. a restore point ( i am using sno-local for openshift local storage here as another disk)

```bash
lvcreate -n sno-snap1 -s fedora_fedora/sno
lvcreate -n sno-local-snap1 -s fedora_fedora/sno-local
```

Stop vm. Restore to snapshots.

```bash
lvconvert --merge /dev/fedora_fedora/sno-snap1
lvconvert --merge /dev/fedora_fedora/sno-local-snap1
```

Restart SNO instance. I have been successfully using this technique to test cluster busting things, then roll back to the thin-lvm snapshot.

## Notes

You can ssh to the node after it boots to check out whats going on

```bash
ssh core@10.1.1.10
sudo crictl ps
```

The virt install i use is optimized for

- hugepages
- host-passthrough cpu
- more network lanes
- io_uring kernel io

On my AMD host, i also found setting the Memory Speed to 2667MHz in the bios (rather than its max. 3200Mhz) prevented soft lockup type errors in the kernel

```bash
general protection fault, probably for non-canonical address 0x9a158ca82eabad75: 0000 [#1] PREEMPT SMP NOPTI
```

Some links for the needy

- https://docs.openshift.com/container-platform/4.10/installing/installing_sno/install-sno-installing-sno.html
- https://gist.github.com/acsulli/912cc974e335128066eaa517f83d693a
- https://github.com/eranco74/bootstrap-in-place-poc
- https://github.com/openshift-hive/installer/tree/master/docs/dev/libvirt
