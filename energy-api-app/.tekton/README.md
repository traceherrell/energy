# **.NET GitOps CI/CD Pipeline for OpenShift**

This directory contains all the necessary Tekton resources to create a fully automated CI/CD pipeline for a .NET application. The pipeline follows GitOps best practices.


## **Pipeline Workflow**

The pipeline automates the following workflow:

1. **Trigger**: Automatically starts when code is pushed to the application's GitHub repository.
2. **Clone & Build**: Clones the application source code and uses buildah to build a new container image from the Dockerfile.
3. **Push to Registry**: Pushes the newly built container image to a Quay.io repository.
4. **Update Deployment Repo**: Clones a separate deployment repository (used by Argo CD), updates the Kubernetes deployment manifest with the new image digest (e.g., image: quay.io/my-org/my-app@sha256:...), and pushes the change back to the deployment repository.
5. **Argo** CD** Sync**: The push to the deployment repository triggers Argo CD to automatically sync the new version of the application to the OpenShift cluster.


## **1. Prerequisites**

Before you can use this pipeline, you must create several resources on your OpenShift cluster. These steps only need to be done once.


### **a. Quay.io Credentials Secret**

This secret allows the pipeline to push the container image to your Quay.io repository. It's recommended to use a [Robot](https://docs.quay.io/glossary/robot-accounts.html) Account token for the password.

# Replace with your actual Quay.io credentials/robot token 
```
oc create secret docker-registry quay-credentials \ 
  --docker-server=quay.io \ 
  --docker-username="your-quay-username" \ 
  --docker-password="your-quay-robot-token" 
```



### **b. Git Deploy Repo Credentials Secret**

This secret allows the pipeline to commit and push changes to your Argo CD deployment repository. Use a [GitHub Personal Access Token (PAT)](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token) with repo scope for the password.

# Replace with your Git username and Personal Access Token 
```
oc create secret generic git-deploy-credentials \ 
  --from-literal=username=your-git-username \ 
  --from-literal=password=your-git-personal-access-token 
```


### **c. Service Account**

Create a dedicated ServiceAccount for the pipeline and link the secrets to it. This gives the pipeline the necessary permissions to access Quay.io and GitHub.

# Create the ServiceAccount 
```
oc create serviceaccount pipeline-sa 
```
 
# Link the secrets to the ServiceAccount 
```
oc patch serviceaccount pipeline-sa \ 
  -p '{"secrets": [{"name": "quay-credentials"}, {"name": "git-deploy-credentials"}]}' 
```



### **d. Persistent Volume Claim (PVC)**

The pipeline needs a workspace to store cloned repositories and other artifacts. Create a PersistentVolumeClaim for this purpose.

# Create a file named tekton-pvc.yaml 
```yaml
apiVersion: v1 
kind: PersistentVolumeClaim 
metadata: 
  name: tekton-pvc 
spec: 
  accessModes: 
    - ReadWriteOnce 
  resources: 
    requests: 
      storage: 1Gi 
```


Apply it with:``` oc apply -f tekton-pvc.yaml```


## **2. Pipeline Installation**

Apply the Tekton resource definitions from this directory to your OpenShift project.

# Apply the Tasks, Pipeline, and Trigger definitions 
``` oc apply -f . ```



## **3. Automatic Trigger Setup (GitHub Webhook)**

To have the pipeline run automatically on every git push, you must configure a webhook in your .NET application's GitHub repository.


### **a. Get the EventListener Route URL**

First, get the public URL for the EventListener that was created in the previous step.
```
oc get route el-dotnet-gitops-listener -o jsonpath='{.spec.host}' 
```


This will output a URL, for example: el-dotnet-gitops-listener-my-project.apps.my-cluster.com


### **b. Create the Webhook in GitHub**



1. Navigate to your .NET application's repository on GitHub.
2. Go to **Settings** > **Webhooks**.
3. Click **Add webhook**.
4. **Payload URL**: Paste the URL you retrieved from the oc get route command.
5. **Content type**: Change this to application/json.
6. **Secret**: You can leave this blank for now. For production, you would add a secret and configure a Tekton Interceptor to validate it.
7. **Which events would you like to trigger this webhook?**: Select **Just the push event**.
8. Ensure **Active** is checked.
9. Click **Add webhook**.

Now, every time you push a commit to this repository, it will automatically trigger a new pipeline run.