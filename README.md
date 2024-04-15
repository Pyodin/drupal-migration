# Migration of Drupal to Kubernetes
This document describes the steps to migrate a Drupal application to Kubernetes.

## Prerequisites
- A running *Drupal* application running on docker
- A *Kubernetes* cluster
- *Cert-manager* installed in your Kubernetes cluster (if you want to use https)
- A *ClusterIssuer* named `ca-issuer` (if you want to use https)
- *Helm* installed in your Kubernetes cluster

## Create the new Drupal deployment
### Deploy the Drupal application
- Add the *Bitnami* repository
    ```bash
    helm repo add bitnami https://charts.bitnami.com/bitnami
    helm repo update
    ```
We will create a new deployment called *my-drupal* in the *drupal* namespace. The following command will create a new deployment using a *ClusterIP* service type either with an Ingress object or an httpproxy object. In both cases, the deployment will have https enabled.

#### Using Ingress 
If you want to use an Ingress object to expose your application, and assuming you have a cert-manager installed in your cluster, you can use the following command:
```bash
helm install my-drupal bitnami/drupal \
--namespace drupal --create-namespace \
--set service.type=ClusterIP \
--set ingress.enabled=true \
--set ingress.hostname=<your_domain> \
--set ingress.tls[0].hosts[0]=<your_domain> \
--set ingress.tls[0].secretName=drupal-tls \
--set ingress.annotations."cert-manager.io/cluster-issuer"=ca-issuer
```

#### Using httpproxy
As I am working on a Tanzu cluster, my ingress controller is Contour and it does not support Ingress objects in my cluster. Therefore, I will use an httpproxy object to expose my application. If you are using a different ingress controller, you can use an Ingress object instead.
```bash
helm install my-drupal bitnami/drupal \
--namespace drupal \
--create-namespace \
--set service.type=ClusterIP
```
This command will create a new deployment called *my-drupal* in the *drupal* namespace. 
Now, if you want to expose the application using an *httpproxy* object using https, follow the steps below:
- If you don't already have one, create a self-signed *ClusterIssuer*
    ```bash
    kubectl apply -f clusterissuer.yaml
    ```
- Create a *certificate* using your ClusterIssuer
    ```bash
    kubectl apply -f certificate.yaml
    ```
- Create a new *Httpproxy* object
    ```bash
    kubectl apply -f httpproxy.yaml
    ```

#### Troubleshooting
As you are migrating the Drupal application, you might want to use the same database password for the old and the new deployment. To do so, you will want to add the following argument to the helm install command:
```bash
--set mariadb.auth.rootPassword=<old_password>
```

#### Get the Drupal login credentials
Once the deployment is created, you can get the Drupal login credentials by running:
```bash
echo Username: user
echo Password: $(kubectl get secret --namespace drupal my-drupal -o jsonpath="{.data.drupal-password}" | base64 -d)
``` 

### Migrate the old Drupal site and database to the new deployment
As we are migrating the Drupal application, we need to backup the old site and database. We will then restore them in the new deployment.
#### Create a backup of the old site and database
- We first need to get the container id of the old deployment:
    ```bash
    docker ps
    ```
- Then, we need to copy the site folder to a local directory:
    ```bash
    docker cp <container_id>:/var/www/html /tmp
    ```
- We also need to backup the database. The command looks like this:
    ```bash
    docker exec CONTAINER_ID mysqldump -u USER -pPASSWORD DATABASE_NAME > backup.sql
    ```

#### Restore the site and database in the new deployment
##### Restore the site:
Use kubectl cp to copy your Drupal site backup into your Drupal pod. You'll need to copy the files into the appropriate directory inside the pod, typically something like /var/www/html.
```bash
kubectl cp /tmp/drupal.sql drupal/<pod_name>:/var/www/html
```
##### Restore the Database:
You can restore your database backup to the Kubernetes-managed database. One way to do this is by using kubectl exec to run mysql commands inside your database pod.
The command looks like this:
```bash	
kubectl -n drupal exec -i DATABASE_POD_NAME -- mysql -u USER -pPASSWORD DATABASE_NAME < backup.sql
```
In order to get the mariadb password, you can run the following command:
```bash
echo username: root
echo Password: $(kubectl get secret -n drupal drupal-mariadb -o jsonpath="{.data.mariadb-root-password}" |base64 -d)
```

## Conclusion
You have successfully migrated your Drupal application to Kubernetes. You can now access your application using the URL you defined in the Ingress or httpproxy object.