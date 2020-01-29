# Deploy gogs with postgresql in OCP 3.11
### Create NFS shares for gogs and post Persistent Storage. In my case, I used the bastion server. NFs "no_root_squash" option Must be used when Velero backup/restore tool is used.
```
$ sudo mkdir /var/nfsshare/{gogs,postgresql}
$ sudo chmod 777 /var/nfsshare/{gogs,postgresql}
$ sudo chown nfsnobody:nfsnobody /var/nfsshare/{gogs,postgresql}
$ cat << EOF > /etc/exports.d/cidc.exports
"/var/nfsshare/gogs" *(rw,no_root_squash)
"/var/nfsshare/postgresql" *(rw,no_root_squash)
EOF
$ sudo exportfs -rva
```

### Create PostgeSQL PV
```
$ cat << EOF > pv_postgresql.yml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-postgresql
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: nfs-lb.lab.example.com
    path: /var/nfsshare/postgresql 
EOF

$ oc create -f pv_postgresql.yml
```

### Deploy postgresql server. It should be done first, because gogs apps will connect to it.
```
$ oc new-project gogs
$ oc new-app postgresql-persistent --param POSTGRESQL_DATABASE=gogs --param POSTGRESQL_USER=gogs\
--param POSTGRESQL_PASSWORD=gogs --param VOLUME_CAPACITY=1Gi \
-lapp=gogs_postgresql --insecure-registry
$ oc port-forward postgresql-3-lh7bl 5432:5432
        $ psql -h127.0.0.1 -Ugogs -W
        $ \l
```

### Create Gogs PV
```
$ cat pv_gogs.yml 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-gogs
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: nfs-lb.lab.example.com
    path: /var/nfsshare/gogs

$ oc create -f pv_gogs.yml
```

### Deploy gogs. 
```
$ oc create -f pv_gogs.yml
$ oc new-app wkulhanek/gogs -lapp=gogs_postgresql --insecure-registry
$ oc patch dc gogs --patch='{ "spec": { "strategy": { "type": "Recreate" }}}'
$ oc set volume dc/gogs --add --overwrite --name=gogs-volume-1 --mount-path=/data/ --type persistentVolumeClaim --claim-name=gogs-data --claim-size=1Gi
$ oc expose svc gogs --hostname=gogs.apps.lab.example.com
---Config Via Web---
    - Database Type: PostgreSQL
    - Host: postgresql:5432
    - User: gogs
    - Password: gogs
    - Database Name: gogs
    - Run User: gogs
    - Application URL: http://gogsroute
---Config Via Web---
```

#### Create a configMap to for gogs configuration.
```
$ oc cp $(oc get pod | grep "^gogs" | grep Running | awk '{print $1}'):/opt/gogs/custom/conf/app.ini $HOME/app.ini
$ oc create configmap gogs --from-file=$HOME/app.ini
$ oc set volume dc/gogs --add --overwrite --name=config-volume -m /opt/gogs/custom/conf/ -t configmap --configmap-name=gogs
```

### Backup, could done manually, getting into the gog's pods and excuting the command gogs backup. Also, can be done with "backup_pvc_gogs.sh" script.
```
$ /opt/gogs/gogs backup --target /data/backup

$ git clone https://github.com/silvinux/ocp-tools.git
$ ./backup_pvc_gogs.sh -p gogs -d /var/nfsshare/backup-rsync
$ ls -la /var/nfsshare/backup-rsync/gogs/20200119/gogs-3-9k7wx/zip/20200119-gogs-backup.tar.gz 
-rw-r--r--. 1 silvinux silvinux 257390 Jan 19 14:55 /var/nfsshare/backup-rsync/gogs/20200119/gogs-3-9k7wx/zip/20200119-gogs-backup.tar.gz
```

##### Sources
- https://discuss.gogs.io/t/how-to-backup-restore-and-migrate/991
- https://github.com/silvinux/ocp-tools/blob/master/backup_pvc_gogs.sh
