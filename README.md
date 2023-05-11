# ceph-kubernetes
Repository containing documentation related to kubernetes and ceph

## Mounting a non default CephFS filesystem to a pod in Kubernetes

The first CephFS you create, will be set as the default filesystem. When mounting a the filesystem using something like this `mount -t ceph {device-string}={path-to-mounted} {mount-point} -o {key-value-args} {other-args}` the default filesystem will be mounted if no filesystem name is specified. The new way to mount a fileystem on a host is like this `mount -t ceph <name>@<fsid>.<fs_name>=/ /mnt/mycephfs`.

### Within kubernetes

The standard way to mount a Ceph FS volume inside a pod is shown here: `https://github.com/kubernetes/examples/blob/master/volumes/cephfs/cephfs.yaml`. This type of volume definition allows you to set monitors, and mount path, but not which Ceph filesystem within your cluster to mount, it will only allow the default.

To mount a non default filesystem, you must first create a Persistent Volume and then a PVC pointing to the cephfs.
Within the Persistent Volume, you can specify standard posix mount options, as the persistent volume uses the old cephFS mount syntax behind the scenes, we can add `-o fs={your_fs_name}` to the mount.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: cephfs-pv
  labels:
    name: cephfs-pv
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi # doesn't matter what you enter here
  mountOptions:
  -  "fs={your_fs_name}"
  accessModes:
    - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain
  cephfs:
    monitors:
      - 10.16.154.78:6789
      - 10.16.154.82:6789
      - 10.16.154.83:6789
    user: admin
    secretRef:
      name: ceph-secret #secret containing your ceph access credentials, more info here: https://github.com/kubernetes/examples/tree/master/volumes/cephfs
    readOnly: true
    path: "path/within/fs"
```

And the PVC:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cephfs-pvc
spec:
  storageClassName: manual
  accessModes:
    - ReadOnlyMany
  resources:
    requests:
      storage: 10Gi
  selector:
    matchLabels:
    - name: cephfs-pv
```

And then finally creating a pod and mounting it within:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu
  labels:
    name: ubuntu
spec:
  volumes:
  - name: cephfs-pvc
    persistentVolumeClaim:
      claimName: cephfs-pvc
  containers:
  - name: ubuntu
    image: ubuntu:latest
    resources:
      limits:
        memory: "1Gi"
        cpu: "1000m"
    volumeMounts:
    - mountPath: "/data"
      name: cephfs-pvc
```
