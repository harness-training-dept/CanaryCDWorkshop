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

![source](/images/cvdemosource.png)
