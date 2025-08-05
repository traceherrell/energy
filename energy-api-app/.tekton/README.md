# **Tekton Pipeline for the .NET Energy API**

This directory contains all the necessary Tekton resources to build the .NET application using a Dockerfile, push the resulting image to Quay.io, and update the deployment manifest in the corresponding deployment repository.

## **Directory Structure**

.  
├── README.md               \<-- This file  
├── pipeline.yaml           \<-- The main pipeline definition using buildah  
├── run  
│   └── pipelinerun.yaml    \<-- An example of how to run the pipeline  
├── setup  
│   ├── 01-pvc.yaml         \<-- A PersistentVolumeClaim for the pipeline workspace  
│   ├── 02-secrets.yaml     \<-- A reference for the required secrets  
│   └── 03-service-account.yaml \<-- ServiceAccount for the pipeline to use  
└── tasks  
    └── git-update-deployment.yaml \<-- Custom task to update the deployment YAML

## **Prerequisites**

1. **OpenShift Pipelines Operator:** Ensure the Tekton operator is installed on your OpenShift cluster.  
2. **tkn CLI (Optional):** The Tekton CLI is useful for interacting with pipelines, but you can also use oc or kubectl.  
3. **Quay.io Robot Account:** You need a robot account token for your Quay.io repository (quay.io/therrell/energy-app-api).  
4. **GitHub Personal Access Token (PAT):** You need a PAT with repo scope to allow the pipeline to push changes to your energy-deploy repository.

## **Setup Instructions**

Follow these steps to configure and run the pipeline.

### **1\. Create the Namespace**

Create a dedicated namespace (or project in OpenShift) for your CI/CD resources.

oc new-project energy-api-ci

### **2\. Create the Secrets**

The pipeline requires two secrets: one for pushing to Quay.io and one for pushing to GitHub.

#### **Quay.io Secret (Using oc)**

This is the recommended method as it creates the secret in the exact format buildah requires. Run the following command, replacing the placeholder with your robot account's token.

oc create secret docker-registry quay-secret \\  
  \--docker-server=quay.io \\  
  \--docker-username='therrell+robot' \\  
  \--docker-password='YOUR\_QUAY\_TOKEN\_HERE' \\  
  \--docker-email='any-email@example.com' \\  
  \-n energy-api-ci

#### **GitHub Secret**

1. **Get your Base64 encoded credentials:**  
   \# Use your GitHub username and your Personal Access Token (PAT)  
   echo \-n 'your-github-username:YOUR\_GITHUB\_PAT' | base64

2. **Create a github-secret.yaml file:**  
   apiVersion: v1  
   kind: Secret  
   metadata:  
     name: git-deploy-credentials  
     annotations:  
       tekton.dev/git-0: \[https://github.com\](https://github.com)  
   type: kubernetes.io/basic-auth  
   data:  
     username: PASTE\_ENCODED\_USERNAME\_HERE  
     password: PASTE\_ENCODED\_PAT\_HERE

3. **Apply the secret:**  
   oc apply \-f github-secret.yaml \-n energy-api-ci

### **3\. Apply Setup, Task, and Pipeline Files**

Apply the rest of the configuration files in the correct order.

\# Apply the Persistent Volume Claim for the workspace  
oc apply \-f setup/01-pvc.yaml \-n energy-api-ci

\# Apply the Service Account that links the secrets  
oc apply \-f setup/03-service-account.yaml \-n energy-api-ci

\# Apply the custom task definition  
oc apply \-f tasks/git-update-deployment.yaml \-n energy-api-ci

\# Apply the main pipeline definition  
oc apply \-f pipeline.yaml \-n energy-api-ci

### **4\. Run the Pipeline**

The pipeline is triggered by creating a PipelineRun resource. An example is provided in run/pipelinerun.yaml.

oc apply \-f run/pipelinerun.yaml \-n energy-api-ci

### **5\. Monitor the PipelineRun**

You can watch the pipeline execute using the tkn CLI:

\# Get the name of the latest pipelinerun  
PIPELINERUN\_NAME=$(tkn pipelinerun list \-n energy-api-ci \-o jsonpath='{.items\[0\].metadata.name}')

\# Follow the logs  
tkn pipelinerun logs $PIPELINERUN\_NAME \-f \-n energy-api-ci

You can also view the pipeline's progress visually in the OpenShift Developer Console under the **Pipelines** section.