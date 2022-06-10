Deploy Confluent Platform
=========================

In this workflow scenario, you'll set up a simple non-secure (no authn, authz or
encryption) Confluent Platform, consisting of all components.

You will also configure additional custom listeners.  

The goal for this scenario is for you to:

* Quickly set up the complete Confluent Platform on the Kubernetes.
* Add additional broker listeners.  
* Configure a producer to generate sample data.


Before continuing with the scenario, ensure that you have set up the
`prerequisites </README.md#prerequisites>`_.

To complete this scenario, you'll follow these steps:

#. Set the current tutorial directory.

#. Deploy Confluent For Kubernetes.

#. Deploy Confluent Platform.

#. Deploy the Producer application.

#. Tear down Confluent Platform.

==================================
Set the current tutorial directory
==================================

Set the tutorial directory for this tutorial under the directory you downloaded
the tutorial files:

::
   
  export TUTORIAL_HOME=<Tutorial directory>/kafka-additional-listeners

===============================
Deploy Confluent for Kubernetes
===============================

#. Set up the Helm Chart:

   ::

     helm repo add confluentinc https://packages.confluent.io/helm


#. Install Confluent For Kubernetes using Helm:

   ::

     helm upgrade --install operator confluentinc/confluent-for-kubernetes
  
#. Check that the Confluent For Kubernetes pod comes up and is running:

   ::
     
     kubectl get pods

========================================
Review Confluent Platform configurations
========================================

You install Confluent Platform components as custom resources (CRs). 

You can configure all Confluent Platform components as custom resources. In this
tutorial, you will configure all components in a single file and deploy all
components with one ``kubectl apply`` command.

The entire Confluent Platform is configured in one configuration file:
``$TUTORIAL_HOME/confluent-platform.yaml``

In this configuration file, there is a custom Resource configuration spec for
each Confluent Platform component - replicas, image to use, resource
allocations.

  
=========================
Deploy Confluent Platform
=========================

#. Deploy Confluent Platform with the above configuration:

   ::

     kubectl apply -f $TUTORIAL_HOME/confluent-platform.yaml
     
#. Check that all Confluent Platform resources are deployed:

   ::
   
     kubectl get confluent

#. Get the status of any component. For example, to check Kafka:

   ::
   
     kubectl describe kafka

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
    replicas: 3 # change to 1 if using single node
    partitionCount: 1
    configs:
      cleanup.policy: "delete"
      
Deploy the producer app:

``kubectl apply -f $TUTORIAL_HOME/producer-app-data.yaml``

Validate in Control Center
^^^^^^^^^^^^^^^^^^^^^^^^^^

Use Control Center to monitor the Confluent Platform, and see the created topic and data.

#. Set up port forwarding to Control Center web UI from local machine:

   ::

     kubectl port-forward controlcenter-0 9021:9021

#. Browse to Control Center:

   ::
   
     http://localhost:9021

#. Check that the ``elastic-0`` topic was created and that messages are being produced to the topic.

Review the additional listeners
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Review Kafka ConfigMap to check the additional custom listeners

::

   kubectl get configmap kafka-shared-config -o jsonpath="{.data.kafka\.properties}" | grep -i listener 

You should see the following, indicating the new listeners 

::

  inter.broker.listener.name=REPLICATION
  listener.security.protocol.map=CUSTOMLISTENER1:PLAINTEXT,CUSTOMLISTENER2:PLAINTEXT,EXTERNAL:PLAINTEXT,INTERNAL:PLAINTEXT,REPLICATION:PLAINTEXT
  listeners=CUSTOMLISTENER1://:9204,CUSTOMLISTENER2://:9205,EXTERNAL://:9092,INTERNAL://:9071,REPLICATION://:9072


=========
Tear Down
=========

Shut down Confluent Platform and the data:

::

  kubectl delete -f $TUTORIAL_HOME/producer-app-data.yaml

::

  kubectl delete -f $TUTORIAL_HOME/confluent-platform.yaml

::

  helm delete operator
  
