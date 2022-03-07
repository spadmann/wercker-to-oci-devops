Migrating from Wercker to OCI DevOps!

We're here to help you move confidently through your migration journey by splitting the whole process into manageable steps and highlighting the tools, resources, and support you'll need at every step. This guide solely covers migration from Wercker to OCI DevOps.



Why should you migrate to OCI DevOps or other CI/CD tools?
On December 1st 2022, the Oracle-Wercker service will reach End of Life (EOL); this means after the EOL date, you will not be able to access Wercker, and your applications on Wercker will be deleted. Before the Wercker EOL date, you can export your applications from Wercker using the offline tool.

This migration guide will help you with the steps required to migrate your application from Wercker to OCI DevOps, OCI's native CI/CD platform. OCI DevOps is continuously evolving and has many of the features supported by Wercker, including serverless build runners and pipelines with different stages, including a build stage. Oracle Cloud Free Tier allows you to sign up for an Oracle Cloud account which provides a number of Always Free services and a Free Trial with US$300 of free credit to use on all eligible Oracle Cloud Infrastructure services for up to 30 days. The Free Trial services may be used until your US$300 of free credits are consumed or the 30 days has expired, whichever comes first.

Steps to migrate from Wercker
To migrate from Wercker, you need to follow the below steps.

Export your application's steps workflows from Wercker into your development machine
Create an account in OCI
Create the necessary resources in OCI DevOps
Adding your application to OCI DevOps
Exporting Wercker workflows
Steps, Workflows, and environment are the core files in an application on the Wercker service. You can export all these files from Wercker using the offline tool.

Wercker CLI: The Wercker command-line interface (CLI) is an open-source application that you can install on your development machine.

Offline Tool: A helper tool that downloads the workflows, environment variables, and steps of a Wercker application and generates the files necessary to run pipelines and workflows offline.

Follow the below instructions to export your application steps and workflows from Wercker.

Wercker export Prerequisites
Install Wercker CLI
Download Wercker CLI and follow the steps to complete the installation on your machine
Verify if the Wercker CLI is working fine.
Download the offline tool
Login to your Wercker app account and generate a personal token (WERCKER_TOKEN). Copy and save the token information in a file or somewhere secure to use later.
In your Wercker app account, click on the application you want to export.
Look for the URL in the browser. If your Wercker application URL is "http://app.wercker.com/owner-username/application-name", then copy 'owner-username/application-name' from the URL.
Once you complete the prerequisites, it's time to export your application workflows.

Running offline-tool
Navigate to your application's folder on your development machine. 
cd <application-directory>
Move the downloaded offline tool file to your application folder
Run the following commands below
chmod +x offline-tool
./offline-tool --application 'owner-username/application-name' --token $WERCKER_TOKEN
Once the export is successful, you will see the output like in the screenshot below.

                 

The Offline-tool will export and add the below files/folders to your application directory.

"steps" folder
"wercker-offline.YML" file
"envs" folder with the environment variables
                 

Now that you have all files related to Wercker, It's time to set up your pipelines in OCI DevOps.

OCI DevOps
What is OCI DevOps?
The Oracle Cloud Infrastructure (OCI) DevOps service is an end-to-end, Continuous Integration and Continuous Delivery (CI/CD) platform for developers.

Use this service to build, test, and deploy software and applications on Oracle Cloud. The DevOps build and deployment pipelines reduce change-driven errors and decrease customers' time building and deploying releases. The service also provides private Git repositories to store your code and supports connections to external code repositories.

With the DevOps service, you can do the following:

Create private code repositories to store and manage source code.
Mirror external Git repositories from GitHub and GitLab to OCI code repositories.
Build and test your latest changes in a build pipeline with a service-managed build runner.
Set up a trigger to automatically run your build pipeline from a source code commit or pull request. Optionally run a deployment pipeline on the successful build run for a complete CI/CD automation.
Orchestrate your software deployment across regions to OCI platforms such as Container Engine for Kubernetes (OKE), Compute instances, and Functions.
Avoid downtime during deployments and automate the complexity of updating applications.
Enhance security and reduce risk in delivery. Automation reduces the chance of human error that might introduce a security vulnerability. As DevOps enables faster software delivery, security bugs can be resolved quickly by rolling out a fix.
                    

