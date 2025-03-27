# EKS Setup with Amazon EBS CSI Driver and SonarQube

## Prerequisites

Ensure you have an **EKS cluster** and **worker nodes** set up before proceeding.

## Steps

### 1. Create IAM Policy for Amazon EBS

Create an IAM policy with the following details:

#### **Amazon\_EBS\_CSI\_Driver**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:AttachVolume",
        "ec2:CreateSnapshot",
        "ec2:CreateTags",
        "ec2:CreateVolume",
        "ec2:DeleteSnapshot",
        "ec2:DeleteTags",
        "ec2:DeleteVolume",
        "ec2:DescribeInstances",
        "ec2:DescribeSnapshots",
        "ec2:DescribeTags",
        "ec2:DescribeVolumes",
        "ec2:DetachVolume"
      ],
      "Resource": "*"
    }
  ]
}
```

### 2. Attach Policy to Worker Nodes IAM Role

Retrieve the IAM role for worker nodes:

```sh
kubectl -n kube-system describe configmap aws-auth
```

Sample output:

```
rolearn: arn:aws:iam::180789647333:role/eksctl-eksdemo1-nodegroup-eksdemo-NodeInstanceRole-IJN07ZKXAWNN
```

Attach the **Amazon\_EBS\_CSI\_Driver** policy to the extracted role.

### 3. Deploy Amazon EBS CSI Driver

Install the EBS CSI Driver using:

```sh
kustomize build "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.41" | kubectl apply -f -
```

Verify that the EBS CSI pods are running:

```sh
kubectl get pods -n kube-system
```

### 4. Deploy Storage Class, Persistent Volume, PostgreSQL, and SonarQube

#### **Storage Class**

``

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
```

#### **Persistent Volume Claim**

``

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pv-claim
  namespace: sonarqube
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 4Gi
```

#### **PostgreSQL Deployment**

``

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: sonarqube
data:
  POSTGRES_DB: sonarqube
  POSTGRES_USER: sonar
  POSTGRES_PASSWORD: sonarpassword
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: sonarqube
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
          image: postgres:15
          envFrom:
            - configMapRef:
                name: postgres-config
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data
              subPath: postgres
      volumes:
        - name: postgres-storage
          persistentVolumeClaim:
            claimName: postgres-pv-claim
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: sonarqube
spec:
  ports:
    - port: 5432
      targetPort: 5432
  selector:
    app: postgres
  type: ClusterIP
```

#### **SonarQube Deployment**

``

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sonarqube
  namespace: sonarqube
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
          image: sonarqube:community
          env:
            - name: SONAR_JDBC_URL
              value: "jdbc:postgresql://postgres:5432/sonarqube"
            - name: SONAR_JDBC_USERNAME
              value: "sonar"
            - name: SONAR_JDBC_PASSWORD
              value: "sonarpassword"
          ports:
            - containerPort: 9000
---
apiVersion: v1
kind: Service
metadata:
  name: sonarqube
  namespace: sonarqube
spec:
  type: NodePort
  selector:
    app: sonarqube
  ports:
    - protocol: TCP
      port: 9000
      targetPort: 9000
      nodePort: 32085
```

## Conclusion

You have now set up Amazon EBS CSI Driver and deployed PostgreSQL and SonarQube on your **EKS cluster**. 🎉

