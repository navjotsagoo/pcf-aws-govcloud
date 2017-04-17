# Pivotal Cloud Foundry on AWS GovCloud
**Instructions to deploy Pivotal Cloud Foundry on AWS GovCloud**

Download the latest AWS Ubuntu Stemcell from [bosh.io](http://bosh.io/stemcells/bosh-aws-xen-hvm-ubuntu-trusty-go_agent)

```bash
wget --content-disposition https://bosh.io/d/stemcells/bosh-aws-xen-hvm-ubuntu-trusty-go_agent?v=3363.15
```

Unzip the stemcell file
```bash
tar -xzf light-bosh-stemcell-3363.15-aws-xen-hvm-ubuntu-trusty-go_agent.tgz
```
## Identify AMIs
Identify the AMI id for us-east-1 region
```bash
more stemcell.MF
```
> us-east-1: ami-93b23185

Identify the [AMI id for Ops Manager](https://network.pivotal.io/products/ops-manager/) in us-east-1 region
> us-east-1: ami-8828a99e

## Launch AMIs
Launch `ami-93b23185` in us-east-1 region to create a Stemcell VM. Select volume size of 10GB.

Launch `ami-8828a99e` in us-east-1 region to create an Ops Manager VM. Select volume size of 100GB (recommended).

## Take Snapshots & Create EBS Volumes
After successful launch, stop the Stemcell VM and take a snapshot.
> Select VM --> Actions --> Image --> Create Image

After successful launch, stop the Ops Manager VM and take a snapshot.
> Select VM --> Actions --> Image --> Create Image

Create a new EBS volume of the earlier stemcell vm snapshot. We will call this `stemcell-source` disk.
> Select Snapshot --> Actions --> Create Volume

Create a new EBS volume 5x the size of `stemcell-source` disk.
> Create Volume --> --> select size: 50G --> name 'stemcell-temp'

Create a new EBS volume of the earlier Ops Manager vm snapshot. We will call this `opsmanager-source` disk.
> Select Snapshot --> Actions --> Create Volume

Create a new EBS volume 5x the size of `opsmanager-source` disk
> Create Volume --> select size: 500G --> name 'opsmanager-temp'

## Create a disk copy for each volume
Launch a generic ubuntu vm in AWS commercial us-east-1 region. SSH into this vm.

### Stemcell
Attach the `stemcell-source` disk to generic VM. Select volume --> Actions --> Attach Volume.
>  /dev/xvdf

Attach `stemcell-temp` disk to generic VM
> /dev/xvdg

View all available disk devices
```bash
lsblk
```
Format the `stemcell-temp` disk with ext4 file system
```bash
sudo mkfs -t ext4 `/dev/xvdg`
```

Mount the `stemcell-temp` disk to VM
```bash
sudo mkdir /stemcell
sudo mount /dev/xvdg /stemcell
```

Copy the `stemcell-source` to `stemcell-temp` with dd utility
```bash
dd if=/dev/xvdf bs=512K | bzip2 -9 -c > /stemcell/3363.stemcell.raw.bz2
```
### Ops Manager
Attach the `opsmanger-source` disk to generic VM
> /dev/xvdh

Attach the `opsmanager-temp` disk to generic VM
> /dev/xvdi

View all available disk devices
```bash
lsblk
```
Format the `opsmanager-temp` disk with ext4 file system
```bash
sudo mkfs -t ext4 `/dev/xvdi`
```

Mount the `opsmanager-temp` disk to VM
```bash
sudo mkdir /opsman
sudo mount /dev/xvdi /opsman
```

Copy the `opsmanager-source` to `opsmanager-temp` with dd utility
```bash
dd if=/dev/xvdh bs=512K | bzip2 -9 -c > /opsman/ops.manager.raw.bz2
```

## Transfer files to AWS GovCloud
Launch a generic VM in AWS GovCloud.
Copy the pem key to your AWS Commercial generic VM.
Secure copy files from AWS Commercial VM to AWS GovCloud VM.
```bash
sudo scp -i awsgov.pem 3363.stemcell.raw.bz2 ubuntu@ec2-...us-gov-west-1.compute.amazonaws.com:/home/ubuntu
sudo scp -i awsgov.pem ops.manager.raw.bz2 ubuntu@ec2-...us-gov-west-1.compute.amazonaws.com:/home/ubuntu
```

## Unzip the files
SSH into the generic VM in AWS GovCloud
```bash
bunzip2 3363.stemcell.raw.bz2
bunzip2 ops.manager.raw.bz2
```

## Create EBS Volumes
Create two EBS volumes, each of the original size of the snapshot taken earlier

Stemcell
> /dev/xvde (10 GB)

Ops Manager
> /dev/xvdg (100 GB)

Format them with ext4 and mount them to generic VM. See commands from above.

## Copy files to EBS volumes
```bash
dd if=3363.stemcell.raw of=/dev/xvde
dd if=ops.manager.raw of=/dev/xvdg bs=512K
```

Detach EBS Volumes from Generic VM
Create Snapshots for each volume

For Stemcell snapshot, create an AMI with /dev/sda1: snapshot + /dev/sdb instance-store, HVM

Note down the AMI ID

For Ops Manager, modify settings
- Remount /dev/xvdg (ebs volume for ops manager) to generic VM
