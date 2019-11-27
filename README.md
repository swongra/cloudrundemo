# Cloud Run Demo
Cloud Run Open Office demo

This is a tuttorial for deploying Open Office via two flavors of Cloud Run: fully managed Cloud Run and Cloud Run for Anthos. Then you will subsequently deploy the same container image on IBM Cloud, running a Kubernetes Cluster with Knative enabled. 

Watch this [video](https://www.youtube.com/watch?v=nhwYc4StHIc) first, which outlines the contents of this demo.

## Set up
1. Authorize the gcloud command-line tool to access your project:

```gcloud auth login```

2. Configure your project for the gcloud tool, where [PROJECT_ID] is your GCP project ID:

```gcloud config set project [PROJECT_ID]```

### Preparing source files
You'll need some sample source code to build. Here you’ll pull the source code and Dockerfile from a Github repository.
1. Git clone this repo.
You can see you get source code - to-pdf.py and a Dockerfile

2. Navigate to pdf/

3. Run the following command to make quickstart.sh executable:

```chmod +x to-pdf.py```

4. Build using Dockerfile

[Cloud Build](https://cloud.google.com/cloud-build/) allows you to build a Docker image using a Dockerfile. Cloud Build is a service that executes your builds on Google Cloud Platform infrastructure. Cloud Build can import source code from Google Cloud Storage, Cloud Source Repositories, GitHub, or Bitbucket, execute a build to your specifications, and produce artifacts like Docker containers or Java archives. You don't need a separate build config file. 

### Create a container image and push to Container Registry

1. First, enable the Container Registry, Cloud Build, and Cloud Run APIs.

```gcloud services enable containerregistry.googleapis.com cloudbuild.googleapis.com run.googleapis.com``` 

2. Update installed gcloud components:

```gcloud components update```

3. Install the gcloud beta components:

```gcloud components install beta```

4. Install the kubectl command-line tool:

```gcloud components update```

5. Run the following command from the directory containing to-pdf.py and Dockerfile, where [PROJECT_ID] is your GCP project ID:

```gcloud builds submit --tag gcr.io/[PROJECT_ID]/pdf-service .```

> Note: Don't miss the "." at the end of the above command. The "." specifies that the source code is in the current working directory at build time.

6. Double Check Container Registry console for container image title pdf-service.

### Deploy container image on Cloud Run
Next, we’re going to deploy our container image on our first flavor: fully managed Cloud Run. 

1. Deploy container image pdf-service on Cloud Run. 

```gcloud beta run deploy --image gcr.io/[PROJECT ID]/pdf-service```

2. Select option 1 to deploy on fully managed Cloud Run. 

3. Select a region.

4. Type in the service name (pdf-service).

5. When it asks you if you want to allow unauthenticated invocations, select yes. 

6. Click on the URL or go to the Cloud Run console and click the new deployment name to find the URL. Voila! Your serverless container image is running on a stable and secure HTTPS endpoint! You can now upload a test Word document and convert it to a pdf. 

### Deploy Container image on Cloud Run for Anthos
Now let’s deploy the same container image on Cloud Run for Anthos deployed on GKE. 

First we need a GKE cluster with these minimum settings:
* Cloud Run on GKE enabled
* Kubernetes version: see recommended versions
* Nodes with 4 vCPU
* Scopes to access cloud-platform, write to logging, write to monitoring

1. First, enable the Kubernetes Engine API:

```gcloud services enable container.googleapis.com```

2. Set the desired zone for the new cluster. You can use any zone where GKE is supported:

```gcloud config set compute/zone ZONE```

3. Create a GKE cluster with the following command: 

```
gcloud beta container clusters create run-cluster1 \
--addons=HorizontalPodAutoscaling,HttpLoadBalancing,Istio,CloudRun \
--machine-type=n1-standard-4 \
--cluster-version=latest --zone=us-central1-a \
--enable-stackdriver-kubernetes \
--scopes cloud-platform 
``` 

>Important: Running a GKE configuration like the one described in this page can be costly. GKE is billed differently than Cloud Run, so you will be billed for each node in your cluster, even if you have no services deployed to them. To avoid charges, you should delete your cluster or scale the number of the nodes in the cluster to zero if you are not using it.

>Services deployed on Cloud Run on GKE aren’t given a domain name like fully managed Cloud Run, so we need to map the GKE cluster’s ingress gateway external IP address to a custom domain, or a free DNS wildcard site like xip.io. 

4. Export the IP of the ingress gateway of your cluster:

```export GATEWAY_IP="$(kubectl get svc istio-ingressgateway \
   --namespace istio-system \
   --output 'jsonpath={.status.loadBalancer.ingress[0].ip}')"
   ```

5. Copy the external IP:

```echo $GATEWAY_IP```

6. Then map the external IP to the free DNS wildcard site. Replace the [EXTERNAL_IP] with the IP address you just copied. 

```kubectl patch configmap config-domain --namespace knative-serving --patch \
  '{"data": {"example.com": null, "[EXTERNAL-IP].xip.io": ""}}'
  ```
7. Now deploy the image on Cloud Run on your GKE cluster. Replace [PROJECT ID] with your project ID. 

```gcloud beta run deploy --image gcr.io/[PROJECT ID]/pdf-service --cluster run-cluster1```

8. Select option 2 to deploy the image on Cloud Run on GKE when prompted, then select the region. Type the service name when prompted (pdf-service). 

9. Click on the URL or go to the Cloud Run console and click the new deployment name to find the URL. Bam! That exact same container image is running on a stable endpoint now on a GKE cluster, and it was deployed as serverless code! 

We now have our two versions of Cloud Run deployments live!

### Port to IBM Cloud using Knative
Using Knative you can easily run the same container image as serverless code and deploy it to another Kubernetes Cluster you had running in IBM Cloud. Here are the general steps:

1. Create a Kubernetes Cluster in another cloud provider like IBM Cloud. There you can enable the Knative add-on. 

2. Now head back to your Google Cloud console. In your GCP Container Registry, enable the pdf-service container image to be publicly available. 

3. In the Google Cloud Shell home directory, create a yaml file named service.yaml with the following content:

```
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
 name: pdf
spec:
 runLatest:
   configuration:
     revisionTemplate:
       spec:
         container:
           image: [CONTAINER IMAGE URL]
```
> Note: Replace [CONTAINER IMAGE URL] with your public URL of your container image. Ex: gcr.io/vpc-demo-225319/pdf-service
In the Cloud Shell, download the IBM SDK and login using [these steps](https://cloud.ibm.com/docs/containers?topic=containers-cs_cli_install). 

4. Run the following to point to your running cluster in IBM Cloud:

```ibmcloud ks cluster-config --cluster [CLUSTER ID]```

You might get a warning that the command is using deprecated behavior but it should give you a confirmation that you downloaded the kubeconfig file successfully. 

5. Now run the export command given to you. 

6. Confirm you’ve set kubeconfig to your IBM cluster by running:

```Kubectl get nodes```

You should see the IP addresses of the nodes running in IBM.

7. Run the following to apply service.yaml to the IBM cluster:

```kubectl apply -f service.yaml```

8. Enter the following to view the container image URL endpoint.   

```Kubectl get ksvc```

9. Copy the Domain in your browser and you should see the service is now running on the IBM Kubernetes cluster!


# Congratulations! 
You’ve just deployed a serverless container running a legacy app on fully hosted Cloud Run, Cloud Run for Anthos, and on IBM Cloud using Knative! Cloud Run coupled with Knative gives you the flexibility to run serverless, the agility of containers, and portability across environments. Other advantages include fast scale up, fast scale down, saving you from cost when the service is not needed, and the ability to use any binary or language because of the versatility of containers. The biggest advantage in my opinion: you have a consistent experience wherever you prefer, full managed, for Anthos/GKE, or your own Kubernetes cluster. It’s another foot forward in the industry effort to give precious time back to coding instead of infrastructure management, even as environments modernize and decouple. If that doesn’t get you up in the morning, I don’t know what will!

## What to do next?
* Learn more about [Cloud Run](https://cloud.google.com/run/)! 

* Subscribe to the [GCP Youtube channel](https://www.youtube.com/user/googlecloudplatform) where you’ll find a lot more on serverless and how to deploy this demo. 

* Follow me on [Twitter](https://twitter.com/swongful) to stay up to date on the latest on GCP.

* And check out the [Google's Cloud events](https://cloud.google.com/events/) near you. We’ve even hosted Cloud Run clubs where you actually get to run a 5k, learn about Cloud Run, do hands-on labs, and meet Googlers and other community members!
