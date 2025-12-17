# Kubernetes Deployment Guide for Picture Bed Application

This directory contains Kubernetes deployment configurations for the Picture Bed (图床共享云存储) application.

## Architecture Overview

The application consists of the following components:
- **MySQL**: Database for storing user information, file metadata, and sharing information
- **Redis**: Cache for session tokens and counters
- **FastDFS Tracker**: Distributed file system tracker (2 replicas) with improved load balancing
- **FastDFS Storage**: Distributed file system storage (2 replicas) with performance monitoring
- **FastCGI Backend**: C++ backend application handling API requests (2 replicas)
- **AI Search**: AI-powered search service (1 replica)
- **Nginx**: Web server serving frontend and proxying requests (2 replicas)
- **Ingress**: Route external traffic to appropriate services

### FastDFS Improved Load Balancing

This deployment implements an improved load balancing algorithm for FastDFS that addresses the limitations of the original maximum remaining space algorithm. 

**Original Algorithm Limitations**:
- Only considered group remaining disk space
- Ignored storage server performance factors
- Could lead to uneven load distribution

**Improved Algorithm Features**:
- **Disk Performance Monitoring**: Tracks disk I/O performance statistics every 60 seconds
- **Task Utilization Tracking**: Monitors current task load on each storage server
- **Comprehensive Selection Criteria**: Combines remaining space, disk performance, and task utilization to select optimal storage servers
- **Dynamic Load Distribution**: Adapts to real-time server performance and load conditions

Configuration options:
- `FDFS_STORE_LOOKUP=2`: Enables improved load balancing (0=round robin, 1=max free space, 2=improved)
- `FDFS_DISK_STAT_INTERVAL=60`: Disk performance statistics collection interval (seconds)
- `FDFS_SYNC_STAT_INTERVAL=300`: Statistics synchronization interval (seconds)

## Deployment Order

Follow this order to deploy the application successfully:

### 1. Storage (PersistentVolumeClaims)
```bash
kubectl apply -f 01-pvc.yaml
```

This creates persistent volume claims for:
- `mysql-pvc`: 10Gi storage for MySQL data
- `redis-pvc`: 5Gi storage for Redis data
- `fastdfs-tracker-pvc`: 5Gi storage for FastDFS tracker
- `fastdfs-storage-pvc`: 50Gi storage for FastDFS file storage

### 2. MySQL and Redis
```bash
kubectl apply -f 02-mysql.yaml
kubectl apply -f 03-redis.yaml
```

Deploy the data layer services:
- MySQL StatefulSet (1 replica) with headless service on port 3306
- Redis StatefulSet (1 replica) with headless service on port 6379

Wait for both services to be ready before proceeding:
```bash
kubectl wait --for=condition=ready pod -l app=mysql --timeout=300s
kubectl wait --for=condition=ready pod -l app=redis --timeout=300s
```

### 3. FastDFS Tracker and Storage
```bash
kubectl apply -f 04-fastdfs-tracker.yaml
kubectl apply -f 05-fastdfs-storage.yaml
```

Deploy the distributed file system with improved load balancing:
- FastDFS Tracker Deployment (2 replicas) on port 22122
- FastDFS Storage StatefulSet (2 replicas) on ports 22000 and 23000

**Improved Load Balancing Algorithm**: The FastDFS configuration uses an enhanced load balancing strategy that considers:
1. **Disk Performance**: Monitors storage server disk I/O performance
2. **Task Utilization**: Tracks current task load on each storage server
3. **Remaining Space**: Still considers available disk space (not solely relied upon)

This addresses the limitations of the original maximum remaining space algorithm, which only considered group remaining space and ignored storage server performance factors.

Wait for tracker to be ready before storage:
```bash
kubectl wait --for=condition=ready pod -l app=fastdfs-tracker --timeout=300s
kubectl wait --for=condition=ready pod -l app=fastdfs-storage --timeout=300s
```

### 4. FastCGI Backend
```bash
kubectl apply -f 06-fastcgi-backend.yaml
```

