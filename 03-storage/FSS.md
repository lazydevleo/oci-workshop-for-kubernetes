# Using File Storage on OCI Kubernetes

### Deploy the FSS Volume Provisioner

First we need to deploy the OCI Volume Provisioner in FSS mode. We do this by setting the PROVISIONER_TYPE to `oracle.com/oci-fss`

```yaml
cat <<'$EOF' | kubectl create -f -
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: oci-volume-provisioner
  namespace: kube-system
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: oci-volume-provisioner
    spec:
      serviceAccountName: oci-volume-provisioner
      containers:
        - name: oci-volume-provisioner
          image: iad.ocir.io/oracle/oci-volume-provisioner:0.9.0
          env:
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: PROVISIONER_TYPE
              value: oracle.com/oci-fss
          volumeMounts:
            - name: config
              mountPath: /etc/oci/
              readOnly: true
      volumes:
        - name: config
          secret:
            secretName: oci-volume-provisioner
$EOF
```

### Create a new Storage Class

Next we create a Storage Class:

```yaml
cat <<'$EOF' | kubectl create -f -
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
  name: oci-fss
provisioner: oracle.com/oci-fss
$EOF
```

You can optionally choose to provide a subnetId in the Storage Class parameters such that mount targets 
will be provisioned on this subnet.

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
  name: oci-fss
provisioner: oracle.com/oci-fss
parameters:
  subnetId: {{SUBNET_OCID}}
```

Additionally you can specify a specific mount target in the storage class also if you already have a mount target to use for FSS volumes.


```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
  name: oci-fss
provisioner: oracle.com/oci-fss
parameters:
  mntTargetId: {{MNT_TARGET_OCID}}
```

### Create a PVC

Next we create a PVC. Note that, although we need to provide a storage capacity in the request, this is not used for FSS file systems and exists purely to make Kubernetes happy since resources.requests.storage is a required field in a PVC.

```yaml
cat <<'$EOF' | kubectl create -f -
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: nginx-fss-volume
spec:
  storageClassName: "oci-fss"
  # The following selector is optional and controls which AD the volume is provisioned in.
  selector:
    matchLabels:
      failure-domain.beta.kubernetes.io/zone: "PHX-AD-1"
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi
$EOF
```

### Consume the PVC storage in a Pod

Now that we have an FSS file system bound to our PVC, we can reference it in a Pod.

```yaml
cat <<'$EOF' | kubectl create -f -
kind: Pod
apiVersion: v1
metadata:
  name: nginx
spec:
  volumes:
    - name: nginx
      persistentVolumeClaim:
        claimName: nginx-fss-volume
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
          name: http
      volumeMounts:
      - mountPath: /usr/share/nginx/html
        name: nginx
$EOF
```