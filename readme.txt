
This simple project is a two container thing -> somewhat like a notebook, where you enter Personal Data, birth date, and it shows you a list of entries, also upcoming birthdays. Created in training purposes. Prepared for building using docker compose, tested on Shipyard, prepared K8S manifests, all tested on Kind cluster.
Project's Docker part is developed in two variants, with authorisation and without authorisation. The second variant grew up to K8S.

Projects docker structure:   

notebook-app/
├── app/
│   ├── app.py
│   ├── requirements.txt
│   └── templates/
│       └── index.html
├── db/
│   └── Dockerfile
├── docker-compose.yml
└── app/
    └── Dockerfile
	
	note: docker compose up --build

Projects K8S Structure:

k8s/
 ├── flask-deployment.yaml
 ├── mysql-deployment.yaml
 ├── mysql-pv.yaml
 ├── mysql-pvc.yaml
 ├── mysql-service.yaml
 ├── flask-service.yaml
 ├── flask-autoscaling.yaml

	note: Can be run using all-in-one.yaml where all manifests included

Used images from personal docker hub repo:
ritchie229/ritchie_docker:notebook-app-web
ritchie229/ritchie_docker:notebook-app-db
	note: mandatory to run #cloud-provider-kind to make LoadBalancer service work properly, adviced to do docker sytem prune

Creation:
kubectl apply -f mysql-pv.yaml
kubectl apply -f mysql-pvc.yaml
kubectl apply -f mysql-deployment.yaml
kubectl apply -f mysql-service.yaml
kubectl apply -f flask-deployment.yaml
kubectl apply -f flask-service.yaml

Useful commands:
# kind load docker-image my-app:latest        > loading an image from the host to kind cluster nodes (all)

# eval $(minikube docker-env)
# docker build -t my-app:latest .             > the same thing for minicube.

                                             > Or enter any node and do "#docker load" или "#ctr image import" manually.

To see images inside nodes:
# docker exec -it kind-control-plane crictl images
# docker exec -it kind-worker crictl images
# docker exec -it kind-worker2 crictl images

# kubectl exec -it flask-app-5f846d7b97-mzqpk -- /bin/bash  > ssh into pod

# kubectl get pods -o wide                   > extended
# kubectl describe pod <pod_name>            > more extended

MANDATORY!!!------
Node Labels:
# kubectl get nodes --show-labels
# kubectl label node kind-worker type=worker
# kubectl label node kind-worker2 type=worker
# kubectl label node kind-worker node-role.kubernetes.io/worker=
# kubectl label node kind-worker2 node-role.kubernetes.io/worker=

NodeSelector(YAML):
spec:
  containers:
    ...
  nodeSelector:
    type: worker
------------------


LOGS:
# kubectl logs deployment/flask-app
# kubectl logs deployment/mysql

PORTFORWARDING:
# kubectl port-forward flask-app-5f846d7b97-8zst6 5000:5000 > port forwarded to a pod
# kubectl port-forward deployment/flask-app 5000:5000       > port forwarded to the deployment, pod are selected automatically

DB:
# kubectl exec -it mysql-d54547467-v9msm -- mysql -u user -p
mysql> SHOW DATABASES;
mysql> USE notebook;
mysql> SHOW TABLES;
mysql> SELECT * FROM person;


SCALING and AUTOSCALING:
# kubectl scale deployment flask-app --replicas=3
# kubectl scale deployment flask-app --replicas=1
# kubectl scale deploy -n <namespace> --replicas=0 --all    >  --all applies the scale to all deployments (in the namespace). 


# kubectl autoscale deployment flask-app --min=4 --max=5 --cpu-percent=80
# kubectl delete hpa flask-app


# kubectl rollout status deployment/flask-app
deployment "flask-app" successfully rolled out

# kubectl rollout history deployment/flask-app
deployment.apps/flask-app
REVISION  CHANGE-CAUSE
1         <none>

# kubectl set image deployment/flask-app <container_name>=<new_iamge:tag> --record    > set new image

# kubectl rollout undo deployment/flask-app                                           > one step back

# kubectl rollout undo deployment/flask-app --to-revision=<REVISION_NO>               > jump to preferred rev

# kubectl rollout restart deployment/flask-app                                        > update the image (in case it was changed in repo)


SERVICES:
# kubectl expose deploy flask-app --type=ClusterIP --port 80                          > access method to the deployment
# kubectl  delete service flask-app

# kubectl expose deploy flask-app --type=NodePort --port 80                          > access method to the deployment
# kubectl expose deploy flask-app --type=LoadBalancer --port 80




HOW TO REACH OUT TO WHAT YOU DEPLOY
1. # docker inspect kind-control-plane
2. Find Networks.kind.IPAddress
3. # kubectl get svc -o wide
4. use Right-Side-Port with the ip addr

or else

1. # kubectl get svc -o wide 
2. use EXTERNAL-IP and Left-Side-Port

eq. http://172.20.0.5:5000/ or http://172.20.0.4:32139/ 




INGRESS
# kubectl apply -f https://projectcontour.io/quickstart/contour.yaml
# kubectl get pods -n projectcontour -o wide
# kubectl get svc -n projectcontour -o wide
# kubectl get ingress
# kubectl decribe ingress

===EXAMPLE===
##### flask-service.yaml #####

apiVersion: v1
kind: Service
metadata:
  name: flask-service
spec:
  selector:
    app: flask         #Deploy sevice selects this label
  ports:
    - port: 5000       #port on Load Balancer
      targetPort: 5000 #port on POD
#  type: NodePort  # или LoadBalancer, если у тебя есть внешний балансировщик
#  type: LoadBalancer
  type: ClusterIP

##### iNgReSs #####

apiVersion: networking.k8s.io/v1 #beta1
kind: Ingress
metadata:
  name: ingress-host-n-path
    #  namespace: etcd-minikube
spec:
  rules:
  - host: flask.webapp.com
    http:
      paths:
        - path:
          pathType: ImplementationSpecific
          backend:
            service:
              name: flask-service
              port:
                number: 5000

  - host: webapp.com
    http:
      paths:
        - path: "/flask"
          pathType: ImplementationSpecific
          backend:
            service:
              name: flask-service
              port:
                number: 5000

### ArgoCD
kubectl apply -f argocd/notebook-app.yaml