Deploy the backend API service:
- FastCGI Backend Deployment (2 replicas)
- Service `fastcgi-svc` exposing port 9000

Wait for the backend to be ready:
```bash
kubectl wait --for=condition=ready pod -l app=fastcgi-backend --timeout=300s
```

### 5. AI Search
```bash
kubectl apply -f 07-ai-search.yaml
```

Deploy the AI search service:
- AI Search Deployment (1 replica) on port 8080

### 6. Nginx and Ingress
```bash
kubectl apply -f 08-nginx.yaml
kubectl apply -f 09-ingress.yaml
```

Deploy the frontend web server and ingress:
- Nginx Deployment (2 replicas) on port 80
- Ingress routing traffic to appropriate services

## Service Endpoints

After deployment, the following internal services will be available:

| Service | Port | Purpose |
|---------|------|---------|
| mysql-svc | 3306 | MySQL database |
| redis-svc | 6379 | Redis cache |
| fastdfs-tracker-svc | 22122 | FastDFS tracker |
| fastdfs-storage-svc | 22000, 23000 | FastDFS storage |
| fastcgi-svc | 9000 | Backend API |
| ai-search-svc | 8080 | AI search service |
| nginx-svc | 80 | Frontend web server |

## Ingress Routes

Access the application through the ingress at `cloud.example.com`:

| Path | Service | Port | Description |
|------|---------|------|-------------|
| `/api` | fastcgi-svc | 9000 | Backend API endpoints |
| `/download` | fastdfs-storage-svc | 23000 | File download service |
| `/` | nginx-svc | 80 | Frontend application |

## Verify Deployment

Check all pods are running:
```bash
kubectl get pods
```

Check all services:
```bash
kubectl get services
```

Check ingress:
```bash
kubectl get ingress
```

## Database Initialization

After MySQL is running, you may need to initialize the database schema. You can exec into the MySQL pod and import the schema:

```bash
# Copy the SQL file to the MySQL pod
kubectl cp tuchuang.sql <mysql-pod-name>:/tmp/tuchuang.sql

# Execute the SQL file
kubectl exec -it <mysql-pod-name> -- mysql -uroot -pqwer < /tmp/tuchuang.sql
```

## Configuration Notes

- **Storage Class**: All PVCs use the `standard` storage class. Adjust based on your cluster's available storage classes.
- **Passwords**: Default MySQL password is `qwer` (matching the original application configuration). **Important**: For production deployments, use Kubernetes Secrets instead of plain-text passwords:
  ```bash
  # Create a secret for MySQL password
  kubectl create secret generic mysql-secret --from-literal=root-password=your-secure-password
  
  # Reference it in your deployment using secretKeyRef instead of value
  ```
- **Image Names**: Replace placeholder image names (e.g., `placeholder/mysql`) with actual Docker images.
- **Host Name**: Update `cloud.example.com` in the ingress configuration to your actual domain.
- **Port Mapping**: The FastCGI backend service maps external port 9000 to the internal container port 8081 for ingress routing compatibility.
- **Resource Limits**: Consider adding resource requests and limits for production deployments.

## Cleanup

To remove all deployed resources:
```bash
kubectl delete -f 09-ingress.yaml
kubectl delete -f 08-nginx.yaml
kubectl delete -f 07-ai-search.yaml
kubectl delete -f 06-fastcgi-backend.yaml
kubectl delete -f 05-fastdfs-storage.yaml
kubectl delete -f 04-fastdfs-tracker.yaml
kubectl delete -f 03-redis.yaml
kubectl delete -f 02-mysql.yaml
kubectl delete -f 01-pvc.yaml
```

## Troubleshooting

### Pods not starting
```bash
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

### Service not accessible
```bash
kubectl get endpoints
kubectl describe service <service-name>
```

### Storage issues
```bash
kubectl describe pvc <pvc-name>
kubectl get pv
```

## Additional Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [FastDFS Documentation](https://github.com/happyfish100/fastdfs)
- [Nginx Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
