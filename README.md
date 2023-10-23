# edge_anomaly_detection_demo

## Prepare your own instance of the gitops repository

Fork [hhttps://github.com/nexus-Six/edge_anomaly_detection_demo.git](hhttps://github.com/nexus-Six/edge_anomaly_detection_demo.git) into your own Git repo (you will need to make changes in this repo).

Clone the forked repo to your local home directory 
```bash
cd ~

git clone <your_git_organisation>/edge_anomaly_detection_demo.git

```

Replace the git repo paths in
- config/instances/manuela-quickstart/manuela-quickstart-line-dashboard-application.yaml
-  config/instances/manuela-quickstart/manuela-quickstart-messaging-application.yaml
-  config/instances/manuela-quickstart/manuela-quickstart-machine-sensor-application.yaml

Replace the OCP paths in
- config/instances/manuela-quickstart/line-dashboard/line-dashboard-configmap-config.json
- config/instances/manuela-quickstart/line-dashboard/line-dashboard-route.yaml
- config/instances/manuela-quickstart/machine-sensor/machine-sensor-1-configmap.properties
- config/instances/manuela-quickstart/machine-sensor/machine-sensor-2-configmap.properties 
- config/instances/manuela-quickstart/messaging/route.yaml 


Push the changes

```bash
git add .

git commit -m "adapt repo URLs"

git push
```

## Deploy the OpenShift GitOps (ArgoCD) Operator

Installing the OpenShift GitOps Operator is a manual step in this quickstart guide.

Follow the OpenShift Documentation. I.e. [Installing OpenShift GitOps Operator in web console](https://docs.openshift.com/container-platform/4.10/cicd/gitops/installing-openshift-gitops.html#installing-gitops-operator-in-web-console_installing-openshift-gitops) 

Install with default settings


Wait until the OpenShift GitOps operator is installed. This might take a short while. 

After the Red Hat OpenShift GitOps Operator is installed, it automatically sets up a ready-to-use Argo CD instance that is available in the openshift-gitops namespace, and an Argo CD icon is displayed in the console toolbar. 


After the installation is complete, ensure that all the pods in the openshift-gitops namespace are running:
```
oc get pods -n openshift-gitops-operator

```
Example output:
```
NAME                                                            READY   STATUS    RESTARTS   AGE
openshift-gitops-operator-controller-manager-6849445bc8-gg49r   2/2     Running   0          21s

```


Note, in this quickstart deployment, we will use the cluster Argo CD instance and won't deploy any Manuela specific Argo CD instance.


Check that you can login into the Argo CD instance by using the Argo CD admin account:
- Follow the instructions in the documentation: [Logging in to the Argo CD instance by using the Argo CD admin account](https://docs.openshift.com/container-platform/4.10/cicd/gitops/installing-openshift-gitops.html#logging-in-to-the-argo-cd-instance-by-using-the-argo-cd-admin-account_installing-openshift-gitops)
- Or, pull the `admin` password with 
  
```
oc get secret openshift-gitops-cluster -o 'go-template={{index .data "admin.password"}}' -n openshift-gitops |  base64 -d
```

The admin credentials are needed because the manual sync runs into problems, when you login into ArgoCD with an OpenShift user.  


## Deploy the Applications via GitOps

We will deploy the three part of the core Manuele Applicatoion step by step:

1. Messaging component inclusive the AMQ Broker Operator and instance
2. Line Dashboard
3. Senor Simulators 

### Install the messaging component inclusive the AMQ Broker Operator and instance

Deploy the Messaging component:

```
oc apply -f config/instances/manuela-quickstart/manuela-quickstart-messaging-application.yaml
```
Example output:
```
application.argoproj.io/manuela-quickstart-messaging created
```

Check the status:
```
oc get application -n openshift-gitops
```
Example output:
```
NAME                                SYNC STATUS   HEALTH STATUS
manuela-quickstart-messaging        Synced        Progressing

```

Use the Argo CD UI to follow the installation.


### Install the line dashboard component 

Deploy the line dashboard component

```
oc apply -f config/instances/manuela-quickstart/manuela-quickstart-line-dashboard-application.yaml
```
Example output:
``` 
application.argoproj.io/manuela-quickstart-line-dashboard created

```

Check the status:
```
oc get application -n openshift-gitops
```
Example output:
```
NAME                                SYNC STATUS   HEALTH STATUS
manuela-quickstart-line-dashboard   Synced        Progressing
manuela-quickstart-messaging        Synced        Healthy
```

Wait until the application is `Healthy`.


Open the Manuela line dashboard web application and view the sensor data by selecting the "Realtime Data" menu item. Retrieve the UI URL via:
```bash
oc -n manuela-quickstart-line-dashboard get route line-dashboard
```

Note, no sensor data is displayed , because the senor simulators are not deployed yet.

### Install the Senor Simulators components 

Deploy the line machine sensor application:

```
oc apply -f config/instances/manuela-quickstart/manuela-quickstart-machine-sensor-application.yaml
```
Example output:
``` 
application.argoproj.io/manuela-quickstart-machine-sensor created
```

Wait until the application is `Healthy`.

```
oc get application -n openshift-gitops
```
Example output:
```
NAME                                SYNC STATUS   HEALTH STATUS
manuela-quickstart-line-dashboard   Synced        Healthy
manuela-quickstart-machine-sensor   Synced        Healthy
manuela-quickstart-messaging        Synced        Healthy
```

### Deploy the Anomaly Detection ML Service

- Install the OpenShift DataScience Operator from the Operatorhub with default settings

Add additional serving runtimes to OpenShift Datascience
```
oc apply -f config/rhods/custom-serving-runtimes.yaml
```

Install Minio S3 Storage

```
oc new-project minio
oc apply -f config/minio/minio.yaml

```

- Look for the minio-ui Route in the minio project and log in with
    - User : minio
    - Pass : minio123
- "Create a Bucket"
    - Bucket Name : ml-edge-demo
- Switch to the Object Browser
- Upload the initial ml model ml/mode.joblib  
 


Open the OpenShift Datascience Dasboard by clicking on the link provided at the top right menu under the square icon

- In the menu on the left click on Data Science Projects
- Create Data Science Project
    - Name : ml edge demo
    - Create
- Create Data Connection
    - Name : ml edge demo
    - Access key : minio
    - Secret key : minio123
    - Endpoint : http:minio-service.minio.svc:9000
    - Region : eu-central-1
    - Bucket : ml-edge-demo
    - Add dataconnection
- Click on Models and model > Add server
    - Model server name : mlserver
    - Serving runtime : mlserver
    - Check : 
Make deployed models available through an external route
    - Add



In the menu on the left click on Model Serving
- Click : Deploy Model
    - Project : ml edge demo
    - Model Name : ml edge demo model
    - Model servers : mlserver
    - Model Framework : skloearn - 0
    - Existing data connection : ml edge demo
    - Path : model.joblib
    - Deploy
- Wait for Model to deploy

Test the model on the command line :

```
curl -k -X POST -H 'Content-Type: application/json' -d '{"inputs": [{ "name": "predict", "shape": [1, 5], "datatype": "FP32", "data": [0.0, 0.0, 0.0, 0.0 , 12.0 ]}]}'  <your_model_serving endpoint>

```

Add a non https Route for your app

- In the OpenSHift Consol go to Project ml-edge-demo
- Create new Route
    - Name : plain
    - Service : modelmesh-serving
    - Target port : 8080
    - Create

Test the new Route as well, adding 'v2/models/test/infer' to the Route

Add the new endpoint to the Edge-Consumer via GitOPS

In the cloned Repository go to config/templates/manuela/messaging/messaging-configmap.properties

and set the Variable ANOMALY_DETECTION_URL with the full plain URL that you have just used for testing