Create an account in OCI
Let's start by creating an account on OCI

If you are an existing OCI customer, log in here.
New users can signup here. You can create a DevOps project as part of your Free Tier account.
Understand Build spec file
The build specification contains "build steps" and settings that the build pipeline uses to run a build.

The build_spec.yml is written in YAML, similar to Wercker.yml. 

The build specification is organized into the following sections.

Configuration of the build runner.
Setup of environment variables - You can define custom variables. 

Input Artifacts - Define a list of input artifacts required for running the current Build stage.
Steps to run in sequence - Defines a list of steps to be run. (Similar to steps in Wercker.yml)
Output Artifacts - Specifies the artifacts produced by the Build stage. The artifacts produced as output in this section are saved for the lifetime of the current build run. They can be used in the subsequent stages in the build pipeline.
Follow the instructions in the build spec documentation to write your first file.

Converting Wercker offline file to Build spec file format
In this step, let's learn how to convert your Wercker.yml to OCI build_spec.yml format. Create a file with "build_spec.yml" and place it in your application directory.



Wercker.yml	build_spec.yml


#Use OpenJDK base docker image from dockerhub and open the application port on the docker container
box:
  id: tomcat:9
  ports:
    - $PORT
workflows:
- name: build
  pipelines:
  - name: build
    pipelineName: build
    envFile: envs/build.env
#Build our application using Maven, just as we always have
build:
  box: openjdk:11
  steps:
    - install-packages:
        packages: maven
    - script:
        name: maven build
        code: |
          mvn clean
          mvn package

inject-secret:
  box:
    id: alpine
    cmd: /bin/sh
  steps:
  - kubectl "file://steps/wercker_kubectl_3.14.0":
      command: delete secret wercker; echo delete registry secret
      insecure-skip-tls-verify: "true"
      name: delete secret
      server: $KUBERNETES_MASTER
      token: $KUBERNETES_TOKEN
  - kubectl "file://steps/wercker_kubectl_3.14.0":
      command: create secret docker-registry wercker --docker-server=$DOCKER_REGISTRY
        --docker-username=$DOCKER_USERNAME --docker-password=$KUBERNETES_TOKEN --docker-email=$DOCKER_EMAIL;
        echo create registry secret
      insecure-skip-tls-verify: "true"
      name: create secret
      server: $KUBERNETES_MASTER
      token: $KUBERNETES_TOKEN

