# EnergyTeam

### Prerequisites
A clean VM with linux installed on it. For this assessment, I will be using ubuntu.
System Requirements:
- 2GB RAM
- 2 vCPU
- 32GB disk space

### Installing minikube

In order to get our minikube instance up and running, we will need to a container manager installed. For this assessment, I will be using docker, with containerd as the container runtime.

##### Installing docker

Follow the steps in the [docker](https://docs.docker.com/engine/install/ubuntu/) installation guide:
```
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update

# Install Docker
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

To run docker as a non-root user, add your user to the "docker" group:
```
sudo groupadd docker
sudo usermod -aG docker $USER
```

Then either refresh your login session or run `newgrp docker`.

Check everything works correctly using `docker ps` and continue to the next step.

##### Installing minikube
With Docker up and running, we now need to install minikube.

Follow the [minikube](https://minikube.sigs.k8s.io/docs/start/?arch=%2Flinux%2Fx86-64%2Fstable%2Fbinary+download) installation guide:

```
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64
```

Then run `minikube start`

To interact with the cluster, we can either install it directly or let minikube download it for us. For this assessment, I will be installing kubectl.

If you don't wish to install kubectl by yourself, you can begin accessing your cluster immediately. Run `minikube kubectl -- get pods -A' to get all the pods in the cluster. Running kubectl like this for the first time tells minikube to install kubectl for you.
It is recommended to create an alias in your shell config by adding the following line:
```
alias kubectl="minikube kubectl --"
```

##### Installing kubectl (optional)

If you choose to install kubectl yourself, follow the [installation guide](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/):
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

You should now be able to use kubectl to access your cluster. Run `kubectl version --client` to ensure you are running the latest version.

Following this, continue to the next step.

### Creating a PostgreSQL instance

After getting minikube running on our machine, we can create our postgres instance.

Start by installing the Percona PostgreSQL Operator using their [installation guide](https://docs.percona.com/percona-operator-for-postgresql/3.1.0/minikube.html#deploy-the-percona-operator-for-postgresql):
```
kubectl create namespace postgres-operator
kubectl apply --server-side -f https://raw.githubusercontent.com/percona/percona-postgresql-operator/v3.1.0/deploy/bundle.yaml -n postgres-operator
```

Check that the pods are running:
```
kubectl get pods -n postgres-operator
```

Then, deploy Percona Distribution for PostgreSQL:
```
kubectl apply -f https://raw.githubusercontent.com/percona/percona-postgresql-operator/v3.1.0/deploy/cr.yaml -n postgres-operator
```

Wait a few moments for everything to initialize, then run `kubectl get pods -n postgres-operator`
Once all the pods are up and running, we can connect to our DB and start running queries on it.

### Connectiong to PostgreSQL
In order to connect, first we will get the connection string for PgBouncr. The connection string is saved as a secret under the name of <cluster_name>-<user_name>-<clustername>. Since we haven't set a cluster name, it should be cluster1 by default.

PgBouncer acts as a connection pooler in order to keep the number of actual database connections low for reduced memory overhead.

Ensure that the secret exists:
```
kubectl get secret cluster1-pguser-cluster1 -n postgres-operator
```

Save the decoded secret into an environment variable:
```
PGBOUNCER_URI=$(kubectl get secret cluster1-pguser-cluster1 --namespace <namespace> -o jsonpath='{.data.pgbouncer-uri}' | base64 --decode)
```
This will take the "pgbouncer-uri" key's value from our secret, decode it using base64 and then save it into an env var for our use.

Finally, we can use this connection string to connect to our postgres database. For simpliciy's sake, we will be creating a container inside our cluster to perform queries using percona's postgres image.

```
kubectl run -i --rm --tty pg-client --image=percona/percona-distribution-postgresql:18.6.1-1 --restart=Never -- psql $PGBOUNCER_URI
```
This will create a container that will connect to our connection pooler for us to finally be able to run queries to the DB.

Create a simple schema, then create a table and add some values into it:
```
CREATE SCHEMA demo;
CREATE TABLE EnergyTeam(
    name TEXT,
    status TEXT,
    description TEXT
);
INSERT INTO EnergyTeam(name, status, description)
VALUES(
    "Itay",
    "Complete"
    "I have completed the assessment for EnergyTeam!"
);

SELECT description FROM EnergyTeam;
```
