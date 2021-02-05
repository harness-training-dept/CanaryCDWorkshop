# Kubernetes Continuous Delivery Canary Deployment Labs

## Lab 1 - Login to app.harness.io and familiarize yourself with the environment


1. Load up Chrome or Firefox web browser and login to https://app.harness.io with the username and password from your lab sheet. 

2. Click around and explore the GUI. Please note depending on the user role you have in your own organization's Harness implementation you may not have access to the menus and settings you see here in our training setup. 

5. We will be visiting the Setup menu most often. It's located on the lefthand side of the UI. Click in there and explore the different connectors and setting options. 

6. Ask your instructor if you don't understand the use of any configuration or dashboard.

## Lab 2 - Setup a Harness Application for our canary deployment

1. Go back to the main Setup page in the UI.

2. Click on the Add Application button and fill out the information for your new application. Be sure to use your student ID as the name of your application so you can find it easily later.

![Application Setup](/images/application.jpg)

3. Click submit and Harness will create your new application. Now we can setup the other parts of the deployment.

4. Click on Services to add our first service to this application, then click on the Add Service button. Give your service a name that includes your student ID (as you did with the application) and set the Deployment Type to Kubernetes.

![Add Service](/images/add_service_can.jpg)

Click submit. That will take you to the Service Overview.

5. In the Service Overview screen click on Add Artifact Source and select Docker Registry. For Source Server select Harness Docker Hub. This is a sample connection to the public hub.docker.com. For the Docker image name put harness/cv-demo . That is pointing to the Docker image we made for this lab.

![Artifact Source](/images/cvdemosource.png)

Click submit when done.

6. Now we need to hook our service up to a directory in Github which contains the yamls to install our demo application. To do this click on the three little dots in the upper right hand corner of the Manifests sections and select Link Remote Manifests

![remote manifests](/images/linkremote.jpg)

7. Provide Harness with the information to find the remote manifests on Gitbut. We are using normal K8s Resource YAMLs in a Go Template format - the same as we use in the Harness UI. We've already setup a Connector to the our Github account for you. Fill out the form like this:

Set the Manifest Format to ```Kubernetes Resource Specs in YAML format```

Set the Source Repository to ```admin_labs_git (https://github.com/harness-training-dept/admin_workloads)```

Set the Branch to ```master```

Set the File/Folder path to ```cvlab```

![edit deployment.yaml](/images/remotemanifests.jpg)

Click Submit when done.

8. Now that we've updated the service definition we can move on to setting up the Environment we're going to deploy to. Click on your application name in the popcorn trail on the upper left of the Harness UI to return to your Application main screen. Then click on Environments to setup an environment to deploy to. 

9. Click Add Environment button and give your environment a name that starts with your student ID. Set the environment type to non-production.

![Environment](/images/canary-evn.jpg)

When you click submit that will take you to the Environment Overview screen. 

10. Add an Infrastructure Definition. Give it a name that starts with your student ID. Select Kubernetes Cluster for your Cloud Provider Type, and set the deployment type to Kubernetes. Then you can select the Cloud Provider we have setup for this workshop. We've labeled the correct one "use-this-cluster-for-the-training." Be sure to change the Namespace setting to your student ID as well!

![infra_def](/images/envdefcan.jpg)

