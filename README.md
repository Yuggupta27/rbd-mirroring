# Configure Snapshot-based RBD Mirroring

---

## Introduction

In today's 365 x 24/7 world, to meet high demanding business challenges, data should be highly available, at the right cost, and at the right time. Data migration techniques and Disaster recovery strategies are some of the solutions that address these challenges.

RBD mirroring is an asynchronous replication of RBD images between multiple Ceph clusters.
This capability is available in two modes:
1. **Journal-based**: Every write to the RBD image is first recorded to the associated journal before modifying the actual image. The remote cluster will read from this associated journal and replay the updates to its local image.
2. **Snapshot-based**: This mode uses periodically scheduled or manually created RBD image mirror-snapshots to replicate crash-consistent RBD images between clusters.

The following guide demonstrates how to perform the basic administrative tasks to configure [snapshot-based mirroring](https://docs.ceph.com/en/latest/rbd/rbd-mirroring/#create-image-mirror-snapshots) using the rbd command and handle Disaster Recovery scenarios.

For more details about RBD Mirroring, refer to the official [Ceph documentation](https://docs.ceph.com/en/latest/rbd/rbd-mirroring/).

## Requirements

* Two Ceph clusters, such that, nodes on both clusters can connect to the nodes
  in the other cluster.
  
* Ceph-CSI is deployed on both Primary and Secondary cluster.

  > Note: snapshot-based mirroring requires the Ceph [Octopus](https://docs.ceph.com/en/latest/releases/)
release or later.

The guide assumes we have a setup of two sites, namely:
  * `cluster-a`(primary site) and
  * `cluster-b`(secondary site)

## Enable Mirroring

The **rbd-mirror** daemon is responsible for pulling image updates from the remote, peer cluster ,and applying them to the image within the local cluster.
Once the rbd-mirror daemon is running on both the clusters, follow the below steps:

> **Note**: rbd-mirror assumes the two clusters are named differently.

* First, check the pool status on both the clusters.

  `[cluster-a]$ rbd mirror pool status --pool=replicapool`
  `[cluster-b]$ rbd mirror pool status --pool=replicapool`

* Create an rbd image in `cluster-a`

    ```sh
    [cluster-a]$ rbd create pvcname-namespace-xxx-xxx-xxx --size=1024 --pool=replicapool
    ```

* Verify if the image is created:

    ```sh
    [cluster-a]$ rbd image ls --pool=replicapool or rbd ls --pool=replicapool
    ```

* Enable mirroring on the created image

    ```sh
    [cluster-a] rbd mirror image enable pvcname-namespace-xxx-xxx-xxx snapshot --pool=replicapool
    ```

    > **Note**: Mirrored RBD images are designated as either primary or non-primary.
Images are automatically promoted to primary when mirroring is
first enabled on an image.

* Check whether the mirrored image is now present in the secondary cluster.

    `[cluster-b]$ rbd ls --pool=replicapool`

    The mirrored image should be visible on the secondary cluster now.

* To fetch additional info about image:

    `[cluster-b]# rbd info pvcname-namespace-xxx-xxx-xxx --pool=replicapool`

  ```sh
  O/P:
  rbd image 'pvcname-namespace-xxx-xxx-xxx':
  size 1 GiB in 256 objects
  order 22 (4 MiB objects)
  snapshot_count: 1
  id: 190d48946362
  block_name_prefix: rbd_data.190d48946362
  format: 2
  features: layering, non-primary
  op_features:
  flags:
    create_timestamp: Thu Oct  8 14:19:35 2020
    access_timestamp: Thu Oct  8 14:19:35 2020
    modify_timestamp: Thu Oct  8 14:19:35 2020
    mirroring state: enabled
    mirroring mode: snapshot
    mirroring global id: 14e42673-a2ad-438a-aa5e-5b4fb59f1d8f
    mirroring primary: false
    ```

Since the image on `cluster-b` is "non-primary"; Notice that, here:

1. `mirroring primary` attribute is `false`
1. Also, take a look at its features; one of the features is `non-primary`.
   This depicts that the image here is not primary and cannot be directly altered.

> **Note**: Since mirroring is configured in image mode for the image’s pool,
> it is necessary to explicitly enable mirroring for each image within the pool

---

## Managing Disaster Recovery

To shift the workload from primary to the secondary site,
there are two possible use cases:

* **Planned Migration**: The primary cluster is shutdown properly.

* **Disaster recovery**: The primary cluster went down without a proper shutdown.

The below mentioned steps demonstrate how we can handle both the use cases
using snapshot-based rbd mirroring:

## Planned Migration

> **Use cases**:  Data center maintenance, Technology refresh, Disaster avoidance etc.

In the case of planned migration of the application, the RBD image is unmapped and demoted on the cluster that the application is currently deployed on and later promoted and mapped on the cluster it is migrated to.

To manage failover and failback, the **promote** and **demote** actions are used.
Here, we fail over from site 'a' to site 'b' by demoting the image on site 'a' and promoting it on site'b'.

### Failover

The failover operation is the process of switching production to a backup facility (normally your recovery site).
In the case of Failover, access to the image on the primary site should be stopped.
The image should now be made primary on the secondary cluster, so that the access
can be resumed there.

* Demote the image on `cluster-a` to prevent any further writes on the image on the primary site.

  ```sh
  [cluster-a]$ rbd mirror image demote poolname/imagename
  ```

* Shutdown the primary site.
* Promote the images on `cluster-b` to resume operations on the secondary site.

  ```sh
  [cluster-b]$ rbd mirror image promote poolname/imagename
  ```

* Perform I/O on the image from the secondary cluster now.

### Failback

A failback operation is a process of returning production to its original location after a disaster or a scheduled maintenance period.
Validate if `cluster-a` is back online. If yes, proceed with the below-mentioned steps:

* Demote the image on the `cluster-b` to prevent any further writes.

  ```sh
  [cluster-b]$ rbd mirror image demote poolname/imagename
  ```

* Promote the images on the `cluster-a` to resume operations.

  ```sh
  [cluster-a]$ rbd mirror image promote poolname/imagename
  ```

* The image on `cluster-a` is now ready to be performed I/O on.

## Recovering from an abrupt shutdown

> **Use cases**:  Natural disasters, System failures and crashes, Power failures etc.
> 
An unexpected shutdown and/or an obstruction to communication channels may lead to a "split-brain" condition. In this case, mirroring daemon in both the cluster might claim to be primary.

Thus, here **forced promotion** is needed as the demotion could not be propagated to the peer Ceph cluster (e.g. Ceph cluster failure, communication outage).
Also, the images will no longer be in-sync until a **force resync** command is issued.

### Failover (transition from primary to secondary site)

The guide assumes that the primary site (`cluster-a`) is down, and the workload now
needs to be transferred to the secondary site (`cluster-b`)

* Force promote the image on the `cluster-b` to make it primary.

  ```sh
  [cluster-b]$ rbd mirror image promote poolname/imagename --force
  ```

* The image on `cluster-b` is now ready to be performed I/O on.

### Failback (transition back to primary from the secondary site)

In case of failback, the images of primary cluster cannot be directly used,
as the images on primary and secondary clusters might not be in sync.
Thus, we need to demote the images and re-sync images on the primary cluster
before using them. 
Validate if cluster-a is back online. If yes, proceed
with the below-mentioned steps:

* Demote the images as the it might not be in-sync with the replicated image on `cluster-b`.

    ```sh
    [cluster-a]$ rbd mirror image demote poolname/imagename
    ```

* In case of a split-brain, the affected images will stop getting mirrored. To resume mirroring, we need to re-sync the images.

    ```sh
    [cluster-a]$ rbd mirror image resync poolname/imagename
    ```

* Demote the images on `cluster-b` as now we want to shift back the
  workload to `cluster-a`

    ```sh
    [cluster-b]$ rbd mirror image demote poolname/imagename
    ```

* Promote the images on `cluster-a` to resume I/0

    ```sh
    [cluster-a]$ rbd mirror image promote poolname/imagename
    ```
