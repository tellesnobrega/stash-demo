# Running Backup & Restore on a Kubernetes environment using Stash and Minio

The goal of this post is to provide a step-by-step tutorial on how to set up, backup and restore a WordPress application running on Minikube,
using Stash for Backup and Restore and Minio as S3-like Object Storage.

This is part of a series of introductions to backup and restore tools I'm playing with.
If you are interested also in Velero, check [Velero Blog Post](https://tellesnobrega.github.io/velero-demo/) and if you are
interested in Kanister, check [Kanister Blog Post](https://tellesnobrega.github.io/kanister-demo/)

## Setting up the Environment

### Install docker
```
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y docker-ce docker-ce-cli containerd.io
sudo systemctl start docker
sudo systemctl enable docker
```

### Running minio container
```
docker pull minio/minio
docker run -p 9000:9000 --name minio -e "MINIO_ACCESS_KEY=minio" -e "MINIO_SECRET_KEY=minio123" -v /mnt/data:/data minio/minio server /data
```
### Install Kubectl
```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
yum install -y kubectl
```

### Install minikube
```
sudo yum -y install conntrack
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-1.9.2-0.x86_64.rpm
sudo rpm -ivh minikube-1.9.2-0.x86_64.rpm
minikube start --driver=none
```
### Install Helm
```
yum -y install openssl
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

### Install Stash
```
helm repo add appscode https://charts.appscode.com/stable/
helm repo update
helm search repo appscode/stash
helm install stash-operator appscode/stash --version v0.9.0-rc.2 --namespace kube-system
```

### Install Stash MySQL Bind
```
helm repo add appscode https://charts.appscode.com/stable/
helm repo update
helm install appscode/stash-mysql --name=stash-mysql-8.0.14 --version=8.0.14
```

### Clone this repo
```
git clone https://github.com/tellesnobrega/stash-demo.git
```

### Deploy wordpress application
```
kubectl create ns wordpress
kubectl create secret -n wordpress generic mysql-pass \
      --from-literal=username=root \
      --from-literal=password=<MYSQL_ROOT_PASSWORD>
kubectl apply -f stash-demo/mysql-deployment.yaml
kubectl apply -f stash-demo/wordpress-deployment.yaml
```
#### Check for wordpress url
```
minikube -n wordpress service wordpress --url
```

### Add some content to WordPress

Now that the environment is set up you can add a post to WordPress

### Prepare the backup setup

#### Create S3 credentials
```
echo -n 'changeit' > RESTIC_PASSWORD
echo -n '<your-aws-access-key-id-here>' > AWS_ACCESS_KEY_ID
echo -n '<your-aws-secret-access-key-here>' > AWS_SECRET_ACCESS_KEY
kubectl create secret generic -n wordpress s3-secret \
    --from-file=./RESTIC_PASSWORD \
    --from-file=./AWS_ACCESS_KEY_ID \
    --from-file=./AWS_SECRET_ACCESS_KEY
```

#### Create the S3 Repository CRD for WordPress and MySQL
```
kubectl apply -f stash-demo/wordpress-s3.yaml
kubectl apply -f stash-demo/mysql-s3.yaml
```
#### Create the MySQL AppBinding CRD
```
kubectl apply -f stash-demo/mysql-appbinding.yaml
```

#### Create the BackupConfiguration CRD for WordPress and MySQL
```
kubectl apply -f stash-demo/wordpress-backupconfiguration.yaml
kubectl apply -f stash-demo/mysql-backupconfiguration.yaml
```

Wait until the backups are done.
```
watch -n 3 kubectl -n wordpress get backupsession
```

Pause the backup jobs.
```
kubectl patch backupconfiguration -n wordpress wordpress-backup --type="merge" --patch='{"spec": {"paused": true}}'
kubectl patch backupconfiguration -n wordpress wordpress-mysql-backup --type="merge" --patch='{"spec": {"paused": true}}'
```

With Stash we can't recover a lost namespace, but we can rebuild an environment with a backup from a destroyed one.
We will recover two different scenarios:

1. We will break the database and also wordpress and recover from backup.

2. We will delete the wordpress namespace, recreate it and recover from backup.

##### Scenario 1

Go into the mysql container and delete the wp-posts tables from wordpress database.

Go into the wordpress container and delete wp-content/ folder.

Refreshing the Wordpress page you will see that it is not working anymore.

###### Run restore command
```
kubectl apply -f stash-demo/wordpress-restoresession.yaml
kubectl apply -f stash-demo/mysql-restoresession.yaml
```
Wait until the restore is finished. You can follow the progress using the command below.
```
watch -n 2 kubectl get restoresession -n wordpress
```

Once the wordpress pod is running again, refresh the wordpress and make sure wordpress is completely recovered.


##### Scenario 2

```
kubectl delete ns wordpress
kubectl create ns wordpress
kubectl create secret -n wordpress generic mysql-pass \
      --from-literal=username=root \
      --from-literal=password=<MYSQL_ROOT_PASSWORD>
kubectl apply -f stash-demo/mysql-deployment.yaml
kubectl apply -f stash-demo/wordpress-deployment.yaml
echo -n 'changeit' > RESTIC_PASSWORD
echo -n '<your-aws-access-key-id-here>' > AWS_ACCESS_KEY_ID
echo -n '<your-aws-secret-access-key-here>' > AWS_SECRET_ACCESS_KEY
kubectl create secret generic -n wordpress s3-secret \
    --from-file=./RESTIC_PASSWORD \
    --from-file=./AWS_ACCESS_KEY_ID \
    --from-file=./AWS_SECRET_ACCESS_KEY

kubectl apply -f stash-demo/wordpress-s3.yaml
kubectl apply -f stash-demo/mysql-s3.yaml
kubectl apply -f stash-demo/mysql-appbinding.yaml
```

###### Run restore command
```
kubectl apply -f stash-demo/wordpress-restoresession.yaml
kubectl apply -f stash-demo/mysql-restoresession.yaml
kubectl -n wordpress patch svc wordpress -p '{"spec": { "type": "NodePort", "ports": [ { "nodePort": <PORT>, "port": 80, "protocol": "TCP", "targetPort": 80 } ] } }'
```
Replace <PORT> with the port from previous wordpress deployment. This is needed because wordpress keeps url information
on the database and after the restore minikube gives the service a new PORT. Patching this solves this redirecting issue.

Refresh WordPress and make sure it is working as expected.
