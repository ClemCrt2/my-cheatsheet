# intro
mon: Service Monitor, gère la répartition du cluster et des pools. Un service systemd sur chaque noeud.
mgr: Service Manager, gère la partie authentification et accès aux données. Un service systemd sur chaque noeud.
mds: Service MDS (MetaDataServer), gère la fonctionnilité CEPH Filesystem. Un service systemd sur chaque noeud.
osd: Stockage Brut Si un OSD fail c'est qu'un disque ou un noeud est HS ou est à remplacer
rgw: RadosGw, gère la partie Ceph S3. Un service systemd sur chaque noeud.


# commands for ceph block operations


ceph -s
ceph health details
ceph osd tree
ceph osd pool create k8s_dev
rbd pool init k8s_dev
ceph auth get-or-create client.k8s_dev mon 'profile rbd' osd 'profile rbd pool=k8s_dev' mgr 'profile rbd pool=k8s_dev'
ceph osd pool ls
rbd ls -l k8s_dev
ceph auth get client.k8s_dev
ceph osd pool set-quota {pool-name} [max_objects {obj-count}] [max_bytes {bytes}]
ceph config get-quota /*pool.....*/


# commands more for cephfs et mds conf
ceph fs ls
ceph fs get /*myfs*/
ceph fs status : for active/standby/off mds service
ceph config set mds  mds_cache_memory_limit 17179869184

ceph osd pool create dri_kns_cephfs_dev 
ceph osd pool create dri_kns_cephfs_dev_meta 128

ceph osd pool application enable dri_kns_cephfs_dev cephfs
ceph osd pool application enable dri_kns_cephfs_dev_meta cephfs

ceph fs new <fs_name> <metadata_pool> <data_pool>
ceph auth caps client.cc_admin_fs_dev mgr "allow rw" osd "allow rwx, allow rw tag cephfs data=dri_cc_dev_fs" mds "allow *, allow rw fsname=dri_cc_dev_fs path=csi" mon "allow r, allow r fsname=dri_cc_dev_fs"

ceph fs subvolumegroup create transway_ftp_fs csi

# ceph rgw s3
radosgw-admin bucket stats --bucket=
radosgw-admin quota set \
  --quota-scope=bucket \
  --bucket= \
  --max-size=100G

# commands for kube csi
helm repo add ceph-csi https://ceph.github.io/csi-charts
helm repo update
kubectl kustomize https://github.com/kubernetes-csi/external-snapshotter/client/config/crd | kubectl apply -f -
helm upgrade --install -n kube-ceph-rbd "ceph-csi-rbd" ceph-csi/ceph-csi-rbd -f values_3_15_0.yaml --version "3.15.0" --set storageClass.pool=k8s_dev  --set volumeGroupSnapshotClass.pool=dri_cc_infra_dev_backup

helm repo add ceph-csi https://ceph.github.io/csi-charts
helm repo update
kubectl kustomize https://github.com/kubernetes-csi/external-snapshotter/client/config/crd | kubectl apply -f -
helm upgrade --install -n kube-ceph-fs "ceph-csi-cephfs" ceph-csi/ceph-csi-cephfs -f values_3_15_0.yaml --version "3.15.0" --set storageClass.fsName=XXXX --create-namespace
kubectl apply -f clusterole-provisionner.yaml

# debug
## recovery : récupérer les priorits de recovery et ralentir les operations

ceph --show-config | egrep "osd_max_backfills| osd_recovery_max_active|osd_max_scrubs|osd_scrub_load|osd_recovery_op_priority" 
 ceph tell 'osd.*' injectargs '--osd-max-backfills 1 --osd-recovery-max-active_ssd 3 --osd-max-scrubs 1 --osd_recovery_op_priority 2' 


## Erreur sur le provisionning pvc

Warning  ProvisioningFailed    40s (x7 over 103s)   cephfs.csi.ceph.com_ceph-csi-cephfs-provisioner-6d44d5bcf4-f557x_2e63c094-40f0-4a61-9c7c-3db30079527c  failed to provision volume with StorageClass "csi-cephfs-sc": rpc error: code = Aborted desc = an operation with the given Volume ID pvc-3c666319-c722-4e46-8cf1-37a154543110 already exists
  Normal   ExternalProvisioning  3s (x12 over 2m44s)  persistentvolume-controller                                                                            Waiting for a volume to be created either by the external provisioner 'cephfs.csi.ceph.com' or manually by the system administrator. If volume creation is delayed, please verify that the provisioner is running and correctly registered


mds_cache_trim_decay_rate de 09 à 098

sudo ceph tell mds.dps66602-2 status

