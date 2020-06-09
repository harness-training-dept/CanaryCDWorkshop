# Kubernetes Continuous Delivery Canary Deployment Labs

## Lab 1 - Login to app.harness.io and familiarize yourself with the environment


1. Load up your Chrome web browser and login to https://app.harness.io with the username and password from your lab sheet. 

2. Click around and explore the GUI. Please note depending on the user role you have in your own organization's Harness implementation you may not have access to the menus and settings you see here in our training setup. 

5. We will be visiting the Setup menu most often. Click in there and explore the different connectors and setting options. 

6. Ask your instructor if you don't understand the use of any configuration or dashboard.

## Lab 2 - Setup a Harness Application for our canary deployment

1. Click on the Setup menu in the upper right hand corner of the Harness GUI.

2. Click on the Add Application button and fill out the information for your new application. Be sure to use your student ID in the name of your application so you can find it easily later.

![Application Setup](/images/application.jpg)

3. Click submit and Harness will create your new application. Now we can setup the other parts of the deployment.

4. Click on Services to add our first service to this application, then click on the Add Service button. Give your service a name that includes your student ID (as you did with the application) and set the Deployment Type to Kubernetes.

![Add Service](/images/add_service_can.jpg)

Click submit. That will take you to the Service Overview.

5. In the Service Overview screen click on Add Artifact Source and select Docker Registry. For Source Server select Harness Docker Hub. This is a sample connection to the public hub.docker.com domain setup automatically for harness.io. In non-training testing environments you would most likely delete this connector. For the Docker image name put harness/cv-demo . That is pointing to the Docker image we made for this lab.

![Artifact Source](/images/cvdemosource.png)

Click submit when done.

6. Now we need to modify the Kubernetes YAML files in the Harness deployment template to fit our canary. By default Harness initially sets up a Kubernetes deployment to be a rolling deployment we're going to change things around a bit to do a canary deployment.

7. First we're going to edit the deployment.yaml file. To do this scroll down to the yaml template editor. Select deployment.yaml on the left hand side, then click on the Edit button on the right.

![edit deployment.yaml](/images/edit_deploymentyaml.jpg)

Once it is in edit mode the easiest thing to do is delete everything that's currently in the file and replace it with this code:

```
apiVersion: v1
kind: Namespace
metadata:
  name: {{.Values.namespace}}
---

{{- if .Values.env.config}}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{.Values.name}}
data:
{{.Values.env.config | toYaml | indent 2}}
---
{{- end}}


apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{.Values.name}}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: {{.Values.name}}
  template:
    metadata:
      labels:
        app: {{.Values.name}}
      annotations:
        prometheus.io/scrape: 'true'
        prometheus.io/path: '/metrics'
    spec:
      containers:
      - name: {{.Values.name}}
        image: {{.Values.image}}
        imagePullPolicy: Always
        resources:
          requests:
            cpu: 500m
            memory: 50Mi
        readinessProbe:
          httpGet:
            path: /config
            port: http
          initialDelaySeconds: 5
          timeoutSeconds: 5
          periodSeconds: 5
          failureThreshold: 2
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
```
