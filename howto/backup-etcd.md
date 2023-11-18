# etcd-backup



## Descripción
Script para realizar backup de la base de datos ETCD de OCP.

Para programar dicho script ejecutar lo siguiente:
```sh
[root@bastion OCP-NOPROD ~]# crontab -l
```
```sh
00 00 * * * sh -x /root/etcd-backup/etcd-backup-v4.5.sh > /root/
```
```sh
etcd-backup/log/backup.log
```

## Script

```bash
#!/bin/bash
set -e
export CLUSTER=prod
export DATE=$(date +'%Y%m%d.%H%M%S-%3N')
export BACKUPDIR=~/prod/backup/etcd/$CLUSTER/backup-$DATE
export SSH_KEY=~/prod/ocp4_prod.priv
export PATH=$PATH:/usr/local/bin/
export KUBECONFIG=/root/prod/auth/kubeconfig
mkdir -p $BACKUPDIR
for master in $(oc get nodes -l node-role.kubernetes.io/master | awk '/master/ {print $1}');
do
  echo "- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -"
  echo "- Backup de base de datos ETCD - $(date)"
  echo "- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -"
  echo "Node backup: $master"
  mkdir -p $BACKUPDIR/$master
  SSH="ssh -i $SSH_KEY core@$master"
  $SSH 'sudo -E rm /home/core/assets/backup/*'
  $SSH 'sudo -E /usr/local/bin/cluster-backup.sh /home/core/assets/backup'
  $SSH 'sudo -E chmod -R 644 /home/core/assets/backup/*'
  scp -r -i $SSH_KEY core@$master:/home/core/assets/backup/* $BACKUPDIR/$master/
  echo "- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -"
  break
  # correrlo una sola vez
done
cd ~/gire/prod/backup/etcd/$CLUSTER/
find .  -mtime +30 -type d  |xargs -d '\n' rm -rf
```