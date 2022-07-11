---
title:  "How To Change Or Add New SSH Key for EC2"
date:   2019-10-25 09:00:00
permalink: how-to-change-or-add-new-ssh-key-for-ec2
---

This is a documentation of how to change or add new ssh key for your EC2 instance if you lost, and maybe compromised your private key.

The gist of it is to add in a new key pair to the disk volume of the EC2 instance. Pretty straightforward! But how can you do it without  being able to ssh into the EC2 instance without the private key you just lost? You will need to attach the root volume of the EC2 instance to another temporary EC2 instance, which you can access with a new key pair, and add in the new key pair to the original volume from there.

## Summon the NewKeyPair!

First, create a new key pair. You can either generate a private and public key pair on your own and import the public one into the AWS console, or create it from the AWS management console and download the private key that they generated for you thereafter. Should you go for the latter, make sure your browser is not blocking the download.

INSERT IMAGE Blocked download

For the rest of the article, the new key pair will be referred to as NewKeyPair, and the old key pair LostKeyPair.

## Retire The Veteran

Stop your old instance. Do not terminate!

NOTE: Your instance root volume need to be EBS backed and not instance store as instance store volumes are ephemeral. They do not persist the data after power down.

Once it has successfully stopped, you will realise that its volume remains attached. That’s EBS for you!

We will come to detaching it in a while. For now, spin up a new server.

## Katon: Summon-The-New-Server-Jutsu

Launch a new server with the `NewKeyPair`. This is a temporary server and can be any of the linux distribution.

## Detach The Old Volume

In the volumes page, select the old instance volume and select Detach as shown. There should be no error unless your old instance is still in the process of shutting down.

INSERT IMAGE - Detach old EBS Volume

Once it is detached, you will observe that its status has changed to available and its Attachment Information will become blank. Now it is freeee! Time to attach it to the new server and receive its new key pair.

## Attach To New Instance

Attach the root volume to the new instance as shown.

INSERT IMAGE

Then select the device to mount on.

INSERT IMAGE attach volume device

I will set `/dev/sdf` as suggested. The other devices reserved for the root volume (`/dev/sda`) and instance store volumes (`/dev/sd[b-e]`). More information on the device naming in AWS EC2 can be found here.

Run the command `lsblk` to see the new volume mounted. Note that the linux kernel has change my mount point from `sdf` to `xvdf` as noted in the warning callout in the image above.

INSERT IMAGE lsblk

## Mounting The Volume

You would not be able to use the volume right away after attaching without mounting the volume in the system. Mounting will tell the EC2 instance how to access this new device via its list of directory. This will require setting up a mount point. Run the commands below.

```bash
sudo mkdir /mnt/tempvol
sudo mount /dev/xvdf1 /mnt/tempvol
```

These commands will mount the root of the device to the directory named `/mnt/tempvol`. You can change directory into the volume and see that it contains content from your old server.

From the image above, you can see that the `authorized_keys` file containing the old public key is placed in `/home/ubuntu/.ssh directory` relative to the mount point. The new public key pair exist in the `/home/ubuntu/.ssh` directory in the absolute path, which exist in the root volume of the new instance.

## Adding The NewKeyPair To The Old Volume

Eventually, we want to use the new key pair to access the old server, with the content of the old volume, just like the good old times. To do get, add the `NewKeyPair` to the ssh folder of the old volume.

Realise that this is now possible because of the attaching and mounting of the old volume to a new instance which can be accessed due to a fresh setup and key pair creation.

I have used the append operation, `>>` instead of the overwrite operation, which is a single `>`. This is not necessary. It is up to you to decide if you want to get rid of the old key pair or not, depending on your situation.

If you lost your old key pair, feel free to overwrite it. There is no point hoarding it, and Marie Kondo can’t help you declutter software.

## Attaching The Volume Back

Next, you can shut down your new server and attach your old volume back to the old instance. Remember, the EC2 instance will not be deleted and is still available if you chose stop instead of terminate when shutting it down initially.

INSERT IMAGE attach volume back

Mount the volume, this time, to `/dev/sda1`.

INSERT IMAGE attach to `sda1`

The reason to mount it at `/dev/sda1` is because we need to give the instance back its root volume for its boot operations. If we were to mount it to another device, you will see this error when starting the server because no root volume is detected.

INSERT IMAGE  error starting old instance

## Back To The Past

Now you can try to connect to your old instance after it has started up.

NOTE: you will see that the connection instruction is still mentioning the `LostKeyPair`. Even if you had overwritten your ssh key pair to the new one, this wrong instructions will still persist. Of course, you should connect with your `NewKeyPair`.

INSERT IMAGE Connect to old instance again

To ascertain that you have the old public keys, head dow to the `~/.ssh` directory and see that the changes you made on the new instance via attaching and mounting of the old instance’s volume has persisted.

Now, we have successfully added a new key pair to the old instance, and we can use it to ssh into the old instance form now on, even though we had lost our original key pair that was used to create it.

