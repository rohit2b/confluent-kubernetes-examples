Deploy Confluent Platform
=========================

In this workflow scenario, you'll set up Kafka with an internal listener using LDAP based authentication.

The goal for this scenario is for you to:

* Deploy LDAP server
* Deploy Kafka, with ZK and Control Center
* Configure a producer to generate sample data.


To complete this scenario, you'll follow these steps:

#. Set the current tutorial directory.

#. Deploy Confluent For Kubernetes.

#. Deploy LDAP server

#. Deploy Confluent Platform.

#. Deploy the Producer application.

#. Tear down Confluent Platform.

==================================
Set the current tutorial directory
==================================

Set the tutorial directory for this tutorial under the directory you downloaded
the tutorial files:

::
   
  export TUTORIAL_HOME=/Users/rbakhshi/dev/confluent/rohit-cfk-examples/security/plaintext-ldap-auth-control-Center

===============================
Deploy Confluent for Kubernetes
===============================

#. Set up the Helm Chart:

   ::

     helm repo add confluentinc https://packages.confluent.io/helm


#. Install Confluent For Kubernetes using Helm:

   ::

     helm upgrade --install operator confluentinc/confluent-for-kubernetes --namespace=confluent
  
#. Check that the Confluent For Kubernetes pod comes up and is running:

   ::
     
     kubectl get pods --namespace=confluent

===============
Deploy OpenLDAP
===============

This repo includes a Helm chart for `OpenLdap
<https://github.com/osixia/docker-openldap>`__. The chart ``values.yaml``
includes the set of principal definitions that Confluent Platform needs for
RBAC.

#. Deploy OpenLdap

   ::

     helm upgrade --install -f $TUTORIAL_HOME/../../assets/openldap/ldaps-rbac.yaml test-ldap $TUTORIAL_HOME/../../assets/openldap --namespace confluent

Note that it is assumed that your Kubernetes cluster has a ``confluent`` namespace available, otherwise you can create it by running ``kubectl create namespace confluent``. 

#. Validate that OpenLDAP is running:  
   
   ::

     kubectl get pods --namespace=confluent

#. Log in to the LDAP pod:

   ::

     kubectl --namespace=confluent exec -it ldap-0 -- bash

#. Run the LDAP search command:

   ::

     ldapsearch -LLL -x -H ldap://ldap.confluent.svc.cluster.local:389 -b 'dc=test,dc=com' -D "cn=mds,dc=test,dc=com" -w 'Developer!'

#. Exit out of the LDAP pod:

   ::
   
     exit 

=========================
Deploy Confluent Platform
=========================

Provide component TLS certificates
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::
   
    kubectl create secret generic tls-group1 \
      --from-file=fullchain.pem=$TUTORIAL_HOME/../../assets/certs/generated/server.pem \
      --from-file=cacerts.pem=$TUTORIAL_HOME/../../assets/certs/generated/cacerts.pem \
      --from-file=privkey.pem=$TUTORIAL_HOME/../../assets/certs/generated/server-key.pem \
      --namespace confluent

Provide authentication credentials
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

   ::
   
     kubectl create secret generic credential \
       --from-file=plain-users.json=$TUTORIAL_HOME/creds/creds-kafka-sasl-users.json \
       --from-file=kafka-server-plain-interbroker.txt=$TUTORIAL_HOME/creds/kafka-server-plain-interbroker.txt \
       --from-file=plain.txt=$TUTORIAL_HOME/creds/creds-client-kafka-sasl-user.txt \
       --from-file=ldap.txt=$TUTORIAL_HOME/creds/ldap.txt \
       --namespace confluent


#. Deploy Confluent Platform with the above configuration:

::

  kubectl apply -f $TUTORIAL_HOME/confluent-platform-ccc-ldap.yaml --namespace=confluent

#. Check that all Confluent Platform resources are deployed:

   ::
   
     kubectl get confluent --namespace=confluent

#. Get the status of any component. For example, to check Kafka:

   ::
   
     kubectl describe kafka --namespace=confluent

========
Validate
========

Deploy producer application
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Now that we've got the infrastructure set up, let's deploy the producer client
app.

The producer app is packaged and deployed as a pod on Kubernetes. The required
topic is defined as a KafkaTopic custom resource in
``$TUTORIAL_HOME/secure-producer-app-data.yaml``.

The ``$TUTORIAL_HOME/secure-producer-app-data.yaml`` defines the ``elastic-0``
topic as follows:

::

  apiVersion: platform.confluent.io/v1beta1
  kind: KafkaTopic
  metadata:
    name: elastic-0
    namespace: confluent
  spec:
    replicas: 1
    partitionCount: 1
    configs:
      cleanup.policy: "delete"
      
Deploy the producer app:

::
   
  kubectl apply -f $TUTORIAL_HOME/producer-app-data.yaml --namespace=confluent

Validate in Control Center
^^^^^^^^^^^^^^^^^^^^^^^^^^

Use Control Center to monitor the Confluent Platform, and see the created topic and data.

#. Set up port forwarding to Control Center web UI from local machine:

   ::

     kubectl port-forward controlcenter-0 9021:9021 --namespace=confluent

#. Browse to Control Center:

   ::
   
     http://localhost:9021


#. Users: 

    Full Control: Username:james Password:james-secret  

    Restricted Control: Username:alice Password:alice-secret

#. Check that the ``elastic-0`` topic was created and that messages are being produced to the topic.

=========
Tear Down
=========

Shut down Confluent Platform and the data:

::

  kubectl delete -f $TUTORIAL_HOME/producer-app-data.yaml --namespace=confluent

::

  kubectl delete -f $TUTORIAL_HOME/confluent-platform-ccc-ldap.yaml --namespace=confluent

::

  helm delete operator --namespace=confluent

::

  helm delete test-ldap --namespace=confluent

::

  kubectl delete pvc ldap-config-ldap-0 --namespace=confluent

::

  kubectl delete pvc ldap-data-ldap-0 --namespace=confluent

::
