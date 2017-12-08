---
layout:     post
title:      "OpenShift 3: Binary Build and Deploy"
subtitle:   "How to deploy an java artifact into OpenShift with a binary build"
date:       2017-10-14 12:00:00
author:     "Kevin McAnoy"
header-img: "img/binary.jpg"
---

This tutorial will show you how to build and deploy an application to Openshift Container Platform (OCP) starting with a binary artifact - in this case a war file. The application is a simple Hello World but a few wrinkles have been added to make it a little more complicated. This application will require authentication and authorization and that will be done using LDAP and configured through JBoss EAP 7.0's included login module. We will assume a basic knowledge of OpenShift and you have installed the oc client. This will also require Maven to build the artifact.

The application will need to pass the LDAP values through OpenShift. Our example uses the free publicly available LDAP testing tool from   [Forum Systems](http://www.forumsys.com/tutorials/integration-how-to/ldap/online-ldap-test-server/).

In this tutorial we will define, build and deploy our application using the OpenShift's oc client.

The source code for the application that produces the war can be found at <https://github.com/mcanoy/ocp-examples.git>

#### Create a new project 
(optional. A project is needed.)

```
oc new-project binary-app
```

#### Create a new app in your project. 

This example will use JBoss EAP 7.0.
In the command below substitute the /path/to/empty/dir with an value that points to empty directory on your filesystem.  

```
oc new-app jboss-eap70-openshift~/path/to/empty/dir --name=binary-app
```

#### Declare all needed environmental variables

In the standalone-openshift.xml that will get configured during deployment of the application a few environmental variables have been defined:


```
oc set env dc/binary-app SAMPLE="test sample property value"
oc create secret generic ldap-password --from-literal=LDAP_CREDENTIAL=password
oc env --from=secret/ldap-password dc/hello-binary
oc set env dc/binary-app LDAP_URL=ldap://ldap.forumsys.com:389
oc set env dc/binary-app LDAP_BIND_DN=cn=read-only-admin,dc=example,dc=com
oc set env dc/binary-app LDAP_BIND_CTX_DN=dc=example,dc=com
oc set env dc/binary-app LDAP_ROLES_CTX_DN=dc=example,dc=com
```

#### Create A Route

Expose the application:

```
oc expose svc/my-crew-lcl --hostname=binary-app.192.168.64.2.nip.io/
```
Note: The hostname is derived from OpenShift settings. This example is run from MiniShift. The hostname should be resolvable by OpenShift. If you are experiencing problems with this step then omit the hostname parameter and OpenShift will define the URL for you. You can see the route by running:
```
oc get route
NAME        HOST/PORT                       PATH   SERVICES    PORT          WILDCARD
binary-app  binary-app.192.168.64.2.nip.io         pricelist   8080-tcp      None
```

#### Create the archive
This is the war file that will be deployed as a binary. From the directory where you cloned the git repository you can 
```
cd ocp-examples\ocp-bin-war
mvn clean package
```

#### Start the build

```
oc start-build helloworld-binary --from-dir=target/
```

You can view the progress of the build in the web console or use the oc cli

```
oc get pod
oc logs -f helloworld-binary-{buildNumber}
```

Once the build is complete the application should start a deployment. During this deployment a pod will be created and run. You can use the web console or cli to monitor the progress of the pod

```
oc get pod
oc logs -f {pod-name}
```

After you have verified that the application has started, navigate to the home page http://binary-app.192.168.64.2.nip.io/ [ This is may vary based on your hostname setting ]. It should be protected by basic authentication. Enter random credentials and verify that they are rejected. Try the credentials `tesla` and `password` and verify that you can view the home page and that the SAMPLE env variable is displayed.

You have now deployed a binary artifact in OpenShift that uses LDAP for security.

