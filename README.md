# Configure Multi-cluster Scheduling Using Admiralty


This Ansible Role will install cert-manager v1.7.1 and Admiralty v0.15.1 on Kubernetes cluster to perform multicluster scheduling.

pre-requisites:
* Helm3<br>
* python <br>
* Ansible

update **contexts** in vars folder in both cert manager and k8s-admiralty.


```
ansible-playbook -i inventory k8s-admiralty.yml
```

with cert-manager
```
ansible-playbook -i inventory k8s-admiralty.yml --tags "cert-manager"
```
:::note Admiralty Open Source uses cert-manager to generate a server certificate for its mutating pod admission webhook. :::

## configuration:

```
kubectl create  ns mpi-jobs --context host
```
```
kubectl create  ns mpi-jobs --context member1
```
```
kubectl label ns mpi-jobs multicluster-scheduler=enabled --context host
```
## Cross-Cluster Authentication (Install jq, the command-line JSON processor, if not already installed.)
```
kubectl -n mpi-jobs --context member1 create serviceaccount host
```
```
SECRET_NAME=$(kubectl -n mpi-jobs --context member1 get serviceaccount host \
	--output json | \
	jq -r '.secrets[0].name')
```
```
TOKEN=$(kubectl -n mpi-jobs --context member1 get secret $SECRET_NAME \
	--output json | \
	jq -r '.data.token' | \
	base64 --decode)
```
```
CONFIG=$(kubectl -n mpi-jobs --context member1 config view \
	--minify --raw --output json | \
	jq '.users[0].user={token:"'$TOKEN'"} | .clusters[0].cluster.server="https://192.168.1.104:6443"')
```
```
kubectl -n mpi-jobs --context host create secret generic member1 \
	--from-literal=config="$CONFIG"
```
# Multi-Cluster Scheduling
## In the management cluster, create a Target for each workload cluster:
```
cat <<EOF | kubectl --context host apply -f -
apiVersion: multicluster.admiralty.io/v1alpha1
kind: Target
metadata:
  name: member1
spec:
  kubeconfigSecret:
    name: member1
EOF
```

## In the workload clusters, create a Source for the management cluster:
```
cat <<EOF | kubectl --context member1 apply -f -
apiVersion: multicluster.admiralty.io/v1alpha1
kind: Source
metadata:
  name: host
spec:
  serviceAccountName: host
EOF
```

## Example:
create a file **volcano-example.yaml** and insert below context
```
apiVersion: batch.volcano.sh/v1alpha1
kind: Job
metadata:
  name: lm-mpi-job
spec:
  minAvailable: 3
  schedulerName: volcano
  plugins:
    ssh: []
    svc: []
  tasks:
    - replicas: 1
      name: mpimaster
      policies:
        - event: TaskCompleted
          action: CompleteJob
      template:
        metadata:
          annotations:
            multicluster.admiralty.io/elect: ""
        spec:
          containers:
            - command:
                - /bin/sh
                - -c
                - |
                  MPI_HOST=`cat /etc/volcano/mpiworker.host | tr "\n" ","`;
                  mkdir -p /var/run/sshd; /usr/sbin/sshd;
                  mpiexec --allow-run-as-root --host ${MPI_HOST} -np 2 mpi_hello_world > /home/re;
              image: volcanosh/example-mpi:0.0.1
              name: mpimaster
              ports:
                - containerPort: 22
                  name: mpijob-port
              workingDir: /home
          restartPolicy: OnFailure
    - replicas: 2
      name: mpiworker
      template:
        metadata:
          annotations:
            multicluster.admiralty.io/elect: ""
        spec:
          containers:
            - command:
                - /bin/sh
                - -c
                - |
                  mkdir -p /var/run/sshd; /usr/sbin/sshd -D;
              image: volcanosh/example-mpi:0.0.1
              name: mpiworker
              ports:
                - containerPort: 22
                  name: mpijob-port
              workingDir: /home
          restartPolicy: OnFailure
```
```
kubectl apply -f volcano-example.yaml -n mpi-jobs --context host
```
This should create pods on both clusters.
