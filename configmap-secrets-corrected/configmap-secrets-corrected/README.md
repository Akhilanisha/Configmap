# Weshopify on Kubernetes

This repository deploys the existing Spring Boot image and a single-replica MySQL database. The database uses a StatefulSet and a dynamically provisioned EBS volume on standard Amazon EKS.

## Before deployment

- Connect `kubectl` to your EKS cluster and check `kubectl get nodes`.
- Confirm that the Amazon EBS CSI driver is installed and that your cluster can provision EBS volumes.
- This example's StorageClass is for standard EKS with the EBS CSI driver (`ebs.csi.aws.com`). EKS Auto Mode uses a different provisioner. If your cluster already has an appropriate StorageClass, you can use it instead of `3_eks-gp3-storageclass.yml`; update the `storageClassName` in the StatefulSet to match.
- Confirm that `vinodscope/spring-boot-mysql:latest` exists and that the application reads the `DB_*` environment variables below. For repeatable deployments, use a versioned application image tag.

## Create the namespace and database Secret

Run these commands in PowerShell. Replace both example passwords before running them. This creates the Secret directly in the cluster; do not add real passwords to a YAML file or commit them to Git.

```powershell
kubectl create namespace weshopify

kubectl create secret generic weshopify-db-secret `
  --namespace weshopify `
  --from-literal=mysql-root-password='ReplaceWithARootPassword' `
  --from-literal=mysql-app-password='ReplaceWithAnAppPassword'
```

The command line may be saved in shell history. Use AWS Secrets Manager or another managed secrets solution for real workloads.

## Create or select storage

Check available classes:

```powershell
kubectl get storageclass
```

For standard EKS, apply the included StorageClass if you do not already have a suitable EBS CSI class:

```powershell
kubectl apply -f 3_eks-gp3-storageclass.yml
```

If the class already exists, do not recreate it. Instead, make `storageClassName` in `4_weshopify-db-statefulset.yml` match the class you intend to use.

## Deploy

Apply the ConfigMap, database StatefulSet, and app Deployment:

```powershell
kubectl apply -f 1_weshopify-db-configmap.yml --namespace weshopify
kubectl apply -f 4_weshopify-db-statefulset.yml --namespace weshopify
kubectl apply -f 5_app-deployment.yml --namespace weshopify
```

Watch the pods and storage claim:

```powershell
kubectl get pods,pvc,services --namespace weshopify --watch
```

Wait until the MySQL pod and the app pod are `Running` and the PVC is `Bound`. Press `Ctrl+C` to stop watching.

## Open the application locally

```powershell
kubectl port-forward service/weshopify-app 8080:8080 --namespace weshopify
```

Keep the command running, then open `http://localhost:8080`. The application's page path depends on the Spring Boot code.

## Troubleshooting

```powershell
kubectl describe pod --selector app=weshopify-db --namespace weshopify
kubectl logs statefulset/weshopify-db --namespace weshopify
kubectl logs deployment/weshopify-app --namespace weshopify
kubectl describe pvc data-weshopify-db-0 --namespace weshopify
```

A PVC stuck in `Pending` usually points to the EBS CSI driver, its permissions, or a StorageClass mismatch. If the app cannot connect to MySQL, check the MySQL logs and confirm that `DB_HOST` is `weshopify-db`.

## Files removed from the original version

The old Secret manifest with `root`/`root` values, the hostPath PersistentVolume, the manually pinned PVC, the competing MySQL Deployment, and the empty StatefulSet file should be removed. The single StatefulSet below owns its PVC through `volumeClaimTemplates`.
