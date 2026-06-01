gcloud compute networks create game-vpc \
  --subnet-mode=custom


#creating Subnet

gcloud compute networks subnets create game-subnet \
  --network=game-vpc \
  --region=us-east1 \
  --range=10.0.1.0/24

#firewall rules

# Allow port 80 from internet
gcloud compute firewall-rules create allow-http \
  --network=game-vpc \
  --allow=tcp:80 \
  --source-ranges=0.0.0.0/0

# Allow internal pod communication
gcloud compute firewall-rules create allow-internal \
  --network=game-vpc \
  --allow=tcp,udp,icmp \
  --source-ranges=10.0.1.0/24

# Block everything else
gcloud compute firewall-rules create deny-all \
  --network=game-vpc \
  --action=DENY \
  --rules=all \
  --priority=65534


  #MAKING CLUSTER
  gcloud container clusters create game-cluster \
  --machine-type=e2-medium \
  --zone=us-east1-b \
  --network=game-vpc \
  --subnetwork=game-subnet \
  --spot \
  --num-nodes=1 \
  --node-locations=us-east1-b,us-east1-c,us-east1-d \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=5

#VERIFY CREDENTIALS
  gcloud container clusters get-credentials game-cluster \
  --zone=us-east1-b

#DEPLOYMENT FILE

cat << 'EOF' > 2048-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: game-2048
spec:
  replicas: 2
  selector:
    matchLabels:
      app: game-2048
  template:
    metadata:
      labels:
        app: game-2048
    spec:
      containers:
      - name: game-2048
        image: public.ecr.aws/l6m2t8p7/docker-2048:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 3
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: game-2048
            topologyKey: kubernetes.io/hostname
EOF

#SERVICE FILE
cat << 'EOF' > 2048-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: game-2048-service
spec:
  type: LoadBalancer
  selector:
    app: game-2048
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80

kubectl apply -f 2048-deployment.yaml
kubectl apply -f 2048-service.yaml

kubectl autoscale deployment game-2048 \
  --min=1 \
  --max=10 \
  --cpu-percent=60

kubectl get services
kubectl get pods
  
