## Renaming Proxmox VE Nodes

I recently set up a dedicated server so I could run Proxmox to host a few different applications for myself. It's the first one I've ever rented, and in my eagerness to start using it, I didn't think about renaming the server within Proxmox right away, assuming there is a setting somewhere, and I started creating VMs and containers. The installation was provided by the company I rented the server from, and they directly imaged it without any configuration input, so the hostname assigned was the default of `pvehost`. I don't know about you, but I think that's a boring and unhelpful name, and if I have to give things boring names, I'd much rather them be _descriptive_ boring names.

A bit of google searching didn't turn up any results for how to do this unless I _A._ could have  had the prescience to know to change it before guests were added or _B._ wanted to backup all of my guests, remove them, make the changes and then restore them. If I could do A, rest assured that I would, and who's really got the time and patience for B?? So after a bit of digging, I found out that it can in fact be done on a standalone node **with guests present**, despite what I've seen and read elsewhere. And quite easily, at that. There's no need to fiddle with backing up and restoring nodes, reinstalling Proxmox or modifying any of the files or directories in `/etc/pve` either. 

It turns out that the hostname only needs to be changed in three places: `/etc/hosts`, `/etc/hostname` and in a Proxmox configuration file: `/var/lib/pve-cluster/config.db`.  Luckily, this is just a SQLite database, and the hostname is stored in a single record within it. You'll only need a text editor and way to run a single query against the Proxmox config file to change the node's hostname. Here's what worked for me. (and of course, everything should be done as root or via sudo).


### Disclaimer
_I make no guarantees that this will work for you and assume no responsibility if this breaks your production server, destroys your guests, or sets your server on fire. Proceed at your own risk._

However, the process can be made to be a bit less risky. I've tried to include enough detailed instruction to help you mitigate some of that risk. I've tested this on PVE 8.2.2, and I'm unsure if this would work for other versions.

### The Magic

1. Before getting started, _be sure you have an alternate means of access_. As others have said elsewhere, if you're attempting to make these changes through the web shell interface, you will almost certainly get kicked out of it. _SSH or console access is a must_, and having additional accounts that can su to root or console access is highly recommended. 

    I made the mistake of not changing `/etc/hostname` once, but the other two changes were made correctly. The result was that I wasn't able to use my SSH key to log back in as root directly and was unable to use the web interface as PVE failed to start. Thankfully, I had a non-root account that could use `su` to get me back in via SSH.

2. Stop the `pve-cluster` and `pvestatd` services: 

    `systemctl stop pve-cluster ; systemctl stop pvestatd`. 
    
    There could be uncommitted changes to the database, and just in case, it's probably best to force any outstanding changes to be flushed before moving forward.

3. Make a backup copy of the configuration database: 
    
    `cp /var/lib/pve-cluster/config.db ~`

    Of course, this doesn't necessarily need to go in your home folder. Just copy it someplace safe.

4. If you've already tried to change the hostname by modifying files in `/etc/pve`, check the contents of `/etc/pve/nodes`. If you don't see any directories other than the *current* hostname of your node, skip steps 5, 6 and 7.

5. Start `pve-cluster` and `pvestatd` again: 

    `systemctl start pve-cluster ; systemctl start pvestatd`. 
    
    This is necessary for the next change since it's the easiest way to undo most of the changes previously made in `/etc/pve`. while the FUSE filesystem that exposes the configuration is mounted. 

6. Delete any directories in `/etc/pve/nodes` other than the one for your current hostname. Note that this is a CRITICAL step, since the creation of any directories has the effect of modifying the configuration database via a FUSE filesystem, this actually changes the configuration database. If you already have a directory named after your new hostname, this will cause Proxmox to fail when starting, and the config database would need to be fixed. 

    If you make a mistake here, just restore the configuration database from step 3 and try again. 

7. Stop `pve-cluster` and `pvstatd` again

8. Edit your machine's hostname in `/etc/hosts` and `/etc/hostname` 

9. Use your SQLite tool of choice to open the configuration database. I just used the SQL CLI tool, which should already be present on your system:

    `sqlite3 /var/lib/pve-cluster/config.db`

10. Issue the following query: 

    `update tree set name = '<newhostname>' where name = '<oldhostname>';` 

11. __BE SURE THE HOSTNAME MATCHES EXACTLY IN ALL THREE LOCATIONS BEFORE CONTINUING.__ 

    The entry in the `/etc/hosts` can be slightly different, but must include an alias that exactly matches the other two. In addition to having the 
 problem I described in step 1, Proxmox will attempt to determine the IP from that setting and will fail if it's not found.
12. Commit the changes to the configuration database (`.exit` if you're using the CLI tool)

13. Reboot. Everything should start back up the same as it was prior to step 1, but this time, with the new hostname

Prior to rebooting, it might be a good idea to also move relevant files in the `/var/lib/rrdcached/db/pve2-*` directories. For me, this would only need to be done for two directories:

`mv /var/lib/rrdcached/db/pve2-node/<oldhostname> /var/lib/rrdcached/db/pve2-node/<newhostname>` 

and 

`mv /var/lib/rrdcached/db/pve2-storage/<oldhostname> /var/lib/rrcached/db/pve2-storage/<newhostname>`. 

I'm not totally sure what these are used for, and your mileage may vary if you choose not to do this. I opted not to do it and haven't seen any problems yet, and if I find any, I'll update this later. It's also possible that there are some other side effects I'm not aware of, but for a quick and dirty, 5 minute fix rather than a significantly larger/longer project, this worked quite well for me. 

Oh, and it's worth mentioning that this MIGHT work for clusters too. I read somewhere that the clusters replicate a configuration database between them, so making this change on just the node in question might replicate them to all other nodes in the cluster. If someone gets brave and tries it, I hope you'll let me know!

Good luck!

Originally posted on _May 26, 2024_
