# Pivotal Cloud Foundry on AWS GovCloud
**Instructions to deploy Pivotal Cloud Foundry on AWS GovCloud**

PreReqs:
- AWS Commercial account with admin privileges
- AWS GovCloud account with admin
---
Overview of Steps
1.
---
## Identify AMI IDs for Ubuntu Stemcell & Ops Manager

Review the [release notes](https://docs.pivotal.io/pivotalcf/1-10/pcf-release-notes/opsmanager-rn.html) to determine the version of stemcell compatible for the specific version of Ops Manager.
We will use `Ops Manger 1.10.4` and  `Stemcell 3363.15` as an example in this tutorial.

Launch a `jumpbox` Ubuntu 16.04 VM in AWS commercial us-east-1 region. SSH into this vm.

Download the AWS Stemcell from Pivotal Network or [bosh.io](http://bosh.io/stemcells/bosh-aws-xen-hvm-ubuntu-trusty-go_agent)
```bash
wget --content-disposition https://bosh.io/d/stemcells/bosh-aws-xen-hvm-ubuntu-trusty-go_agent?v=3363.15
```

Unzip the stemcell file
```bash
tar -xzf light-bosh-stemcell-3363.15-aws-xen-hvm-ubuntu-trusty-go_agent.tgz
```

Identify the `AMI id for Ubuntu Stemcell` in us-east-1 region
```bash
more stemcell.MF
```
> us-east-1: ami-93b23185

Identify the [AMI id for Ops Manager](https://network.pivotal.io/products/ops-manager/) in us-east-1 region
> us-east-1: ami-8828a99e

## Launch AMIs in AWS Commercial

Launch `ami-93b23185` in us-east-1 region to create a Stemcell VM. Select volume size of 10 GB.

Launch `ami-8828a99e` in us-east-1 region to create an Ops Manager VM. Select volume size of 50 GB.

## Take Snapshots & Create EBS Volumes

After successful launch, stop the Stemcell vm and take a snapshot.

After successful launch, stop the Ops Manager vm and take a snapshot.

Create a new EBS volume `stemcell-source` from the earlier Stemcell vm snapshot.

Create a new EBS volume `stemcell-temp` 4x the size of stemcell-source disk.
> 10 GB (Original Stemcell VM ) * 4 = 40 GB

Create a new EBS volume `opsmanager-source` from the earlier Ops Manager vm snapshot.

Create a new EBS volume `opsmanager-temp` 4x the size of opsmanager-source disk
> 50 GB (Original Ops Manager VM )* 4 = 200 GB

## Create a Disk Copy of Each Volume

### Create a disk copy of Stemcell

Attach the `stemcell-source` disk to jumpbox vm.
>  /dev/xvdf

Attach the `stemcell-temp` disk to jumpbox vm.
> /dev/xvdg

Confirm all available disks are attached.
```bash
lsblk
```

Format the `stemcell-temp` disk with ext4 file system
```bash
sudo mkfs -t ext4 `/dev/xvdg`
```

Mount the `stemcell-temp` disk to jumpbox vm
```bash
sudo mkdir /stemcell
sudo mount /dev/xvdg /stemcell
```

Copy the `stemcell-source` disk to `stemcell-temp` disk with data duplicator(dd)
```bash
dd if=/dev/xvdf bs=512K status=progress | bzip2 -9 -c > /stemcell/3363.stemcell.raw.bz2
```

### Create a disk copy of Ops Manager

Attach the `opsmanger-source` disk to jumpbox vm
> /dev/xvdh

Attach the `opsmanager-temp` disk to jumpbox vm
> /dev/xvdi

View all available disk devices
```bash
lsblk
```

Format the `opsmanager-temp` disk with ext4 file system
```bash
sudo mkfs -t ext4 `/dev/xvdi`
```

Mount the `opsmanager-temp` disk to jumpbox vm
```bash
sudo mkdir /opsman
sudo mount /dev/xvdi /opsman
```

Copy the `opsmanager-source` disk to `opsmanager-temp` disk with data duplicator(dd)
```bash
dd if=/dev/xvdh bs=512K status=progress | bzip2 -9 -c > /opsman/ops.manager.raw.bz2
```

## Transfer files to AWS GovCloud

Launch a jumpbox Ubuntu 14.04 VM in AWS GovCloud. Select volume size of 200 GB during provisioning.
Create or update this jumpbox's security group to allow ssh access from AWS Commercial jumbox vm.

Copy the pem key to your AWS Commercial jumpbox vm.  

Secure copy files from AWS Commercial VM to AWS GovCloud vm.
```bash
sudo scp -i awsgov.pem 3363.stemcell.raw.bz2 ubuntu@ec2-...us-gov-west-1.compute.amazonaws.com:/home/ubuntu
sudo scp -i awsgov.pem ops.manager.raw.bz2 ubuntu@ec2-...us-gov-west-1.compute.amazonaws.com:/home/ubuntu
```
Alternative: Use Amazon S3 to transfer files to AWS GovCloud jumpbox vm.

** All actions below are now done in AWS GovCloud **
## Uncompress Files

SSH into the jumpbox VM in AWS GovCloud

```bash
cd /home/ubuntu
bunzip2 3363.stemcell.raw.bz2
> 3363.stemcell.raw
bunzip2 ops.manager.raw.bz2
> ops.manager.raw
```

## Create New EBS Volumes

Create two new EBS volumes, each of the original size of the snapshots taken earlier.
> Stemcell volume = 10 GB
> Ops Manager volume = 50 GB

Format them with ext4 file system and mount them to jumpbox vm. See commands from above.

Stemcell volume disk device name
> /dev/xvde

Ops Manager volume disk device name
> /dev/xvdg

## Copy Uncompressed Files to EBS volumes

Download Pipe Viewer, a terminal based tool for monitoring the progress of data transfer.
```bash
sudo apt-get install pv
```

Copy the uncompressed files to EBS volumes. This will take several hours. :sleeping:
```bash
dd if=3363.stemcell.raw | pv -tapbe | dd of=/dev/xvde
dd if=ops.manager.raw | pv -tapbe | dd of=/dev/xvdg bs=512K
```

## Create Stemcell AMI

Detach Stemcell EBS Volume `/dev/xvde` from Generic VM. Create Snapshot.

From Stemcell snapshot, create an AMI with /dev/sda1 as root and /dev/sdb as instance-store.
This is Stemcell AMI ID `ami-4ee6622f` in AWS GovCloud.

## Create Ops Manager AMI

Create a directory and mount the `part` segment type of Ops Manager EBS volume to that directory.
```bash
sudo mkdir /opsman
sudo mount /dev/xvdg1 /opsman
```

Add AWS GovCloud Region
```bash
sudo vi /opsman/home/tempest-web/tempest/web/app/models/persistence/models/aws/aws_infrastructure.rb
> Search for `SUPPORTED_REGIONS` text. Add `us-gov-west-1` to the list.
```

Download the original stemcell from bosh.io or Pivotal Network & open stemcell
```bash
cd /home/ubuntu
mkdir stemcells && cd stemcells
wget --content-disposition https://bosh.io/d/stemcells/bosh-aws-xen-hvm-ubuntu-trusty-go_agent?v=3363.15
tar -xzf light-bosh-stemcell-3363.15-aws-xen-hvm-ubuntu-trusty-go_agent.tgz
```

Modify the `stemcell.MF` file with the AWS GovCloud Stemcell AMI ID from previous step.
```
us-gov-west-1: ami-4ee6622f
```

Remove original stemcell file and repackage the new stemcell
```bash
rm light-bosh-stemcell-3363.15-aws-xen-hvm-ubuntu-trusty-go_agent.tgz
tar zcvf light-bosh-stemcell-3363.15-aws-xen-hvm-ubuntu-trusty-go_agent.tgz *
```

Create `SHA-1` of the stemcell file
```bash
sha1sum light-bosh-stemcell-3363.15-aws-xen-hvm-ubuntu-trusty-go_agent.tgz > sha1.txt.sha1
cat sha1.txt.sha1
> `b0d2faba37b5ee17aad322fdfa0c5dfa1a14370a`
```

Edit Ops Manager AWS Configuration
```bash
sudo vi /opsman/home/tempest-web/tempest/web/config/versions.yml
> Edit `sha1` field for `aws` under stemcells
```

Remove the original stemcell from Ops Manager stemcell default directory
```bash
cd /opsman/var/tempest/stemcells/
rm light-bosh-stemcell-3363.15-aws-xen-hvm-ubuntu-trusty-go_agent.tgz
```
Move the modified stemcell to the Ops Manager stemcell default directory
```bash
cp /home/ubuntu/stemcells/* /opsman/var/tempest/stemcells/
```

Unmount and detach `/dev/xvdg1` disk from jumpbox v
```bash
umount -d /dev/xvdg1
```

Create a snapshot and then create an AMI of the snapshot.
Congratulations, you now have an Ops Manager AMI in GovCloud. :hand:
> ami-10f67271

Launch the Ops Manager AMI `ami-10f67271` and proceed with standard PCF installation. :clap:
