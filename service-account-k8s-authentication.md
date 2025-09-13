#  First create servvice account in kube-system namespace with below yml file
ci-cd-sa.yml
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-deployer-sa
  namespace: kube-system
```
# Run the below command for create service accout 
```
kubectl apply -f ci-cd-sa.yml
```
# Create cluster rolebinding with existing role (cluster-admin)
ci-cd-cluster-rolebinging.yml
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-deployer-cluster-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: jenkins-deployer-sa
    namespace: kube-system
```
# Run the below command for creating cluster rolebing 
```
kubectl apply -f ci-cd-cluster-rolebinging.yml
```
# Get the Token of service account running below command then after geting token we need to store in jenkins credentails as secret text with k8s-token-sa
```
kubectl create token jenkins-deployer-sa -n kube-system
```
# Get the cluster api end point running below command then we need to store in jenkins credentials as secrete text with k8s-server-api-end-point
```
kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}'
```
# Get CA certificate of cluster running with below command then we need to store in jenkins credentials as secrete file with k8s-ca-cert
```
kubectl config view --raw --minify -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' | base64 --decode > ca.crt
```
# Jenkinsfile like below 
```
pipeline {
    agent any
    stages {
        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    string(credentialsId: 'k8s-token-sa', variable: 'K8S_TOKEN'),
                    string(credentialsId: ' k8s-server-api-end-point', variable: 'K8S_SERVER'),
                    file(credentialsId: 'k8s-ca-cert', variable: 'K8S_CA_FILE')
                ]) {
                    sh '''
                    echo ">>> Setting up kubeconfig"
                    mkdir -p ~/.kube

                    # Cluster entry
                    kubectl config set-cluster my-dev-cluster \
                        --server=$K8S_SERVER \
                        --certificate-authority=$K8S_CA_FILE \
                        --embed-certs=true

                    # User credentials
                    kubectl config set-credentials jenkins-user --token=$K8S_TOKEN

                    # Context
                    kubectl config set-context my-dev-cluste-context \
                        --cluster=my-dev-cluste \
                        --user=jenkins-user

                    kubectl config use-context dev-context

                    echo ">>> Testing cluster access"
                    kubectl get nodes
                    kubectl apply -f my-app-ns.yml
                    kubectl apply -f k8s/deployment.yaml -n my-app-ns
                    '''
                }
            }
        }
    }
}
```



