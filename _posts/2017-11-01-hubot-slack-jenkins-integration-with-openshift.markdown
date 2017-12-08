---
layout:     post
title:      "Hubot Chatops: Using Slack to Interact with Jenkins"
subtitle:   "A simple example for using Slack to approve builds in Jenkins using Hubot on OpenShift. Ansible required."
date:       2017-12-01
author:     "Kevin McAnoy"
header-img: "img/hubot.png"
---

# Preamble

Jenkins pipelines have become more widespread recently. Now we can automated a deployment from code merge to production. Along the way we might build, test and deploy the application in mutliple different environments all in a single pipeline. Sometimes the pipeline requires a manual approval step to move the pipeline along some stages in the pipeline. Implementing this is trivial in Jenkins with a single line of groovy code. When this is implemented, an approver can log in to Jenkins and open the job and approve the activity. 

But that task can sometimes be tedious and Jenkins sessions expire after 30 minutes. In the Labs, as we are constantly working on creating pipelines with our customers, we ended looking at the Jenkins login screen many times a day. Sure, we could alter the timeout expiration to extend the session but there is another way and it is more fun and usually more transparent to those interested in that pipeline. That way is using Slack to proceed and abort pipelines and to integrate that we can use Hubot.

Hubot is a tool developed by Github to facilitate a lot of DevOps activities they needed each day. Hubot is an application that provides an integration point to many different applications. Here I will demonstate how to implement a simple Jenkins pipeline that will connect to a Hubot we deploy in OpenShift 3 and approve a pipeline step using Slack.

# Setting Up Slack

The first thing you will need is a Slack workspace that you are able to configure. You can create one at slack.com for free. 

Then find the Hubot app at `/apps/manage`. Search for Hubot in the search box. Follow the steps to create an integration point. For this you need:
 - The name of the bot. It can be anything. When invoking Hubot will we proceed the command with this name. I would consider making this an easy to type name
 - The integration token. This will allow Hubot to interact with Slack. It can be regenerated at anytime. 
 - A default channel to send messages. We will take advantage of this in the example but the bot will respond in any channels it is a member of.

 Invite your new point to the default channel.

 ![Slack Snapshot](/img/hubot-snapshot.png)

# Access OpenShift

This example deploys the application to OpenShift so you will need to have access to OpenShift. At https://developers.redhat.com you can a free localmachine version called MiniShift. Follow the install instructions there or use any available OpenShift instance. You need to have privileges to create projects for the code used here. You will also need the oc client that will interact with Ansible. If you are using MiniShift this client is provided with the install found somewhere like `~/.minishift/cache/oc/v3.6.173.0.5/oc`

login

```
oc login
```

# Install Ansible

This uses <img src="https://pbs.twimg.com/profile_images/428287509047435264/ElOjna20.png" width="50" align="left">  [Ansible](https://www.ansible.com/) and the OpenShift Applier. The example was run with Ansible 2.4. To verify you can run the command in your terminal

```
> ansible --version
ansible 2.4.0.0
  config file = None
  ...
```

# Set up Hubot and Jenkins

For this example, all the code you need for this example to run. The code is located at <https://github.com/mcanoy/ocp-examples.git>. You can clone this or fork and clone it and modify. Once it is cloned you have all the code you need. The Hubot code was developed by following the getting started guide at <https://hubot.github.com/docs/>. The code was then modified to allow Jenkins authentication by OCP token instead of user credentials. 

Change into the directory hubot-slack.

Modify the file `params/hubot/build-deploy` by adding your slack token to the parameter named HUBOT_SLACK_TOKEN

If necessary, modify the file `params/jenkins/deploy` by adding your slack channel name to the parameter named HUBOT_DEFAULT_ROOM

# Run the Applier

First import the ansible applier code (one time only)
 ```
ansible-galaxy install --roles-path . -r casl-ansible.yml
 ```

 Then run the playbook (as needed)
 ```
ansible-playbook --connection=local -i inventory casl-ansible/playbooks/openshift-cluster-seed.yml
 ```

The applier applies several templates that create applications in OpenShift in a new project named hubot. If something fails you can fix the templates or parameters and re-run the applier it will update only the changes you made. Navigate to the hubot project in OpenShift web console. 

![Console Snapshot](/img/hubot-ocp.png)

Observe the build and deploy process working. After a few minutes everything should be up and running and the first request for approval will appear automatically in your default chat room.

![Slack Snapshot](/img/hubot-slack.png)

# How does this work?

The inventory of applications resides in the file `inventory/group_vars/all.yml` and lists the template url and parameter file for each of Jenkins, Hubot and a sample application.

## Hubot

The hubot code and OpenShift template resides in another github.com repository <https://github.com/mcanoy/labs-ci-cd.git> which is a fork of the rht-labs repository. It provides a build config with a base image of `nodejs:6` and a deployment config for the hubot. It also provides a service, imagestream and route. It is an all in one template to get hubot running.

## Jenkins

Jenkins is split into 2 OpenShift templates. A build template and deploy template. The deploy template is customize to be able to reference hubot in the application. The code to build Jenkins resides in the <https://github.com/mcanoy/labs-ci-cd.git>. This code provides some add ons to the standard OpenShift Jenkins image - relevant for us is the inclusion of the hubot plugin.

## Sample App

To put it all together we need a sample Jenkins pipeline to invoke and make the request to hubot and slack. The code for that reside in the repository you cloned in the directory sample-app which contains a single Jenkinsfile. This pipeline should be invoke as soon Jenkins comes online. It is possible it may fail if the hubot application has not yet come online. If this is the case (or you simple want to invoke the pipeline) you can go to the OpenShift Console in the hubot project, select Builds --> Pipelines and start the sample-app pipeline to kick off a build. 

## Slack

When the pipeline is invoked you should see a message appear in the default channel specified for Slack. You can resond with a message to approve the step
```
bot-name jenkins proceed /job/sample-app/1/
```

Thanks for reading!



