# Gluster 3 node dispersed volume setup with client
This repo contains files needed to build a virtualized gluster cluster to reproduce [Gluster issue #3980](https://github.com/gluster/glusterfs/issues/3980).

The idea here is to set up a 3-node dispersed cluster, with each node having 15 extra disks attached for brick storage.

I am assuming that you have working KVM/libvirt setup for whatever user you're running this as. The Ansible playbook to do local operations doesn't use `become` because my user is already part of the `libvirt` group so I don't need to `sudo` anything - you can do that too or just add `become: true` where necessary.

You'll probably need 

## Dependencies
- Use [virt-lightning](https://virt-lightning.org/) to set up VMs or DIY but that's more painful. I will assume virt-lightning here and document for that.
    - Note - virt-lightning relies on libvirt for python which seems like it might be broken in 3.10 due to a breaking API change in the 3.10 standard library. I used 3.8.
- Ansible. I used version 2.9.

## How to reproduce
### Setup
First, set up your VMs. In this directory:
```
$ vl up
⚡ gluster-server-1 
⚡ gluster-server-2 
⚡ gluster-server-3 
⚡ gluster-client-1 
⌛ ok Waiting...
💻 gluster-server-2 found at 192.168.123.5!
💻 gluster-server-1 found at 192.168.123.6!
💻 gluster-client-1 found at 192.168.123.7!
💻 gluster-server-3 found at 192.168.123.8!
👍 You are all set
```

Great, now you have some VMs. Next let's make an Ansible inventory for them:
```
$ vl ansible_inventory > inventory/inventory_vl
```

Note: An example of the above file (`inventory/inventory.vl`) already exists in this repo. It probably won't work for you.

Next, make sure you change `ansible_user` in `inventory/virt-host.yml` from `thalin` to your user. I don't know that this is actually required, but it's what worked for me so let's assume it is. This is the user on your machine which will be used to configure VMs and stuff.

The last bit of setup is to change `storage_pool` in the same file if you want to set up your own storage pool for the 45 volumes we're going to be attaching to the gluster server VMs. I just used the default storage pool, since I also include a clean up playbook to remove these. However, if you ever find yourself stuck partially through an ansible execution...you might have to go delete these manually with `virsh`. Good luck. I'm sure you'll figure it out. Hint: `virsh volume-list --pool default --name` will get you the names of all the volumes in the default storage pool.

### Configure Virtual Machines
In this step we'll run an Ansible playbook to create a bunch of qcow volumes & attach them to the VM nodes we created above.

```
$ ansible-playbook -i inventory virt-setup.yml
```

When at the end you see some stuff about deleting gluster bricks - yeah, that step is skipped in this playbook and is for the `virt-teardown.yml` playbook later on to clean up.

### Configure Gluster
Here's where we actually configure each node to run Gluster. This consists of a few steps:
1. Format & mount each disk created in the previous step for each node
1. Install & start Glusterd
1. Set up hosts file so that each hosts knows about the others (so we can peer probe & mount the volume)
1. Peer probe
1. Create the volume
1. Apply configuration to the volume (yeah we can probably combine these but I didn't)
1. Install the glusterfs-client package on the client node
1. Make the mountpoint & mount it on the client

```
$ ansible-playbook -i inventory gluster-setup.yml
```

### Reproduce the problem
Ok, now we have a fully configured gluster cluster! Let's go make some directories and see how slow it is.

```
$ vl ssh gluster-client-1
$ bash # because tab completion is nice
$ cd /mnt/test-volume
$ for i in {1..4500}; do mkdir this_is_a_really_long_directory_name_with_a_number_${i}; done
$ strace -o ~/ls_strace.txt ls
$ time ls -l
real	0m20.622s
user	0m0.089s
sys	0m0.231s
```

Cool.

## Clean up after yourself
Ok so we made a bit of a mess here. We have at least 45 volumes that we probably don't want to keep long term, plus some VMs that are wasting CPU cycles. Let's clean up after ourselves.

### TEAR IT DOWN, TEAR IT ALL DOWN!
Along side the setup playbooks, I included a couple of handy teardown playbooks as well to help you dispose of your unwanted volumes. Conveniently, virt-lightning will get rid of the VMs for us once we're done with the volumes.

Technically, you don't _need_ to do this step, but it makes me feel better so here it is:
```
$ ansible-playbook -i inventory gluster-teardown.yml
```

This unmounts the volume, destroys the volume, and unmounts the bricks.

Next, let's get rid of those volumes:
```
$ ansible-playbook -i inventory virt-teardown.yml
```

This playbook detaches all the volumes from the VMs and then deletes them. This will work without destroying the gluster volume, above, but it makes me uncomfortable to do it that way so I don't usually.

Finally, we can get rid of the VMs:

```
$ vl down
🗑 purging gluster-client-1
🗑 purging gluster-server-1
🗑 purging gluster-server-3
🗑 purging gluster-server-2
```

Ok. We're all cleaned up.

# In conclusion

This is pretty slow on some pretty nice machines. It's even slower on my little VM host with VMs which have 1/16 the ram & 1/64 the CPUs of the production machines. I guess this is no surprise.

Also, I'm including some output requested in the ticket (logs, straces, etc) in the `output` directory.
