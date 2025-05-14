

Nom : AMANI Kouakou Jaures De Sales

ESGI 5SRC2 


# PROJET DEVOPS - Orchestration

## Introduction

La société IC GROUP souhaite mettre en place un site web vitrine donnant accès à deux de ses applications principales : Odoo et pgAdmin. Le projet implique la conteneurisation d'une application web Flask, le déploiement des services dans un cluster Kubernetes (Minikube), la gestion de la persistance des données, et l’orchestration de l’ensemble.

- Odoo est un ERP open-source qui permet de gérer de nombreuses fonctions d’entreprise. La version 13.0 a été retenue car elle intègre un LMS utile pour la diffusion de formations internes.
- pgAdmin est une interface web de gestion de base de données PostgreSQL. Elle permettra l’administration de la base utilisée par Odoo.

Le site vitrine codé avec Flask permet l'accès centralisé à ces deux applications via des liens dynamiques définis par variables d’environnement.

## Pré-requis

## Installer les outils nécessaires
sudo apt update && sudo apt install -y docker.io kubectl minikube git

## Démarrer Minikube
minikube start
kubectl config use-context minikube

## Clonage et conteneurisation de l'application vitrine Flask
Cloner le dépôt

git clone https://github.com/sadofrazer/ic-webapp.git
cd ic-webapp

## Étape I - Clonage et conteneurisation de l'application web vitrine ic-web

1. Création du Dockerfile

```bash
FROM python:3.6-alpine
WORKDIR /opt
COPY . /opt
RUN pip install flask==1.1.2
ENV ODOO_URL=https://www.odoo.com
ENV PGADMIN_URL=https://www.pgadmin.org
EXPOSE 8080

ENTRYPOINT ["python", "app.py"]
```