Now that we've setup a Service (what we're deploying) and an Environment (where we're deploying). Our next step is to build the canary deployment.

11. Using the popcorn trail switch to Workflows in your application. Click on the Add Workflow buttom. Give your new workflow a name that includes your student ID. Select Canary Deployment for your Workflow Type, and finally select the Environment you setup in the previous step.

![workflow](/images/workflowsetupcan.jpg)

Click submit when done. Now we are in our Workflow builder. Harness has setup an empty Canary template for us. Now let's fill it out.

12. First we're going to add a Deployment Phase. Under Deployment Phases click on + Add Phase

![add phase](/images/addphasecan.jpg)

In the Phase definition select the Service and Infrastructure Definition you setup earlier. 

![phase setup](/images/phasesetupcan.jpg)

Click submit when done. Now your Workflow screen should look like this:

![deployment screen](/images/deployscreencan.jpg)

13. Ok! Now we have the Deployment framework setup. First off we're going to configure the Canary Phase of the Workflow with a verification and rollback step. First up we're going to need to add a verification phase to our canary. (This is the step where we will consult metrics from Prometheus about our Canary. (TL;DR Prometheus is an open source metrics gathering agent and search engine for distributed systems.) To set this up we'll need to add a Verification step to our Canary Phase. In the Verify section of the Canary Phase click on Add Step. Your screen should look something like this:

![add step](/images/addsteppromcan.jpg)

If you don't see the Prometheus step listed just type "prom" in the search box and that should bring it up. Once it's there select it and click Next.

14. Next we are going to configure the verification by specifying what metrics we're interested in. First specify the Prometheus server we setup called Prometheus CV. Next we're going to specify our first metric to monitor. 

Now we need to add one more Metric to Monitor. Click on the + Add button under Metrics to Monitor and add a second metric with the following values:

For the Metric Name specify "error_call"

For the Metric Type pick "Error" in the dropdown

For Group Name specify "custom" 

And for Query specify this:
```io_harness_custom_metric_error_call{kubernetes_pod_name="$hostName"}```

Click on the +Add button to add an additional metric

For the 2nd Metric:

Metric Name specify "normal_call"

For the Metric Type pick "Throughput" in the dropdown

For Group Name specify "custom" 

And for Query specify this:
```io_harness_custom_metric_normal_call{kubernetes_pod_name="$hostName"}```

When you're done it should look like this:

![two metrics](/images/metricstomonitorcan.jpg)

15. Set your analysis time to 5 mins and your algorhytim to "very sensitive". 

Should look like this:

![analysis time](/images/analysistimecan.jpg)

Hit Submit when done. Your Canary Phase should now look like this: 

![verifyfinal](/images/verifyfinal.jpg)

16. Now we need to specify a Roll Back Setp to delete the Canary if the deployment fails. Under Rollback Steps select "Add Step" under "1. Deploy" Search for the "Delete" step by typing del into the search box. Select the Delete command and click Next.

![delsearch](/images/delsearchcan.jpg)

17. Give your Delete step a name and then specify ```${k8s.canaryWorkload}``` for Resources. That's a variable that specifies the Canary workload during the deployment.

![candel](/images/aurevoircan.jpg)

Click Submit when done. Now your Workflow should look like this:

![canwf](/images/wfmore.jpg)

18. Go back out of the Canary Phase of your Workflow and into the main part of your workflow. Your screen should look like this:

![nophase](/images/nophase.jpg)

19. Now we're going to add the main deployment phase that will happen after a successful canary. To do that click on + Add Phase under Deployment Phases. You will need to select the same Service and Infrastructure as you did for the Canary Phase. 

![phasephase](/images/phasephase.jpg)

Click Submit when done. Now you have a Rolling Deployment phase all setup that runs after a sucessful Canary.

20. Go back out to the main part of your Workflow to add two Workflow Variables to our workflow. Click on edit icon next to Workflow Variables.

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

21. Now! we are ready to start deploying. But remember a Canary is designed to be compared to an already running microservice. So first we have to Deploy once to setup a baseline to compare our Canary to. To do this we'll run our Deployment but skip the verification stage. Scroll up to the top of the Workflow Overview screen and Click on the big blue Deploy button:

![deploybang](/images/deploybang.jpg)

22. That brings up our Start New Deployment screen. Change the verify_canary variable to no - this tells Harness to skip/automatically pass the Verify step. Leave metric_verification the default. Select the Tag# stable artifact.

![firstgood](/images/firstgood.jpg)

Hit submit when done. 

![alldonefirst](/images/alldone1st.jpg)

Now we have one good deployment to compare our canaries to! Take a moment to inspect the Prometheus step in your Canary Phase. You may need to scroll to left to get back over to it. 

![firstprom](/images/1stprom.jpg)

23. Take a 5 minute coffee / tea / beer / soda break to let our stable version build up some nice stable data in Prometheus. 

24. Ok welcome back. Now we can rerun our Deployment. But this time we're going to enable verification and we're going to deploy a less stable version. Go back into the Setup screen and select your application. Then go to your Workflow and hit Deploy. Leave both variables at their defaults but this time select the Tag# unstable version of the container. Hit Deploy when done. 

![badcanary](/images/badcanary.jpg)

25. While your canary deployment is running. Click on the Prometheus step to watch the verification process run. This can take a bit of time so feel free to freshen up the beverage from step 27. 

![promrunnin](/images/promrunnin.jpg)

26. Once the Prometheus step has completed and we realize that the artifact tagged unstable turned out to be unstable! The Canary gets marked failed and the Canary Phase Rollback step we created was run to delete the suspect Canary. 

![arcan](/images/arcan.jpg)
