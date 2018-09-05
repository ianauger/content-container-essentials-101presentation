Storage Drivers
===============

Which driver for which OS?
--------------------------

- Storage drivers define how a container stores information -- not how mounted volumes store information
- Using a volume for image storage is preferrable, however, writing to a container may be a necessity (e.g., installing software inside the container)
- Need to pick a driver that is performant and compatible with host OS
	- Docker provides a prioritized list, based first on amount of customization/configuratio needed, then by stability and performance
	- CE and EE implementations may support differentt drivers
	- Driver availabilty varies by filesystem time and platform
- Docker EE availability
	- Centos, Oracle Linux: devicemapper
	- RHEL: devicemapper, overlay2
	- SUSE Enterprise Linux: btrfs
	- Ubuntu: aufs3
- Docker CE -- less of an official recommendation, more of a "this *should* work"
	- Centos: devicemapper, vfs
	- Oracle Linux: devicemapper, overlay2
	- RHEL: devicemapper, overlay2
	- SUSE Enterprise Linux: btrfs
	- Debian/Ubuntu: aufs3, devicemapper, overlay2, overlay, vfs
	- Fedora: devicemapper, overlay2, overlay, vfs
	- Overall, recommended to use overlay2 where possible (will be default on install)
- Storage driver workload use cases
	- aufs, overlay, overlay2
		- Operates at file level as opposed to block level
		- More efficient memory utilization, but container layer grows quickly
	- devicemapper, btrfs, zfs
		- Operates at block level
		- Allows better performance in write-heavy workloads at cost of memory
	- overlay
		-Can be more efficient than overlay2 in use cases with small writes, containers with many layers or deep filesystems
- Remember -- changing storage driver changes it for all containers, back up or export your images before changing storage drivers

Images are composed of multiple layers on the filesystem
--------------------------------------------------------

- Images built up from a series of layers
- Each layer is composed of a single instruction in the docker file
- Layeres read-only until the last one
- Instructions counted per instruction in the dockerfile -- commands chained together all part of same layer
- Each successive layer applied as a delta to previous layers, including the final thin R/W layer
- Storage driver handles image layer interactions
- Each container has its own writable container layer -- maintains 1 to N ratio of image to container storage
- Copy-on-write strategy moves files from lower read-only layers to highest read-write layer only when needed, i.e. when a change is made
- Containers and deletion
	- When container is deleted, any data in the R/W layer that isn't in a volume gets deleted
	- Volumes not controlled by storage driver
	- Allows for portability while providing persistent storage

Using Storage and Volumes Across Cluster Nodes
----------------------------------------------

- It can't!  Kinda.
- Services can't assume that mounted volumes are necessarily available across the cluster
- So if a volume is needed, the service can create one...
- But it's not available across the cluster, as clusters don't share files
- Two volume drivers
	- Local volume driver
	- Cloud volume storage
		- Allows for shared storage on AWS/Azure -- not covered by certification
- Volume creation:
	- `docker volume create <name>`
- To mount on service, need the --mount option, not --volume
	- `--mount source=<source folder>,target=<container mountpoint>`
	- Replicates mounts across nodes in the cluster, but not files
- Removing volume on manager removes it on workers as well
- Better practice to store externally to container where possible

Cleaning up unused images (and other things)
--------------------------------------------

- Docker creates a number of services with associated fileystem objects that can take up space on the filesystem
- `docker system prune`
	- Has a mollyguard because it removes rather a lot
		- All stopped container
		- All networks not used by at least one container at the time the command is run
		- All dangling (unused) images
		- All cached build layers
	- Will not delete anything needed by Docker configuration (e.g. the default networks)
	- Adding other options will add more to the prune, e.g. --volumes adds unused volumes
	- -a removes all images not associated to a container
- Pruning occurs only on the host you run it on -- does not extend across cluster
- Can also specify specific objects to prune -- `docker network prune`, `docker image prune`, etc.