![image](https://github.com/Jauresamani/esgis2projetfinal/blob/main/ScreenREADME/1dockerfile.png)


2. Fichier requirements.txt

Flask

3. Build de l’image

docker build -t ic-webapp:1.0 .

![image](https://github.com/Jauresamani/esgis2projetfinal/blob/main/ScreenREADME/2image.png)
4. Test local

docker run -d --name test-ic-webapp -p 8080:8080 -e ODOO_URL=https://www.odoo.com -e PGADMIN_URL=https://www.pgadmin.org ic-webapp:1.0

![image](https://github.com/Jauresamani/esgis2projetfinal/blob/main/ScreenREADME/3docker.png)


5. Suppression du container de test

docker rm -f test-ic-webapp

![image](https://github.com/Jauresamani/esgis2projetfinal/blob/main/ScreenREADME/4dockerrm.png)

6. Push sur Docker Hub

docker tag ic-webapp:1.0 kamani3/ic-webapp:1.0
docker push kamani3/ic-webapp:1.0

![image](https://github.com/Jauresamani/esgis2projetfinal/blob/main/ScreenREADME/5dockerpush.png)


![image](https://github.com/Jauresamani/esgis2projetfinal/blob/main/ScreenREADME/14dockerhub.png)


## Étape II - Déploiement Kubernetes

1. Namespace

```bash
apiVersion: v1
kind: Namespace
metadata:
  name: icgroup
  labels:
    env: prod
```

kubectl apply -f namespace.yaml

![image](https://github.com/Jauresamani/esgis2projetfinal/blob/main/ScreenREADME/6namespace.png)


## Étape III - Déploiement PostgreSQL

1. Déploiement + PVC

PVC
```bash
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: icgroup
  labels:
    env: prod
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```
kubectl apply -f postgres-pvc.yaml

Déploiement

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: icgroup
  labels:
    env: prod
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:13
          envFrom:
            - configMapRef:
                name: postgres-config
          ports:
            - containerPort: 5432
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: postgres-storage
      volumes:
        - name: postgres-storage
          persistentVolumeClaim:
            claimName: odoo-pvc
```
kubectl apply -f postgres-deployment.yaml

![image](https://github.com/Jauresamani/esgis2projetfinal/blob/main/ScreenREADME/7posgres.png)



## Étape IV - Déploiement Odoo

PVC 

```bash
apiVersion: v1
kind: PersistentVolume
metadata:
  name: odoo-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data/odoo"  

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: odoo-pvc
  namespace: icgroup
  labels:
    env: prod
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```
kubectl apply -f odoo-pvc.yaml


Deploiement 

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: odoo
  namespace: icgroup
  labels:
    env: prod
spec:
  replicas: 2
  selector:
    matchLabels:
      app: odoo
  template:
    metadata:
      labels:
        app: odoo
    spec:
      securityContext:
        fsGroup: 1000
      containers:
        - name: odoo
          image: odoo:13.0
          args: ["-i", "base"]
          securityContext:
            runAsUser: 0
          env:
            - name: HOST
              value: postgres
            - name: USER
              value: odoo
            - name: PASSWORD
              value: odoo
          ports:
            - containerPort: 8069
          volumeMounts:
            - mountPath: /var/lib/odoo
              name: odoo-storage
      volumes:
        - name: odoo-storage
          persistentVolumeClaim:
            claimName: odoo-pvc
```
kubectl apply -f odoo-deployment.yaml 

![image](https://github.com/Jauresamani/esgis2projetfinal/blob/main/ScreenREADME/9odoo-depl.png)




## Étape V - Déploiement pgAdmin avec configuration

1. Création d’un ConfigMap avec servers.json

fichiers servers.json

```bash
{
  "Servers": {
    "1": {
      "Name": "Odoo PostgreSQL",
      "Group": "Servers",
      "Host": "postgres",
      "Port": 5432,
      "MaintenanceDB": "odoo",
      "Username": "odoo",
      "SSLMode": "prefer"
    }
  }
}
```

kubectl create configmap pgadmin-config --from-file=servers.json=servers.json -n icgroup


```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: pgadmin-config
  namespace: icgroup
data:
  servers.json: |
    {
      "Servers": {
        "1": {
          "Name": "Odoo PostgreSQL",
          "Group": "Servers",
          "Host": "postgres",
          "Port": 5432,
          "MaintenanceDB": "odoo",
          "Username": "odoo",
          "SSLMode": "prefer"
        }
      }
    }
```




2. Déploiement pgAdmin avec volume

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pgadmin
  namespace: icgroup
  labels:
    env: prod
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pgadmin
  template:
    metadata:
      labels:
        app: pgadmin
    spec:
      containers:
        - name: pgadmin
          image: dpage/pgadmin4
          env:
            - name: PGADMIN_DEFAULT_EMAIL
              value: admin@icgroup.com
            - name: PGADMIN_DEFAULT_PASSWORD
              value: admin
          ports:
            - containerPort: 80
          volumeMounts:
            - name: pgadmin-config
              mountPath: /pgadmin4/servers.json
              subPath: servers.json
      volumes:
        - name: pgadmin-config
          configMap:
            name: pgadmin-config
```
kubectl apply -f pgadmin-deployment.yaml

![image](https://github.com/Jauresamani/esgis2projetfinal/blob/main/ScreenREADME/10pgadmindep.png)



Étape VII - Services & accès

```bash
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: icgroup
  labels:
    env: prod
spec:
  selector:
    app: postgres
  type: ClusterIP
  ports:
    - port: 5432
      targetPort: 5432
---
apiVersion: v1
kind: Service
metadata:
  name: odoo-service
  namespace: icgroup
  labels:
    env: prod
spec:
  selector:
    app: odoo
  type: NodePort
  ports:
    - name: http
      protocol: TCP
      port: 8069
      targetPort: 8069
      nodePort: 31987
---
apiVersion: v1
kind: Service
metadata:
  name: pgadmin-service
  namespace: icgroup
  labels:
    env: prod
spec:
  selector:
    app: pgadmin
  type: NodePort
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30975
---
apiVersion: v1
kind: Service
metadata:
  name: ic-webapp-service
  namespace: icgroup
  labels:
    env: prod
spec:
  selector:
    app: ic-webapp
  type: NodePort
  ports:
    - name: http
      protocol: TCP
      port: 8080
      targetPort: 8080
      nodePort: 31276
```

kubectl apply -f services.yaml

![image](https://github.com/Jauresamani/esgis2projetfinal/blob/main/ScreenREADME/10pgadmindep.png)


## Étape VI - Déploiement de l’ic-webapp dans le cluster


```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ic-webapp
  namespace: icgroup
  labels:
    env: prod
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ic-webapp
  template:
    metadata:
      labels:
        app: ic-webapp
    spec:
      containers:
      - name: ic-webapp
        image: kamani3/ic-webapp:1.0
        ports:
        - containerPort: 8080
        env:
        - name: ODOO_URL
          value: http://192.168.49.2:31987
        - name: PGADMIN_URL
          value: http://192.168.49.2:30975
```
kubectl apply -f ic-webapp-deployment.yaml 

![image](https://github.com/Jauresamani/esgis2projetfinal/blob/main/ScreenREADME/12icwe.png)







Étape VIII - Vérifications et conclusion

1. Vérifier les pods et services

kubectl get all -n icgroup

![image](https://github.com/Jauresamani/esgis2projetfinal/blob/main/ScreenREADME/11getall.png)


![image](https://github.com/user-attachments/assets/ec17b6da-213e-4528-8abf-36679f42ddac)


2. Conclusion

Grâce à cette orchestration complète dans Minikube, la société IC GROUP dispose désormais d’une solution robuste et automatisée pour héberger Odoo, pgAdmin ainsi qu’un site web Flask comme vitrine. L’utilisation de Kubernetes assure l’évolutivité et la résilience des services.