#Push the docker image with our built and tested application to the Oracle Container Registry
push-release:
  steps:
    - script:
        name: copy war to tomcat
        code: |
          cp /pipeline/source/target/*.war /usr/local/tomcat/webapps/
          mv /usr/local/tomcat/webapps/$DOCKER_NAME*.war /usr/local/tomcat/webapps/$DOCKER_NAME.war 
    - internal/docker-push:
        username: $DOCKER_USERNAME
        password: $OCI_AUTH_TOKEN
        repository: $DOCKER_REGISTRY/$DOCKER_REPO
        registry: https://$DOCKER_REGISTRY/v2
        tag: $WERCKER_GIT_BRANCH-$WERCKER_GIT_COMMIT
        working-dir: /pipeline/source
        ports: $PORT
        env: PORT=$PORT
        cmd: /usr/local/tomcat/bin/catalina.sh run

#Deploy our container from the Oracle Container Registry to the Oracle Container Engine (Kubernetes)
deploy-to-cluster:
    box:
        id: alpine
        cmd: /bin/sh

    steps:

    - bash-template

    - kubectl:
        name: delete secret
        server: $KUBERNETES_MASTER
        token: $KUBERNETES_AUTH_TOKEN
        insecure-skip-tls-verify: true
        command: delete secret wercker; echo delete registry secret

    - kubectl:
        name: create secret
        server: $KUBERNETES_MASTER
        token: $KUBERNETES_AUTH_TOKEN
        insecure-skip-tls-verify: true
        command: create secret docker-registry wercker --docker-server=$DOCKER_REGISTRY --docker-email=nobody@oracle.com --docker-username=$DOCKER_USERNAME --docker-password='$OCI_AUTH_TOKEN'; echo create registry secret

    - script:
        name: "Visualise Kubernetes config"
        code: cat kubernetes.yml

    - kubectl:
        name: deploy to kubernetes
        server: $KUBERNETES_MASTER
        token: $KUBERNETES_AUTH_TOKEN
        insecure-skip-tls-verify: true
        command: apply -f kubernetes.yml






version: 0.1
component: build
timeoutInSeconds: 6000
runAs: root
shell: bash
env:
  # these are local variables to the build config
  variables:
    key: "value"

  # the value of a vaultVariable is the secret-id (in OCI ID format) stored in the OCI Vault service
  # you can then access the value of that secret in your build_spec.yaml commands
  vaultVariables:
  #  EXAMPLE_SECRET: "YOUR-SECRET-OCID"
  
  # exportedVariables are made available to use as parameters in sucessor Build Pipeline stages
  # For this Build to run, the Build Pipeline needs to have a BUILDRUN_HASH parameter set
  exportedVariables:
    - DOCKER_USERNAME
    - USER_AUTH_TOKEN
    - BUILDRUN_HASH

    #  - DOCKER_REGISTRY_AUTH
    - TRIGGER_SOURCE_BRANCH_NAME
    - TRIGGER_COMMIT_HASH
    - DOCKER_REPO

steps:
  - type: Command
    name: "Define unique image tag"
    timeoutInSeconds: 40
    command: |
      export BUILDRUN_HASH=`echo ${OCI_BUILD_RUN_ID} | rev | cut -c 1-7`
      echo "BUILDRUN_HASH: " $BUILDRUN_HASH

  - type: Command
    timeoutInSeconds: 600
    name: "Install Node Prereqs"
    command: |
      cd ${OCI_WORKSPACE_DIR}/wercker_to_oci
      # install nvm
      curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash
      export NVM_DIR="$HOME/.nvm"
      [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm
      nvm install lts/erbium
      echo "NODE VERSION: $(node -v)"
      echo "NPM VERSION: $(npm -v)"

    onFailure:
      - type: Command
        command: |
          echo "Handling Failure"
          echo "Failure successfully handled"
        timeoutInSeconds: 40
        runAs: root
  
  - type: Command
    timeoutInSeconds: 600
    name: "NPM install"
    command: |
      cd ${OCI_WORKSPACE_DIR}/wercker_to_oci
      npm install
    onFailure:
      - type: Command
        command: |
          echo "Handling Failure"
          echo "Failure successfully handled"
        timeoutInSeconds: 40
        runAs: root
  
  - type: Command
    name: "Export Variables"
    timeoutInSeconds: 10
    command: |
      export DOCKER_USERNAME=${gDOCKER_USERNAME}
      export USER_AUTH_TOKEN=${gUSER_AUTH_TOKEN}
      #export DOCKER_REGISTRY_AUTH=`echo "${gDOCKER_USERNAME}:${gUSER_AUTH_TOKEN}" | base64|xargs`
      export TRIGGER_SOURCE_BRANCH_NAME=$OCI_TRIGGER_SOURCE_BRANCH_NAME
      export TRIGGER_COMMIT_HASH=$OCI_TRIGGER_COMMIT_HASH
      export DOCKER_REPO=${gDOCKER_REPO}
      echo "DockerRepo " $DOCKER_REPO "BUILDRUN_HASH: " $BUILDRUN_HASH

  - type: Command
    timeoutInSeconds: 1200
    name: "Run Tests"
    command: |
      cd ${OCI_WORKSPACE_DIR}/wercker_to_oci
      npm test

  - type: Command
    timeoutInSeconds: 1200
    name: "Build container image"
    command: |
      cd ${OCI_WORKSPACE_DIR}/wercker_to_oci
      docker build --pull --rm -t wercker_to_oci-getting-starter .

outputArtifacts:
  - name: output01
    type: DOCKER_IMAGE
    # this location tag doesn't effect the tag used to deliver the container image
    # to the Container Registry
    location: wercker_to_oci-getting-starter:latest








Congratulations, now you have the build spec file ready. You are now ready to create your first OCI DevOps project and migrate your Wercker application.

Let's start by completing the prerequisites before creating a DevOps project. 

OCI DevOps Prerequisites
Create a compartment
The compartment is an "Identity and Access Management (IAM)" mechanism in OCI DevOps, which helps you organize and isolate your cloud resources. Before you start creating any resource in OCI, you need to create a compartment first. The same goes for a DevOps project. 

               

Create a Dynamic group
Dynamic groups allow you to group OCI compute instances as "principal" actors. Create a "dynamic group" by following the steps mentioned in this link. While creating a dynamic group, add the below-mentioned rules. These rules give your dynamic group access to DevOps repositories, build and deploy pipelines, connections and read artifacts.

To create a Dynamic Group,

Open the navigation menu and click Identity & Security. Under Identity, click Dynamic Groups.

Click Create Dynamic Group.

In the rule field, enter the rules mentioned below. (Replace "compartment_OCID" and "compartment name" with actuals)

ALL {resource.type = 'devopsrepository', resource.compartment.id = '<compartment_OCID>'}
ALL {resource.type = 'devopsbuildpipeline', resource.compartment.id = '<compartment_OCID>'}
ALL {resource.type = 'devopsconnection', resource.compartment.id = '<compartment_OCID>'}
All {resource.type = 'devopsdeploypipeline', resource.compartment.id = '<compartmentOCID>'}
Allow dynamic-group <Dynamic Group name> to read all-artifacts in compartment <compartment_name>

    


Create a policy

A policy is a document that specifies who can access which Oracle Cloud Infrastructure resources your company has and how. A policy simply allows a group to work in certain ways with specific types of resources in a particular compartment.

To create a policy

Open the navigation menu and click Identity & Security. Under Identity, click Policies. A list of the policies in the compartment you're viewing is displayed.
Click Create Policy.
In the rule field, add the rule mentioned below.



Allow dynamic-group <Dynamic-groupname> to manage all-resources in compartment <compartment-name>


              

Create a Topic - A topic is a communication channel for sending messages to its subscriptions.
To create a topic:
Open the navigation menu and click Developer Services. Under Application Integration, click Notifications.
Choose a compartment you have access to

Click Topics.

Click on Create topic.

    

Create a vault, key, and secret - In OCI, the vault is a managed service that lets you centrally manage the encryption keys that protect your data and the secret credentials you use to access resources securely. You can save all your Git repository user-name and password in the vault as secret keys.
In OCI DevOps, you can either create a new repository in the DevOps service or mirror a repository hosted on GitHub or GitLab.
Before you create a new repository in the DevOps Service:
You must create an Auth token by following the steps mentioned in this document.
Once you create an Auth token, you can create a key by adding the auth token in the vault created above.
To mirror an external repo like GitHub or GitLab.
You must create a "Personal Access Token (PAT)" in your GitHub or GitLab. If you are new to this process, refer to the documentation here on how to do this.
Once you have created the PAT, you can create a key by adding the PAT to the above vault.
 
                        


                        


Congratulations on reaching another milestone. You are have now finished the prerequisites to create your first DevOps project.

OCI DevOps project
A DevOps project logically groups the DevOps resources needed to implement a CI/CD workflow. Now, let's create all the necessary resources to build and deploy your application using OCI DevOps. 

Create a DevOps project
 Create a DevOps project to build and deploy applications using the DevOps service successfully. 

To create a DevOps project -

In the Console, open the navigation menu, and then under Developer Services, click DevOps.
In the left-side menu, choose a compartment you have permission to work.
On the DevOps Projects - Overview page, click Create DevOps project.
                        

Create an External connection
You have already created a vault that stored your GitHub or GitLab account's personal access token (PAT) as keys and secrets in one of the above steps.

To mirror your code repository in an OCI DevOps project, you need to create an External connection first.

To create an external connection.

Open the navigation menu and click Developer Services. Under DevOps, click Projects.
Select a project and click External Connections on the left-side menu.
Click Create External Connection.
Fill in all other necessary fields and complete the create connection flow.
                       

Create a code repository
In the DevOps service, you can create your private code repository or connect to external code repositories such as GitHub and GitLab.

To create a repository.

Open the navigation menu and click Developer Services. Under DevOps, click Projects.
Select a project and click Code Repositories on the left-side menu.
Click Create Repository.
Once you create a repository, you can clone the repo in your development machine via SSH or HTTPS.
To connect an external repository, mirror the repository

Open the navigation menu and click Developer Services. Under DevOps, click Projects.
Select a project and click Code Repositories on the left-side menu.
Click Mirror Repository.
Fill out the necessary fields and complete mirroring your repo.
                      

Create a build pipeline
Managed Build stage
In your Build Pipeline, first, add a Managed Build stage

The Build Spec File Path is the relative location in your repo of the build_spec.yaml. Leave the default for this example
For the Primary Code Repository, choose the Code Repository you created above
The Name of your Primary Code Repository is used in the build_spec.yaml. In this example, you will need to use the name wercker_to_oci for the build_spec.yaml instructions to access this source code
Select the main branch
Create a Container Registry repository
Create a Container Registry repository for the wercker_to_oci-getting-started container image built in the Managed Build stage.

You can name the repo: wercker_to_oci-getting-started. So if you create the repository in the Ashburn region, the path is iad.ocir.io/TENANCY-NAMESPACE/wercker_to_oci-getting-started
Set the repository access to public so that you can pull the container image without authorization from OKE. Under "Actions", choose Change to public.
Create a DevOps Artifact for your container image repository
A parameter defines the container image version delivered to the OCI repository in the Artifact URI that matches a Build Spec exported variable or Build Pipeline parameter name.

Create a DevOps Artifact to point to the Container Registry repository location you just created above. Enter the information for the Artifact Location:

Name: wercker_to_oci-getting-started container
Type: Container image repository
Path: iad.ocir.io/TENANCY-NAMESPACE/wercker_to_oci-getting-started
Replace parameters: Yes
Next, you'll set the container image tag to use the Managed Build stage exportedVariables: name for the version of the container image to deliver in a run of a build pipeline. In the build_spec.yaml for this project, the variable name is: - DOCKER_USERNAME, USER_AUTH_TOKEN, BUILDRUN_HASH, TRIGGER_SOURCE_BRANCH_NAME, TRIGGER_COMMIT_HASH
- DOCKER_REPO

  exportedVariables:
    - DOCKER_USERNAME
    - USER_AUTH_TOKEN
    - BUILDRUN_HASH 
    - TRIGGER_SOURCE_BRANCH_NAME
    - TRIGGER_COMMIT_HASH
    - DOCKER_REPO
Edit the DevOps Artifact path, to add the tag value as a parameter name.

Path: iad.ocir.io/TENANCY-NAMESPACE/wercker_to_oci-getting-started:${BUILDRUN_HASH}
Add a Deliver Artifacts stage
Let's add a Deliver Artifacts stage to your Build Pipeline to deliver the wercker_to_oci-getting-started container to an OCI repository.

The Deliver Artifacts stage maps the output Artifacts from the Managed Build stage with the version to deliver to a DevOps Artifact resource and then to the OCI repository.

After the Managed Build stage, add a Deliver Artifacts stage to your Build Pipeline. To configure this stage:

In your Deliver Artifacts stage, choose Select Artifact
From the list of artifacts, select the wercker_to_oci-getting-started container artifact that you created above
In the next section, you'll assign the container image outputArtifact from the build_spec.yaml to the DevOps project artifact. For the "Build config/result Artifact name", enter: output01
Run your Build in OCI DevOps
From your Build Pipeline, choose Manual Run
Use the Manual Run button to start a Build Run

Manual Run will use the latest commit to your Primary Code Repository; if you want to specify a specific commit, you can optionally make that choice for the Primary Code Repository in the dropdown and selection below the Parameters section.

Connect your Code Repository to your Build Pipeline
To automatically start your Build Pipeline from a commit to your Code Repository, navigate your project and create a Trigger.

A Trigger is a resource to filter the events from your Code Repository, and a matching event will start the run of a Build Pipeline.

Push a commit to your DevOps Code Repository
Test out your Trigger by editing a file in this repo and pushing a change to your DevOps code repository.

Connect your Build Pipeline with a Deployment Pipeline
For CI + CD: continuous integration with a Build Pipeline and continuous deployment with a Deployment Pipeline, first create the Deployment Pipeline to deploy this example web application service to your OKE cluster. To review Deployment Pipelines, see the example Reference Architecture and docs. You'll also need to set up the policies to enable deployments.

Because the K8s manifest doesn't change each build, we're just going to create a single version of the K8s manifest by hand (or via API/CLI) in the Artifact Registry.

Create a DevOps Environment, Artifact Registry file, and DevOps Artifact
Create an Environment to point to your OKE cluster destination for this example. You will already need to create an OKE cluster or go through the Reference Architecture automated setup.

Create a new, or use an existing Artifact Registry repository

Upload the sample k8s resources manifest to your new repository

Name this artifact: web-app-manifest.yaml
Specify a version, i.e.: 1.0. You won't change this in the example.
From your upload choice (Console, Cloud Shell, or CLI), choose the included manifest in this repo: gettingstarted-manifest.yaml
Now create a DevOps Artifact to point to your Artifact Registry repository file

Select Type "Kubernetes manifest" so that you can use this artifact in your Deployment pipeline stage.
Select Artifact Registry repository as the Artifact source

Select the Artifact Registry repository that you just created
For the Artifact location, choose Select Existing Location and select the file and version: web-app-manifest.yaml:1.0 that you just uploaded above
Save!
Create your Deployment Pipeline
You've created the references to your OKE cluster and manifest to deploy; now, create your Deployment Pipeline.

Create a new Deployment Pipeline
Add your first stage - choose the type to release to OKE: Apply manifest to your Kubernetes cluster
Choose the environment that you created above
For Select Artifact, select the Kubernetes manifest DevOps artifact that points to web-app-manifest.yaml
Add
Add the pipeline parameters needed by the K8s manifest: ${namespace}
From the Parameters tab, add new values:
Name: namespace
Default value: whatever you want here: devops-sample-app
Description: namespace value needed by the k8s manifest
Smash that "+" button
To run this pipeline on its own, you can add a parameter for BUILDRUN_HASH or trigger it from the Build Pipeline, which will forward the build_spec.yaml exported variables to the Deployment Pipeline.

Add a Trigger Deployment stage to your Build Pipeline.
Once you've created your Deployment Pipeline, you can add a Trigger Deployment stage as the last step of your Build Pipeline.

After the latest version of the container image is delivered to the Container Registry via the Deliver Artifacts stage, we can start a deployment to an OKE cluster

Add a stage to your Build Pipeline
Choose a Trigger Deployment stage type
Choose Select Deployment Pipeline to choose the Deployment Pipeline that you created above.
From the Deployment Pipeline, you selected, you can confirm the parameters of that pipeline in the Deployment Pipeline details.

