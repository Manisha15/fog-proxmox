# Compute

This document shows you the compute service available with fog-proxmox.

Proxmox supports both virtual machines (QEMU/KVM) and containers (LXC) management.

You can see more details in [Proxmox VM management wiki page](https://pve.proxmox.com/wiki/Qemu/KVM_Virtual_Machines) and [Proxmox containers management wiki page](https://pve.proxmox.com/wiki/Linux_Container).

## Starting irb console

```ruby
irb
```

```ruby
require 'fog/proxmox'
```

## Create compute service

```ruby
compute = Fog::Compute::Proxmox.new(
        pve_username: PVE_USERNAME, # your user name
        pve_password: PVE_PASSWORD, # your password
        pve_url: PVE_URL # your server url
)
```

Optional [connection parameters](connection_parameters.md) are also available.

## Fog Abstractions

Fog provides both a **model** and **request** abstraction. The request abstraction provides the most efficient interface and the model abstraction wraps the request abstraction to provide a convenient `ActiveModel` like interface.

### Request Layer

The request abstraction maps directly to the [Proxmox VE API](https://pve.proxmox.com/wiki/Proxmox_VE_API). It provides an interface to the Proxmox Compute service.

To see a list of requests supported by the service:

```ruby
compute.requests
```

To learn more about Compute request methods refer to source files.

To learn more about Excon refer to [Excon GitHub repo](https://github.com/geemus/excon).

### Model Layer

Fog models behave in a manner similar to `ActiveModel`. Models will generally respond to `create`, `save`,  `persisted?`, `destroy`, `reload` and `attributes` methods. Additionally, fog will automatically create attribute accessors.

Here is a summary of common model methods:

<table>
	<tr>
		<th>Method</th>
		<th>Description</th>
	</tr>
	<tr>
		<td>create</td>
		<td>
			Accepts hash of attributes and creates object.<br>
			Note: creation is a non-blocking call and you will be required to wait for a valid state before using resulting object.
		</td>
	</tr>
	<tr>
		<td>save</td>
		<td>Saves object.<br>
		Note: not all objects support updating object.</td>
	</tr>
	<tr>
		<td>persisted?</td>
		<td>Returns true if the object has been persisted.</td>
	</tr>
	<tr>
		<td>destroy</td>
		<td>
			Destroys object.<br>
			Note: this is a non-blocking call and object deletion might not be instantaneous.
		</td>
	<tr>
		<td>reload</td>
		<td>Updates object with latest state from service.</td>
	<tr>
		<td>ready?</td>
		<td>Returns true if object is in a ready state and able to perform actions. This method will raise an exception if object is in an error state.</td>
	</tr>
	<tr>
		<td>attributes</td>
		<td>Returns a hash containing the list of model attributes and values.</td>
	</tr>
		<td>identity</td>
		<td>
			Returns the identity of the object.<br>
			Note: This might not always be equal to object.id.
		</td>
	</tr>
	<tr>
		<td>wait_for</td>
		<td>This method periodically reloads model and then yields to specified block until block returns true or a timeout occurs.</td>
	</tr>
</table>

The remainder of this document details the model abstraction.

#### Nodes management

Proxmox supports cluster management. Each hyperviser in the cluster is called a node.
Proxmox installs a default node in the cluster called `pve`.

List all nodes:

```ruby
service.nodes.all
```

This returns a collection of `Fog::Compute::Proxmox::Node` models:

Get a node:

```ruby
node = service.nodes.find_by_id 'pve'
```

#### Servers management

Proxmox supports servers management. Servers are also called virtual machines (VM).

VM are QEMU/KVM managed. They are attached to a node.

More details in [Proxmox VM management wiki page](https://pve.proxmox.com/wiki/Qemu/KVM_Virtual_Machines)

You need to specify a node before managing VM. Fog-proxmox enables it by managing VM from a node.

List all servers:

```ruby
node.servers.all
```

This returns a collection of `Fog::Identity::Proxmox::Server` models.

Before creating a server you can get the next available server id (integer >= 100) in the cluster:

```ruby
nextid = node.servers.next_id
```

You can also verify that an id is free or valid:

```ruby
node.servers.id_valid? nextid
```

Now that you have a valid id, you can create a server in this node:

```ruby
node.servers.create({ vmid: nextid })
```

Get this server:

```ruby
server = node.servers.get nextid
```

Add options: boot at stratup, OS type (linux 4.x), french keyboard, no hardware KVM:

```ruby
server.update({ onboot: 1, keyboard: 'fr', ostype: 'l26', kvm: 0 })
```

Add a cdrom volume:

```ruby
server.update({ ide2: 'none,media=cdrom' })
```

Add a network interface controller (nic):

```ruby
server.update({ net0: 'virtio,bridge=vmbr0' })
```

##### Volumes server management

Before attaching a hdd volume, you can first fetch available storages that could have images in this node:

```ruby
storages = node.storages.list_by_content_type 'images'
storage = storages[0] # local-lvm
```

Four types of storage controllers emulated by Qemu are available:

* **IDE**: ide[n], n in [0..3]
* **SATA**: sata[n], n in [0..5]
* **SCSI**: scsi[n], n in [0..13]
* **VirtIO Block**: virtio[n], n in [0..15]

The hdd id is the type controller appended with an integer (n).

More details on complete configuration options can be find in [Proxmox VE API](https://pve.proxmox.com/wiki/Proxmox_VE_API).

Then attach a hdd from this storage:

```ruby
disk = { id: 'virtio0', storage: storage.storage, size: '1' } # virtualIO block with 1Gb
options = { backup: 0, replicate: 0 } # nor backup, neither replication
server.attach(disk, options)
```

Resize a disk:

```ruby
server.extend('virtio0','+1G')
```

Move a disk

```ruby
server.move('virtio0','local')
```

Detach a disk

```ruby
server.detach 'virtio0'
```

Actions on your server:

```ruby
server.action('start') # start your server
server.wait_for { server.ready? } # wait until it is running
server.ready? # you can check if it is ready (i.e. running)
```

```ruby
server.action('suspend') # pause your server
server.wait_for { server.qmpstatus == 'paused' } # wait until it is paused
```

```ruby
server.action('resume') # resume your server
server.wait_for { server.ready? } # wait until it is running
```

```ruby
server.action('stop') # stop your server
server.wait_for { server.status == 'stopped' } # wait until it is stopped
```

Fetch server disk_images:

```ruby
disk_images = server.disk_images
```

This returns an array of `Fog::Compute::Proxmox::Volume` instances.

Delete server:

```ruby
server.destroy
```

##### Backup and restore server

You can backup all node's guests or just one guest.

You need first to get a node or a server to manage its backups:

```ruby
node = compute.nodes.get 'pve'
server = node.servers.get vmid
```

Then you can backup one server:

```ruby
options = { compress: 'lzo'}
server.backup options
```

or backup all servers on a node:

```ruby
node.backup options
```

You can restore a server from a backup.
Backups are volumes which content type is `backup` and owned by a server.

You first fetch the backup volumes of this server:

```ruby
volumes = server.backups
```

This returns an array of `Fog::Compute::Proxmox::Volume` instances.

Then you choose one:

```ruby
backup = volumes[0] # local:backup/vzdump-qemu-100-2018_05_15-15_18_31.vma.lzo
```

This returns a `Fog::Compute::Proxmox::Volume` instance:

```ruby
 <Fog::Compute::Proxmox::Volume
    volid="local:backup/vzdump-qemu-100-2018_05_15-15_18_31.vma.lzo",
    content="backup",
    size=376,
    format="vma.lzo",
    node=nil,
    storage=nil,
    vmid="100"
  >
```

Then you can restore it:

```ruby
options = { compress: 'lzo'}
server.restore backup
```

You can delete it:

```ruby
backup.delete
```

More details on complete backup `options` configuration hash can be find in [Backup and restore wiki page](https://pve.proxmox.com/wiki/Backup_and_Restore).

##### Snapshots server management

You need first to get a server to manage its snapshots:

```ruby
server = node.servers.get vmid
```

Then you can create a snapshot on it:

```ruby
snapname = 'snapshot1' # you define its id
server.snapshots.create snapname
```

Get a snapshot:

```ruby
snapshot = server.snapshots.get snapname
```

Add description:

```ruby
snapshot.description = 'Snapshot 1'
snapshot.update
```

Rollback server to this snapshot:

```ruby
snapshot.rollback
```

Delete snapshot:

```ruby
snapshot.destroy
```

##### Clones server management

Proxmox supports cloning servers. It creates a new VM as a copy of the server.

You need first to get a server to manage its clones and a valid new VM id:

```ruby
server = node.servers.get vmid
newid = node.servers.next_id
```

Then you can clone it:

```ruby
server.clone(newid)
```

It creates a new server which id is newid. So you can manage it as a server.

Destroy the clone:

```ruby
clone = node.servers.get newid
clone.destroy
```

#### Tasks management

Proxmox supports tasks management. A task enables to follow all asynchronous actions made in a node: VM creation, start, etc. Indeed, some of these tasks could be long to execute.

You need first to get a node to manage its tasks:

```ruby
node = compute.nodes.find_by_id 'pve'
```

Search tasks (limit results to 1):

```ruby
tasks = node.tasks.search { limit: 1 }
```

Get a task by its id. This id can be retrieved as a result of an action:

```ruby
taskid = snapshot.destroy
task = node.tasks.find_by_id taskid
task.wait_for { succeeded? }
```

Stop a task:

```ruby
task.stop
```

### Examples

More examples can be seen at [examples/compute.rb](examples/compute.rb) or [spec/compute_spec.rb](spec/compute_spec.rb).