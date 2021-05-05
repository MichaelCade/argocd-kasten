# Argo CD and Kasten K10
# Incorporating data management into your Continuous Deployment workflows and GitOps model

This is a very simple example of how we can integrate kasten with argo cd. It's voluntary kept very simple because we focus on using kasten with a [pre-sync phase](https://argoproj.github.io/argo-cd/user-guide/sync-waves/) in Argo CD.

## Install Argo CD

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Username is admin and password can be obtained with this command.

``` 
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## Phase 1 - Deploying the Application 

We create a mysql app for sterilisation of animals in a pet clinic. 

This app is deployed with Argo CD and is made of : 
*  A mysql deployment 
*  A PVC 
*  A secret 
*  A service to mysql 

We also use a pre-sync job (with corresponding sa and rolebinding)to backup the whole application with kasten before application sync. 

At the first sync an empty restore point should be created.

## Phase 2 - Adding Data

Vets are creating the row of the animal they will operate. 

```
mysql_pod=$(kubectl get po -n mysql -l app=mysql -o jsonpath='{.items[*].metadata.name}')
kubectl exec -ti $mysql_pod -n mysql -- bash

mysql --user=root --password=ultrasecurepassword
CREATE DATABASE test;
USE test;
CREATE TABLE pets (name VARCHAR(20), owner VARCHAR(20), species VARCHAR(20), sex CHAR(1), birth DATE, death DATE);
INSERT INTO pets VALUES ('Puffball','Diane','hamster','f','2021-05-30',NULL);
INSERT INTO pets VALUES ('Sophie','Meg','giraffe','f','2021-05-30',NULL);
INSERT INTO pets VALUES ('Sam','Diane','snake','m','2021-05-30',NULL);
INSERT INTO pets VALUES ('Medor','Meg','dog','m','2021-05-30',NULL);
INSERT INTO pets VALUES ('Felix','Diane','cat','m','2021-05-30',NULL);
INSERT INTO pets VALUES ('Joe','Diane','crocodile','f','2021-05-30',NULL);
SELECT * FROM pets;
exit
exit
```

## Phase 3 - ConfigMaps + Data

We create a config map that contains the list of species that won't be eligible for sterilisation. This was decided based on the experience of this clinic, operation on this species are too expansive. We can see here a link between the configuration and the data. It's very important that configuration and data are captured together.

```
cat <<EOF > forbidden-species-cm.yaml 
apiVersion: v1
data:
  species: "('crocodile','hamster')"
kind: ConfigMap
metadata:
  name: forbidden-species
EOF 
git add forbidden-species-cm.yaml
git commit -m "Adding forbidden species" 
git push
```

When deploying the app with Argo Cd we can see that a second restore point has been created

## Phase 4 - The failure scenario

At this stage of our application we want to remove all the rows that have species in the list, for that we use a job that connects to the database and that deletes the rows. 

But we made a mistake in the code and we accidentally delete other rows. 

Notice that we use the wave 2 `argocd.argoproj.io/sync-wave: "2"` to make sure this job is executed after the kasten job.

```
cat <<EOF > migration-data-job.yaml 
apiVersion: batch/v1
kind: Job
metadata:
  name: migration-data-job
  annotations: 
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/sync-wave: "2"
spec:
  template:
    metadata:
      creationTimestamp: null
    spec:
      containers:
      - command: 
        - /bin/bash
        - -c
        - |
          #!/bin/bash
          # Oh no !! I forgot to the "where species in ${SPECIES}" clause in the delete command :(
          mysql -h mysql -p${MYSQL_ROOT_PASSWORD} -uroot -Bse "delete from test.pets" 
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              key: mysql-root-password
              name: mysql        
        - name: SPECIES
          valueFrom:
            configMapKeyRef:
              name: forbidden-species
              key: species      
        image: docker.io/bitnami/mysql:8.0.23-debian-10-r0
        name: data-job
      restartPolicy: Never
EOF 
git add migration-data-job.yaml
git commit -m "migrate the data to remove the forbidden species from the database, oh no I made a mistake, that will remove all the species !!" 
git push
```
## Phase 5 - The Recovery
At this stage we could roll back our ArgoCD to our previous version, prior to Phase 4 but you will notice that this just brings back our configuration and it is not going to bring back our data! 

Fortunately we can use kasten to restore the data using the restore point.

You will see from the above now when we check the database our data is gone! It was lucky that we have this presync enabled to take those backups prior to any code changes. We can now use that restore point to bring back our data. 

## Phase 6 - Making things right 
We have rectified our mistake in the code and would like to correctly implement this now into our application. 

```
cat <<EOF > migration-data-job.yaml 
apiVersion: batch/v1
kind: Job
metadata:
  name: migration-data-job
  annotations: 
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/sync-wave: "2"
spec:
  template:
    metadata:
      creationTimestamp: null
    spec:
      containers:
      - command: 
        - /bin/bash
        - -c
        - |
          #!/bin/bash
          # Oh no !! I forgot to the "where species in ${SPECIES}" clause in the delete command :(
          mysql -h mysql -p${MYSQL_ROOT_PASSWORD} -uroot -Bse "delete from test.pets where species in ${SPECIES}" 
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              key: mysql-root-password
              name: mysql        
        - name: SPECIES
          valueFrom:
            configMapKeyRef:
              name: forbidden-species
              key: species      
        image: docker.io/bitnami/mysql:8.0.23-debian-10-r0
        name: data-job
      restartPolicy: Never
EOF 
git add migration-data-job.yaml
git commit -m "migrate the data to remove the forbidden species from the database, oh no I made a mistake, that will remove all the species !!" 
git push
```

At this stage you will have your desired data in your database but peace of mind that you have a way of recovering if this accident happens again. 
