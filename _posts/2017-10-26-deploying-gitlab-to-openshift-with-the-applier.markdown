---
layout:     post
title:      "Deploying Gitlab to OpenShift with the OpenShift Applier"
date:       2017-10-26 12:00:00
author:     "Kevin McAnoy"
header-img: "img/post-bg-05.jpg"
---

Automating the configuraiton and deployment of Gitlab using the Omnibus container can be a bit of pain. This post will show you how to use the OpenShift Applier to automate this. The Openshift Applier uses Ansible to automate the deployment of OpenShift objects. The source code and documentation can be found [here](https://github.com/redhat-cop/casl-ansible/tree/master/roles/openshift-applier).

Ansible and the oc client are prerequisites for this example. This was executed with version 2.4.0.

Two OpenShift templates will be used. The first will configure and deploy Gitlab, Redis and Postgresql. It is an extension of the template that Gitlab [provides](https://gitlab.com/gitlab-org/omnibus-gitlab/blob/master/docker/openshift-template.json). The second template will provide the security permission needed since Gitlab needs to run as root (lame!). 


For this example, I will use [MiniShift](https://developers.redhat.com/products/cdk/overview/), referencing OpenShift 3.6. The example source code can be found [here](https://github.com/mcanoy/ocp-examples.git).

## Gitlab Applier Automation Example Explained

The OpenShift Applier will apply OpenShift objects to an existing OpenShift Cluster. See the `oc apply` function in the oc client to get an idea of how this works on the cli. The automation code will be installed through Ansible Galaxy. The example code will leverage this automation to set up Gitlab. 

Specifically, an inventory file is created  name `all.yml`

```
# all.yml
openshift_cluster_content:
- object: projectrequest
  content:
  - name: "gitlab-spaces"
    file: "{{ inventory_dir }}/../projectrequests/gitlab-spaces.yml"
    file_action: create
- object: deployments
  content:
  - name: "gitlab"
    namespace: "gitlab-proj"
    template: "https://raw.githubusercontent.com/rht-labs/labs-ci-cd/master/templates/gitlab/template-ldap.json"
    params: "{{ inventory_dir }}/../params/gitlab/deploy_params"
- object: policy
  content:
  - name: scc-gitlab-policy
    template: "https://raw.githubusercontent.com/rht-labs/labs-ci-cd/master/templates/scc/template-project-run-anyuid.json"
    params: "{{ inventory_dir }}/../params/policy/scc_params"
```

In this file, three OpenShift objects are created. The first is the project that will be created. All our configuration (besides the policy) will be placed inside this project. The second is the deployments object. Since all the containers have already been built for this example we only need to deploy them. Here we provide the gitlab openshift template, the project name (namespace) and a parameter file providing key/value pairs for configuration. The third object is the policy that needs to be assigned to the project and service account that will run the docker container. This is needed due to a limitation in the Gitlab docker container that requires it to run as root. The policy also applies parameters. 

In addition to the files referenced in all.yml, there are 2 other files. The `hosts` tells ansible to run locally and the `casl-ansible.yml` file will import the required ansible project. 

The example uses a publicly available LDAP server provided by [Forum Systems](http://www.forumsys.com/tutorials/integration-how-to/ldap/online-ldap-test-server/).

## Run the example

#### Verify the Parameters

Check the scc_params and deploy_params to verify that the information is accurate. By default, they should already be provided.

```
#scc_params
NAMESPACE=gitlab-repo
NAME=gitlab-user
PRIORITY_LEVEL=9
```

These parameters tell the policy template to allow the service account to run as root. The priority level is set so that it can be picked up ahead of other SCCs which are ranked ordered with tiebreakers going from least priviledged to most. Not setting the value will be rank ordered last. 

Note the APPLICATION_HOSTNAME in the deploy_params file. Ensure that the value that matches a URL resolvable by OpenShift. When using Minishift ensure that the IP address matches. 

```
#Gitlab deploy_params
APPLICATION_NAME=gitlab-app
APPLICATION_HOSTNAME=gitlab-app-gitlab-proj.192.168.64.2.nip.io
GITLAB_ROOT_PASSWORD=root_pw
POSTGRESQL_USER=pg_gitlab
POSTGRESQL_PASSWORD=gitlab_password
POSTGRESQL_ADMIN_PASSWORD=gitlab_admin_password
POSTGRESQL_DATABASE=gitlabhq
UNICORN_WORKERS=2
ETC_VOL_SIZE=100Mi
GITLAB_DATA_VOL_SIZE=5Gi
POSTGRESQL_VOL_SIZE=2Gi
REDIS_VOL_SIZE=512Mi
LDAP_LABEL=OpenShift
LDAP_HOST=ldap.forumsys.com
LDAP_PORT=389
LDAP_BIND_DN=cn=read-only-admin,dc=example,dc=com
LDAP_PASSWORD=password
LDAP_BASE=dc=example,dc=com
LDAP_USER_FILTER=
LDAP_ENCRYPTION=plain
```


#### Install the dependent ansible 

```
ansible-galaxy install --roles-path . -r casl-ansible.yml
```

#### Run the ansible

Make sure that the user that runs the ansible has privileges to create SCCs. You can verify for your account by running the oc command 
```
oc get scc
```

When using Minishift you can log in a cluster-admin
```
oc login -u system:admin
```

Run:

```
ansible-playbook --connection=local -i inventory casl-ansible/playbooks/openshift-cluster-seed.yml
```

#### Minishift: Grant Access to Web

(Optional) When using minishift by default two accounts are created: a developer account system:admin account. Since the system account cannot view the web console and the developer does not have access to see system:admin projects you will need to give the develop project access if you want to view the results in the web console. To do that you can run the command:

```
oc policy add-role-to-user admin developer
```

#### Verify

Ensure the expected results. In Openshift there should be 3 running pods for Gitlab, Postgresql and Redis. 

Go to the Gitlab URL and attempt to login. A valid user is `telsa / password`

You're done. Congratulations!

