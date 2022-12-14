# The first things I do when installing Proxmox

## Background
Like many home-labbers I'm a fan of using Proxmox as a bare-metal hypervisor and occassionally have a need to install it on a new host. The purpose of this post is to remind me of the steps that I normally go through immediately after installing Proxmox on a new host and before spinning up any guest VMs.

## Configure updates
After installing Proxmox, the first step is to SSH into the host to configure updates. Since I don't currently have a subscription for Proxmox, I'll be adding the non-production update channel.

1. Open the apt sources list in `nano` or `vi` to add the non-production source

```bash
nano /etc/apt/sources.list
```

2. Add the following couple of lines to the end of the file.

```bash
# not for production use
deb http://download.proxmox.com/debian bullseye pve-no-subscription
```

3. Now open up the enterprise sources list and comment out the line(s) in that file. This prevents errors when running `apt-get update` and other similar commands.

```bash
nano /etc/apt/sources.list.d/pve-enterprise.list
```

## Add NAS storage
I use a NAS to centrally store `iso` files on my network, as well as a serving as a target for VZDump backup files from Proxmox. The next step is to tell Proxmox where to find this storage and supply the credentials for the service account used to access it.

1. Within the Proxmox GUI, go to `Datacentre -> Storage` and click `Add`.
2. Select `SMB/CIFS` from the dropdown menu.
3. Provide the required server name and authentication details for Proxmox to add a new share, as well as specifying the content types that it can be used to store. I typically select `Disk image`, `ISO image` and `VZDump backup file`.
4. Click `Add` to, well, add the new storage location.

## Schedule backups
With a file share configured, we can now configure backups to be sent to that location.

1. Within the Proxmox GUI, go to `Datacentre -> Backups` and click `Add`.
2. Set your preferences to create a backup job for your guest VMs. I normally select `ZSTD (fast and good)` as the compression mode.

## Enable VLAN awareness
This is another simple one, but I do it on each Proxmox host that I run.
1. Within the Proxmox GUI, click on the hostname of your server and then go to `System/Network` and then select the Bridge and click `Edit` in the menu.
2. Check the box next to `VLAN aware` and then click `OK`.

## Optional: Add node to Proxmox cluster
Adding the new node to a cluster enables you to move VMs back and forth between different nodes, as well as giving you visibility of all your nodes from a single instance of the Proxmox GUI. To add a new node to an existing cluster:

1. Access the Proxmox GUI on one of the nodes already in the cluster.
2. Navigate to `Datacentre -> Cluster` and click on `Join Information` then `Copy Information`.
3. Switch to the Proxmox GUI on the new node and again go to the `Cluster` section.
4. Click `Join Cluster` and then paste the join information details from the clipboard. In a minute or so you should see the new node added to the cluster.
