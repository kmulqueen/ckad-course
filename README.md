follow along for ckad course

# State Persistence

## Volumes

### Docker

Docker containers are transient in nature (meant to last for only a short period of time). The same is true for the data in the container, it gets destroyed along with the container. To persist the data generated from the container, we create a `volume` to hold onto the data once the container is destroyed.

### K8s

Just like in Docker, our pods in k8s are transient. To keep the data generated from the pod, we attach a volume to the pod to persist the data even when the pod is terminated.

Add this to a pod definition yaml under the `spec` property.

```
volumes:
  - name: data-volume
    hostPath:
      path: /data
      type: Directory
```

When we create a volume, we can configure the storage in different ways. The above example uses a directory on the host. Any files created in the volume would be stored in the `/data` directory on the node.

#### Volume Mounts

Once the volume is created, to access it from the container we need to mount the volume to a directory inside of the container. Add this under the `containers` property in the pod definition yaml file.

```
  volumeMounts:
    - mountPath: /opt
      name: data-volume
```

This mounts the volume `data-volume` to the `/opt` directory inside of the container. The data will be written to `/opt` which is mounting the `data-volume` volume, which in turn is storing the data at `/data` on the host (node).

#### Volume Storage Options

In the above example, we used the host node to store the data. This works fine on a single node. However, it's not recommended for use on a multi-node cluster. This is because the data would be different across all of the nodes. Some options/services that are supported for multi-node clusters are

- NFS
- GlusterFS
- Flocker
- ceph
- ScaleIO
- AWS Elastic Block Store (EBS)
- Azure Disk/File
- GCP Persistent Disk

Here's an example for AWS EBS:

```
volumes:
- name: data-volume
  awsElasticBlockStore:
    volumeID: <volume-id>
    fsType: ext4
```

The volume storage across all of the nodes would now be found in that AWS EBS.

## Persistent Volumes

Previously we defined our volume within the pod-definition yaml file. When we have a large environment with a lot of pods, users would have to configure storage for each pod. Instead, we can centrally manage storage and have users carve out pieces of it when required. A `persistent volume` is a cluster-wide pool of storage volumes to be used by users deploying apps on a cluster. Users can now select storage from the pool using a `persistent volume claim`.

## Persistent Volume Claims

Usually an admin will create a persistent volume, and users/devs will create persistent volume claims to use the storage. These persistent volume claims makes part of the storage pool available to a node. Once the claims are created, K8s _binds_ the persistent volumes to claims based on the request and properties set on the volume. Every persistent volume claim is bound to a single persistent volume.
