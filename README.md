# DevOps_Assignment

### Setting up the Host OS controller machine using Vagrant:
A vagrant machine is used as the controller for the deployment, on it ansible is installed using `centos.sh` script, AWS key is copied to `~/.ssh/id_rsa` to allow logging in to the EC2 instances, the key is also referenced in `ansible.cfg` file to allow ansible to execute the roles made.

### Allowing access to AWS Resources:
From IAM a new user was added with `Programmatic access` type, the `AdministratorAccess` Policy was given to the user.

### Using Ansible-Vault to encrypt the credentials:
a new vault file was generated using `ansible-vault create creds.yml` where both `aws_key_id` and `secret_access_key` were specified.
When running the Deploy.yml file the secret file must be included under `vars_files` and to make it easier to run the playbook the `--vault-password-file` option was selected where the password is stored in another file.

```
ansible-playbook Deploy.yml --vault-password-file=secret.txt
```

### Deploying K8s Cluster:
Deploying K8s can be done either using kubeadm or using a cloud managed service like EKS.

### Deploying DevOps Tools using the created K8s Cluster:
#### Jenkins deployment:

1. Using kubectl 
```bash
k create deploy jenkins --image=jenkins/jenkins --port=8080 -o yaml --dry-run=client
```

<details><summary>Jenkins Deployemnt</summary>
<p>

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: jenkins
  name: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: jenkins
    spec:
      containers:
      - image: jenkins/jenkins
        name: jenkins
        ports:
        - containerPort: 8080
        resources: {}
status: {}
```

</p>
</details>

2. Using ansible K8s module

Pre requisites:
pip `openshift` package
By default when installed using `pip3 install --user openshift` it didn't work and i needed to execute `sudo pip3 install --upgrade --user openshift`


<details>
<summary>Jenkins Deployment using ansible</summary>
<p>

```yml
- hosts: localhost
  tasks:
  - name: Create Jenkins Deployment
    community.kubernetes.k8s:
      api_version: apps/v1 
      namespace: default
      definition:
        kind: Deployment 
        metadata:
          name: jenkins-deploy
        spec:
          replicas: 1
          selector:
            matchLabels:
              app: jenkins
          template:
            metadata:
              labels:
                app: jenkins
            spec:
              containers:
              - name: jenkins 
                image: jenkins/jenkins 
                ports:
                - containerPort: 8080
    environment:
        KUBECONFIG: /home/vagrant/.kube/config

```

</p>
</details>

Output:
```
[vagrant@master ~]$ k get deploy
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
jenkins-deploy   0/1     1            0           17s
```

Accessing the deployment is done through a `service` object
```
k expose deploy/jenkins-deploy --type=NodePort --port=8080 --target-port=8080 -o yaml --dry-run=client
```
<details>
<summary>Generated SVC yaml file</summary>
<p>

```yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  name: jenkins-deploy
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: jenkins
  type: NodePort
status:
  loadBalancer: {}
```

</p>
</details>

Now we can access the deployment through the `ep` from the `svc` object

```
[vagrant@master ~]$ k get ep
NAME             ENDPOINTS              AGE
jenkins-deploy   192.168.226.114:8080   71s
```

---

Final result 

---

#### SonarQube Deployment:
* SonarQube uses an Embedded H2 DB by default and in case of this deployment it's left as is.
* The generated yaml still needs modifications for volume attachements as per the [documentation](https://hub.docker.com/_/sonarqube) 

```
k create deploy sonarqube --image=sonarqube --port=9000 -o yaml --dry-run=client
```

<details>
<summary>SonarQube Deployment</summary>
<p>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: sonarqube
  name: sonarqube
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sonarqube
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: sonarqube
    spec:
      containers:
      - image: sonarqube
        name: sonarqube
        ports:
        - containerPort: 9000
        resources: {}
status: {}
```

</p>
</details>

Create SVC for the deployment 

```bash
k expose deploy sonarqube --port=9000 --target-port=9000 --type=NodePort -o yaml --dry-run=client
```

<details>
<summary>SonarQube SVC</summary>
<p>

```yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: sonarqube
  name: sonarqube
spec:
  ports:
  - port: 9000
    protocol: TCP
    targetPort: 9000
  selector:
    app: sonarqube
  type: NodePort
status:
  loadBalancer: {}       
```

</p>
</details>

<details><summary>SonarQube deployment using ansible</summary>
<p>

```yml
- hosts: localhost
  tasks:
  - name: Create SonarQube Deployment
    community.kubernetes.k8s:
      api_version: apps/v1 
      namespace: default
      definition:
        kind: Deployment 
        metadata:
          name: sonarqube
        spec:
          replicas: 1
          selector:
            matchLabels:
              app: sonarqube 
          template:
            metadata:
              labels:
                app: sonarqube 
            spec:
              containers:
              - name: sonarqube 
                image: sonarqube
                ports:
                - containerPort: 9000
    environment:
        KUBECONFIG: /home/vagrant/.kube/config

```

</p>
</details>

---

#### Nexus Deployment:
1. Using kubectl 
```bash
k create deploy nexus --image=sonatype/nexus3 --port=8081 -o yaml --dry-run=client 
```

<details>
<summary>Nexus Deployment</summary>
<p>

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nexus
  name: nexus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nexus
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nexus
    spec:
      containers:
      - image: sonatype/nexus3
        name: nexus3
        ports:
        - containerPort: 8081
        resources: {}
status: {}

```

</p>
</details>

2. Using ansible 
<details><summary>Nexus Deployment Using ansible</summary>
<p>

```yml
- hosts: localhost
  tasks:
  - name: Create Nexus Deployment
    community.kubernetes.k8s:
      api_version: apps/v1 
      namespace: default
      definition:
        kind: Deployment 
        metadata:
          name: nexus
        spec:
          replicas: 1
          selector:
            matchLabels:
              app: nexus 
          template:
            metadata:
              labels:
                app: nexus 
            spec:
              containers:
              - name: nexus 
                image: sonatype/nexus3
                ports:
                - containerPort: 8081
    environment:
        KUBECONFIG: /home/vagrant/.kube/config

```

</p>
</details>

#### Issues in making a single role for the 3 deployments:
Another approach that could have made it easier is to create single task for a deployment and loop over the required deployments, currently i've faced an issue when it comes to using the loop with the k8s module in ansible, the containerPort part of the deployment always throws an error.
This is a problem with openshift since the module relies on it, it can be seen [here](https://github.com/openshift/openshift-restclient-python/issues/321)