# WORDPRESS with MYSQL and PHPMYADMIN in Kubernetes

### create 64bit password
```
echo -n 'mypassword' | base64
```

### secrets
create a file named project-secrets.yaml
```
apiVersion: v1
kind: Secret
metadata:
  name: project-secrets
type: Opaque
data:
  root-password: password
```

deploy:
```
kubectl create -f project-secrets.yaml

```
or a  kustomization.yaml
```
secretGenerator:
- name: mysql-pass
  literals:
  - password=mypassword

resources:
  - project-secrets.yaml
  - mysql-pv.yaml
  - mysql-service.yaml
  - mysql-deployment.yaml
  - phpmyadmin-service.yaml
  - phpmyadmin-deployment.yaml
```
apply:
```
kubectl apply -k ./
```

delete
```
kubectl delete -k ./
```
## MYSQL
### create Persistent Volume

create a file named mysql-pv.yaml
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

and deploy
```
kubectl create -f mysql-pv.yaml
```
check if volumes is created:
```
kubectl get pv

kubectl describe pv mysql-pv-volume
```

### service
create a file named mysql-service.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - protocol: TCP
    port: 3306
    targetPort: 3306
  selector:
    app: mysql
  clusterIP: None
```
deploy:
```
kubectl create -f mysql-service.yaml
```

check if created:
```
kubectl get service

kubectl get svc

kubectl describe svc mysql
```

### deployments
create a file named mysql-deployment.yaml
```
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:5.7
        name: mysql
        env:
          # Use secret in real usage
        - name: MYSQL_ROOT_PASSWORD
          value: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
```
deploy:
```
kubectl create -f mysql-deployment.yaml
```

check:
```
kubectl get pods
```

enter in mysql shell:
```
kubectl run -it --rm --image=mysql:5.6 --restart=Never mysql-client -- mysql -h mysql -ppassword
```

## PHPMYADMIN

### service
create a file named phpmyadmin-service.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: phpmyadmin-service
spec:
  type: NodePort
  selector:
    app: phpmyadmin
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

deploy:
```
kubectl create -f phpmyadmin-service.yaml 
```

check:
```
kubectl get svc

kubectl describe svc phpmyadmin-service
```

### deployment
create a file named phpmyadmin-deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: phpmyadmin-deployment
  labels:
    app: phpmyadmin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: phpmyadmin
  template:
    metadata:
      labels:
        app: phpmyadmin
    spec:
      containers:
        - name: phpmyadmin
          image: phpmyadmin/phpmyadmin
          ports:
            - containerPort: 80
          env:
            - name: PMA_HOST
              value: mysql # nome del servizio mysql
            - name: PMA_PORT
              value: "3306"
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: project-secrets
                  key: root-password
```
deploy:
```
kubectl create -f phpmyadmin-deployment.yaml
```

check:
```
kubectl get pods
```

check on the browser:
```
minikube service phpmyadmin-service --url
```

## WORDPRESS
### Persistent Volume
create a file named wordpress-pv.yaml
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: wordpress-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/var/www/html"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

deploy:
```
kubectl create -f wordpress-pv.yaml
```

### Service
create a file named wordpress-service.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  ports:
    - port: 80
  selector:
    app: wordpress
    tier: frontend
  type: LoadBalancer
```

deploy:
```
kubectl create -f wordpress-service.yaml
```

### Deployment
create a file named wordpress-deployment.yaml
```
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
      - image: wordpress:4.8-apache
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: mysql
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 80
          name: wordpress
        volumeMounts:
        - name: wordpress-persistent-storage
          mountPath: /var/www/html
      volumes:
      - name: wordpress-persistent-storage
        persistentVolumeClaim:
          claimName: wordpress-pv-claim
```

deploy:
```
kubectl create -f wordpress-deployment.yaml
```

## delete
```
kubectl delete -f project-secrets.yaml

kubectl delete -f mysql-pv.yaml

kubectl delete -f mysql-service.yaml

kubectl delete -f mysql-deployment.yaml

kubectl delete -f phpmyadmin-service.yaml 

kubectl delete -f phpmyadmin-deployment.yaml

kubectl delete -f wordpress-pv.yaml

kubectl delete -f wordpress-service.yaml

kubectl delete -f wordpress-deployment.yaml
```