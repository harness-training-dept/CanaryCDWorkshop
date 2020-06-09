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

Once it is in edit mode the easiest thing to do is delete everything that's currently in the file and replace it with the yaml quoted below. You can also reference this file [here:](https://github.com/harness-training-dept/CanaryCDWorkshop/blob/master/yamls/deployment.yaml). Just a note please be careful in the next few steps when editing, copying, and pasting YAMLs. They are sensitive to space and special characters (welcome to Kubernetes aka "death by YAML").

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

Once you've copied and pasted the above yaml into deployment.yaml in the Harness UI click on the Save button at the top to commit your changes.

8. Now we need to makes some changes to the namespace.yaml file. Click on namespace.yaml on the left hand side to select that file. Then hover your mouse over the file name. You should see 3 verticle dots. Click on those and select Rename. Change the name from namespace.yaml to ingress.yaml.

![rename](/images/rename.jpg)

9. Now that we've renamed the file, we need to update it with the following code. Using a similar procedure from step 7, you're going to first delete everything in the file then replace it with the following code. Note you can also reference this code [here.](https://github.com/harness-training-dept/CanaryCDWorkshop/blob/master/yamls/ingress.yaml)

```
#
# Ingress targeting canary only, at /canary/*
#
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: {{.Values.name}}-canary
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  rules:
  - http:
      paths:
      - path: /canary/(.*)
        backend:
          serviceName: {{.Values.name}}-canary
          servicePort: http-canary
---
#
# Ingress targeting stable only, at /api/*
#
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: {{.Values.name}}-stable
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  rules:
  - http:
      paths:
      - path: /api/(.*)
        backend:
          serviceName: {{.Values.name}}-stable
          servicePort: http-stable
```

Once you have updated the file click on Save. 

10. Now we need to follow the same procedure again but this time with service.yaml.  [Here's](https://github.com/harness-training-dept/CanaryCDWorkshop/blob/master/yamls/service.yaml) the code to replace what's already in the file:

```
#
# Service targeting canary only
#
apiVersion: v1
kind: Service
metadata:
  name: {{.Values.name}}-canary
spec:
  type: ClusterIP
  ports:
  - name: http-canary
    port: 8080
    targetPort: http
    protocol: TCP
  selector:
    app: {{.Values.name}}
    harness.io/track: canary
---
#
# Service targeting stable only
#
apiVersion: v1
kind: Service
metadata:
  name: {{.Values.name}}-stable
spec:
  type: ClusterIP
  ports:
  - name: http-stable
    port: 8080
    targetPort: http
    protocol: TCP
  selector:
    app: {{.Values.name}}
    harness.io/track: stable
```

Click save when you are done.

11. Ok we have one last file to update: the values.yaml file. Again following a similar procedure from the previous three steps, select and edit the values.yaml file. Make it look like [this:](https://github.com/harness-training-dept/CanaryCDWorkshop/blob/master/yamls/values.yaml)

```
name: cv-demo
namespace: ${infra.kubernetes.namespace}

image: ${artifact.metadata.image}

env:
  config:
    # Override in environment with ingress controller load balancer IP or host
    ALLOWED_ORIGINS: http://acfb419eb0a5b4beea6a0f1aca17ea99-100423743.us-east-1.elb.amazonaws.com:8080
```

Click save when you are done. Yes we know we have you just copying and pasting a bunch of stuff. If you have questions about these yamls and some more detailed specifics here, please stick around after class and ask us about them. We'll be happy to unpack them then.

12. Now that we've updated the service definition we can move on to setting up the Environment we're going to deploy to. Click on your application name in the popcorn trail on the upper left of the Harness UI to return to your Application main screen. Then click on Environments to setup an environment to deploy to. 

13. Click Add Environment button and give your environment a name that starts with your student ID. Set the environment type to non-production.

![Environment](/images/canary-evn.jpg)

When you click submit that will take you to the Environment Overview screen. 

14. Add an Infrastructure Definition. Give it a name that starts with your student ID. Select Kubernetes Cluster for your Cloud Provider Type, and set the deployment type to Kubernetes. Then you can select the Cloud Provider we have setup for this workshop. We've labeled the correct one "use-this-cluster-for-the-training." Be sure to change the Namespace setting to your student ID as well!

![infra_def](/images/envdefcan.jpg)

Now that we've setup a Service (what we're deploying) and an Environment (where we're deploying). Our next step is to build the canary deployment.

15. Using the popcorn trail switch to Workflows in your application. Click on the Add Workflow buttom. Give your new workflow a name that includes your student ID. Select Canary Deployment for your Workflow Type, and finally select the Environment you setup in the previous step.

![workflow](/images/workflowsetupcan.jpg)

Click submit when done. Now we are in our Workflow builder. Harness has setup an empty Canary template for us. Now let's fill it out.

16. First we're going to add a Deployment Phase. Under Deployment Phases click on + Add Phase

![add phase](/images/addphasecan.jpg)

In the Phase definition select the Service and Infrastructure Definition you setup earlier. 

![phase setup](/images/phasesetupcan.jpg)

Click submit when done. Now your Workflow screen should look like this:

![deployment screen](/images/deployscreencan.jpg)

17. Ok! Now we have the Deployment framework setup. First off we're going to configure the Canary Phase of the Workflow with a verification and rollback step. First up we're going to need to add a verification phase to our canary. (This is the step where we will consult metrics from Prometheus about our Canary. (TL;DR Prometheus is an open source metrics gathering agent and search engine for distributed systems.) To set this up we'll need to add a Verification step to our Canary Phase. In the Verify section of the Canary Phase click on Add Step. Your screen should look something like this:

![add step](/images/addsteppromcan.jpg)

If you don't see the Prometheus step listed just type "prom" in the search box and that should bring it up. Once it's there select it and click Next.

18. Next we are going to configure the verification by specifying what metrics we're interested in. First specify the Prometheus server we setup called Prometheus CV. Next we're going to specify our first metric to monitor. 

For the Metric Name specify "normal_call"

For the Metric Type pick "Throughput" in the dropdown

For Group Name specify "custom" 

And for Query specify this:
```io_harness_custom_metric_normal_call{kubernetes_pod_name="$hostName"}```

It should look like this:

![promsetup1](/images/promsetupcan.jpg)

Now we need to add one more Metric to Monitor. Click on the + Add button under Metrics to Monitor and add a second metric with the following values:

For the Metric Name specify "error_call"

For the Metric Type pick "Error" in the dropdown

For Group Name specify "custom" 

And for Query specify this:
```io_harness_custom_metric_error_call{kubernetes_pod_name="$hostName"}```

When you're done it should look like this:

![two metrics](/images/metricstomonitorcan.jpg)

19. Set your analysis time to 5 mins and your algorhytim to "very sensitive". 

Should look like this:

![analysis time](/images/analysistimecan.jpg)

Hit Submit when done. Your Canary Phase should now look like this: 

![verifyfinal](/images/verifyfinal.jpg)

20. Now we need to specify a Roll Back Setp to delete the Canary if the deployment fails. Under Rollback Steps select "Add Step" under "1. Deploy" Search for the "Delete" step by typing del into the search box. Select the Delete command and click Next.

![delsearch](/images/delsearchcan.jpg)

21. Give your Delete step a name and then specify ```${k8s.canaryWorkload}``` for Resources. That's a variable that specifies the Canary workload during the deployment.

![candel](/images/aurevoircan.jpg)

Click Submit when done. Now your Workflow should look like this:

![canwf](/images/wfmore.jpg)

22. Go back out of the Canary Phase of your Workflow and into the main part of your workflow. Your screen should look like this:

![nophase](/images/nophase.jpg)

23. Now we're going to add the main deployment phase that will happen after a successful canary. To do that click on + Add Phase under Deployment Phases. You will need to select the same Service and Infrastructure as you did for the Canary Phase. 

![phasephase](phasephase.jpg)

Click Submit when done. Now you have a Rolling Deployment phase all setup that runs after a sucessful Canary.

24. Go back out to the main part of your Workflow to add two Workflow Variables to our workflow. Click on edit icon next to Workflow Variables.

![wfvedit](/images/workflowvar1.jpg)

Then click on the + Add button and add the following two variables:

Variable Name: verify_canary

Default Value: yes

All else leave as is. 

Variable Name: metric_verification

Default Value: Prometheus

All else leave as it. 

Should look exactly like this before you hit Save. 

![wfvar](/images/wfvar2.jpg)

Now you're back out with a completed Canary Workflow! Well done!

![wfalldone](/images/wfalldone.jpg)

25. Now! we are ready to start deploying. But remember a Canary is designed to be compared to an already running microservice. So first we have to Deploy once to setup a baseline to compare our Canary to. To do this we'll run our Deployment but skip the verification stage. Scroll up to the top of the Workflow Overview screen and Click on the big blue Deploy button:

![deploybang](/images/deploybang.jpg)

26. That brings up our Start New Deployment screen. Change the verify_canary variable to no - this tells Harness to skip/automatically pass the Verify step. Leave metric_verification the default. Select the Tag# stable artifact.

![firstgood](/images/firstgood.jpg)

Hit submit when done. 

![alldonefirst](/images/alldone1st.jpg)

Now we have one good deployment to compare our canaries to! Take a moment to inspect the Prometheus step in your Canary Phase. You may need to scroll to left to get back over to it. 

![firstprom](/images/1stprom.jpg)

27. Take a 5 minute coffee / tea / beer / soda break to let our stable version build up some nice stable data in Prometheus. 

28. Ok welcome back. Now we can rerun our Deployment. But this time we're going to enable verification and we're going to deploy a less stable version. Go back into the Setup screen and select your application. Then go to your Workflow and hit Deploy. Leave both variables at their defaults but this time select the Tag# unstable version of the container. Hit Deploy when done. 

![badcanary](/images/badcanary.jpg)

29. While your canary deployment is running. Click on the Prometheus step to watch the verification process run. This can take a bit of time so feel free to freshen up the beverage from step 27. 

![promrunnin](/images/promrunnin.jpg)

30. Once the Prometheus step has completed and we realize that the artifact tagged unstable turned out to be unstable! The Canary gets marked failed and the Canary Phase Rollback step we created was run to delete the suspect Canary. 

![arcan](/images/arcan.jpg)


