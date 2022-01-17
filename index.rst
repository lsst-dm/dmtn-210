:tocdepth: 3

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

Overview
========

Rubin's Alert Distribution System is implemented in the integration environment at the interim data facility (the "IDF").
The implementation runs on the shared Rubin Science Platform Kubernetes cluster in the IDF.

This document aims to be a point-in-time record of what exists, and to explain implementation decisions made during construction.
An overview is provided of the system's concepts and components, and then each is described in detail.

At the highest conceptual level, it is composed of an Apache Kafka :cite:`kafka` cluster, a Confluent Schema Registry :cite:`confluent-schema-registry`, software to generate simulated alerts, and an alert database implemenntation following the design laid out in DMTN-183 :cite:`DMTN-183`.

Terminology and Concepts
========================

In order to explain the components that make up the Alert Distribution System, it's helpful to first establish some basic concepts behind deployments to Kubernetes in general, and to Rubin's Science Platform Kubernetes cluster in particular.

Resources
---------

Kubernetes is built on *resources*.
These are abstract descriptions of persistent entities that should be configured and run in a particular Kubernetes cluster.
Resources are defined in YAML files which are submitted to the Kubernetes cluster.

For example, Kubernetes uses resources to define what `services <https://kubernetes.io/docs/concepts/services-networking/service/>`__ should be running, how `traffic <https://kubernetes.io/docs/concepts/services-networking/network-policies/>`__ should be routed, and how `persistent storage <https://kubernetes.io/docs/reference/kubernetes-api/config-and-storage-resources/persistent-volume-v1/>`__ should be provisioned.

Kubernetes users are able to provide custom resource types; these are used in the Alert Distribution System to describe the desired Kafka configuration.

In addition, resources are configured to reside within a particular namespace.
These namespaces act as boundaries for authorization, as well as providing naming uniqueness for resources.
Somewhat unconventionally, all of the alert stream resources are in one namespace, "``alert-stream-broker``", which is explained in :ref:`single-namespace`.

Operators
---------

Kubernetes *operators* are programs that run within the Kubernetes cluster, and take actions when resources are created, modified, or deleted.
There are many default operators, and others that are installed to the cluster explicitly.
The Alert Distribution System uses two such custom operators: Strimzi :cite:`strimzi` and Strimzi Registry Operator :cite:`strimzi-registry-operator`.

Helm Charts
-----------

Helm :cite:`helm` is a project which provides tools for templating the YAML resource definitions used in Kubernetes.
The templates are called _Charts_, and provide a flexible way to represent common or repeated configuration.

For the Rubin IDF, all charts are defined in the `lsst-sqre/charts`_ repository.

.. _lsst-sqre/charts: https://github.com/lsst-sqre/charts

Charts are reified into infrastructure using a set of *values* which populate the templates.
For the Rubin IDF, the values to be used are defined in the `lsst-sqre/phalanx`_ repository.

.. _lsst-sqre/phalanx: https://github.com/lsst-sqre/phalanx

These two repositories are part of the Phalanx system, described in SQR-056 :cite:`SQR-056` and at `phalanx.lsst.io <https://phalanx.lsst.io/>`__.

Argo
----

While Helm can be run as a tool on the command line, the Science Platform convention is to run it through Argo CD :cite:`argo-cd`.
Argo CD is a platform for coordinating changes to a Kubernetes cluster, and it is able to run Helm directly.
It is configured through a set of conventions in the `lsst-sqre/phalanx`_ repository.

Terraform
---------

Terraform :cite:`terraform` is a tool for provisioning infrastructure through code.
On the Rubin Science Platform, Terraform is used to provision non-Kubernetes resources, such as Google Cloud Storage buckets and account permissions for applications running inside Kubernetes to access Google Cloud Platform APIs.

Terraform source code resides in the `lsst/idf_deploy`_ repository.

.. _lsst/idf_deploy: https://github.com/lsst/idf_deploy

Principal Components
====================

The Alert Distribution System has six principal components:

1. The **Strimzi Operator** is responsible for managing a Kafka Cluster. It configures broker nodes, topics, and Kafka user identities.
2. The **Kafka Cluster** is an instance of Apache Kafka with several broker nodes and Zookeeper metadata nodes. It holds the actual alert packet data.
3. The **Strimzi Registry Operator** is responsible for managing a Confluent Schema Registry instance, correctly connecting it to the Kafka Cluster.
4. The **Schema Registry** is an instance of the Confluent Schema Registry, along with an ingress configured to allow read-only access from the internet by anonymous users.
5. The **Alert Stream Simulator** is a subsystem which encodes a static set of alerts and publishes them to the Kafka Cluster every 37 seconds.
6. The **Alert Database** is a subsystem which archives schemas and alerts which have been published to Kafka, storing them in Google Cloud Storage buckets. It also provides HTTP-based access to this archive.

.. figure:: ArchitectureDiagram.png

   A diagram of the principal components and their relationships.

Each of these components will now be described in more detail.

Strimzi Operator
----------------

Strimzi :cite:`strimzi` is a third-party software system for managing a Kafka cluster on Kubernetes.
It is used in the Alert Distribution System as an abstraction layer around the details of configuring Kafka on individual Kubernetes Pods and Nodes.

Strimzi works through Custom Resource Definitions, or "CRDs", which are installed once for the entire Kubernetes cluster across all namespaces.
This installation is performed automatically by Argo CD when installing the Strimzi Helm chart, as configured `in Phalanx <https://github.com/lsst-sqre/phalanx/tree/master/services/strimzi>`__ as the 'strimzi' service.

The Strimzi Operator is a long-running application on Kubernetes which does all the work of actually starting and stopping Kubernetes Pods which run Kafka.
It also sets up Kubernetes Secrets which are used for authentication to connect to the Kafka broker, and can install ingresses for providing external access to the Kafka broker.

The Alert Distribution System generally uses the default settings for the Strimzi Operator.
There are only two settings which are explicitly enabled:

.. code-block:: yaml

  watchNamespaces:
    - "alert-stream-broker"
  logLevel: "INFO"


``watchNamespaces`` is a list of Kubernetes *namespaces* to be watched for Strimzi Custom Resources by the Strimzi Operator.
In our case, this is configured to watch for any resources created in the ``alert-stream-broker`` namespace, since that namespace holds all the resources used to define the Alert Distribution System.
All resources go in one namespace; this is explained further in :ref:`single-namespace`.

``logLevel`` is set explicitly to ``INFO`` to enable logging by the Strimzi Operator itself.
Note that this configures the Operator, **not** the Kafka broker or anything else.
This can be set to ``DEBUG`` to help with debugging thorny internal issues.

Kakfa Cluster
-------------

The Kafka Cluster is at the heart of the Alert Distribution System, and is defined in terms of custom Strimzi resources.
These resources are defined with Helm templates in the `alert-stream-broker <https://github.com/lsst-sqre/charts/tree/master/charts/alert-stream-broker>`__ chart.

The chart has the following subresources:

 1. A ``Kafka`` resource which defines the cluster's size, listeners, and core configuration, including that of the ZooKeeper nodes, in `kafka.yaml`_.
 2. A ``Certificate`` resource used to provision a TLS certificate for the Kafka cluster's external address, defined in `certs.yaml`_.
 3. A list of ``KafkaUsers`` used to create client identities that can access the Kafka Cluster, defined in `users.yaml`_ and `superusers.yaml`_.
 4. A ``VaultSecret`` used to store superuser credentials in Vault, which provides gated human access to the credential values through 1Password; see the `Phalanx Documentation on VaultSecrets <https://phalanx.lsst.io/service-guide/add-a-onepassword-secret.html>`__ for more details. This is defined in `vault_secret.yaml`_.

.. _kafka.yaml: https://github.com/lsst-sqre/charts/blob/master/charts/alert-stream-broker/templates/kafka.yaml
.. _certs.yaml: https://github.com/lsst-sqre/charts/blob/master/charts/alert-stream-broker/templates/certs.yaml
.. _users.yaml: https://github.com/lsst-sqre/charts/blob/master/charts/alert-stream-broker/templates/users.yaml
.. _superusers.yaml: https://github.com/lsst-sqre/charts/blob/master/charts/alert-stream-broker/templates/superusers.yaml
.. _vault_secret.yaml: https://github.com/lsst-sqre/charts/blob/master/charts/alert-stream-broker/templates/vault_secret.yaml

These will each now be explained in further detail.

``Kafka`` resource
~~~~~~~~~~~~~~~~~~

The ``Kafka`` resource is the primary configuration object of the Kafka cluster, defined in `kafka.yaml`_.
There's a lot going on in its configuration; this section attempts to explain some of the most important sections without going every line.

.. _listeners:

Listeners
*********

The ``spec.kafka.listeners`` field of the resource defines the Kafka *listeners*, which are the network addresses which it opens to receive requests; this section is essential for configuring the Kafka cluster for both internal and external access.

Kafka's listeners are complicated, and configuring them through Kubernetes is even more so.
The Strimzi blog post series on "Accessing Kafka" :cite:`accessing-kafka`  provides very useful background for understanding this section.

We use three listeners: two internal listeners with ``tls`` authentication (meaning that clients need to use mTLS authentication to connect) and one external listener.

The first internal listener, on port 9092 and named 'internal', is used by applications internal to the Alert Distribution System, such as the Alert Database and Alert Stream Simulator.

The second internal listener, on port 9093 and named 'tls', is used by the Schema Registry, since it the Strimzi Registry Operator is currently hardcoded to only use configure a Registry to connect to a listener with that name.

Because these are ``internal``-typed listeners, they are only accessible within the Kubernetes cluster, not to any users from across the internet.

The third listener is an external one, meaning that it is accessible over the internet.
It is configured to be ``loadbalancer``-typed, which tells the Strimzi Operator that we would like a `Kubernetes Service with a type of LoadBalancer`_ to be provisioned on our behalf.
This, in turn, `triggers creation`_ of a Google Cloud Network Load Balancer, which has a public IP address which can be used to connect to the service.
There are two important things to note about this system.

.. _Kubernetes Service with a type of LoadBalancer:  https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer
.. _triggers creation: https://cloud.google.com/kubernetes-engine/docs/concepts/service#services_of_type_loadbalancer


First, it is fairly specific to Google Cloud Platform; an implementation of the Alert Distribution System on a different Kubernetes platform might require a different strategy for this external listener.

Second, it provisions an IP address automatically, without any explicit choice.
This is important because it means that we cannot automatically assign a DNS record to give a name to this external listener until the Kafka cluster has been created: we wouldn't know what IP address to have the DNS record resolve to.

This chicken-and-egg issue actually causes even more complexity, since without a valid DNS name we cannot use TLS encryption for connections to the broker, since the broker wouldn't have any hostname that it could claim.

This isn't really resolvable in a single resource creation step, but we *can* pin to a specific public IP address for the load balancer once it has already been provisioned using the ``spec.kafka.listeners.configuration.bootstrap.loadBalancerIP`` configuration field of the Strimzi ``Kafka`` resource.

The solution then is to require a multi-step process when first setting up the Kafka cluster.
First, the cluster is created without any explicit ``loadBalancerIP``.
The cluster will start with an unusable ``external`` listener, but a Google Cloud Network Load Balancer will be created.
That Load Balancer's IP address can be retrieved through the Google Cloud console, and then fed back in as the ``loadBalancerIP`` to be used by the ``Kafka`` resource, and also used to provision a DNS record for the broker's actual hostname

Then the broker can be updated, now with a valid ``external`` listener, and able to accept traffic.

Note that this needs to be done for *each broker replica*, in addition to the cluster-wide bootstrap address, since each broker needs to be separately accessible on the internet.
"Accessing Kafka" :cite:`accessing-kafka` is a useful reference to explain why this is necessary in greater detail.

An example of this pinning process can be found in Phalanx's set of values for the ``idfint`` environment of alert-stream-broker (`values-idfint.yaml`_), where the external listener's IP addresses have been pinned explicitly:

.. _values-idfint.yaml: https://github.com/lsst-sqre/phalanx/blob/66d2f3a2ae18efc79ebae7eb2763bf7e866e84a6/services/alert-stream-broker/values-idfint.yaml

.. code-block:: yaml

    # Addresses based on the state as of 2021-12-02; these were assigned by
    # Google and now we're pinning them.
    externalListener:
      bootstrap:
        ip: 35.188.169.31
        host: alert-stream-int.lsst.cloud
      brokers:
        - ip: 35.239.64.164
          host: alert-stream-int-broker-0.lsst.cloud
        - ip: 34.122.165.155
          host: alert-stream-int-broker-1.lsst.cloud
        - ip: 35.238.120.127
          host: alert-stream-int-broker-2.lsst.cloud

Broker Configuration
********************

The Apache Kafka configuration for the broker (that is, configuration using Java properties, just as Kafka documentation suggests) is handled through the 'config' field of `kafka.yaml`:

.. code-block:: yaml

    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      log.message.format.version: {{ .Values.kafka.logMessageFormatVersion }}
      inter.broker.protocol.version: {{ .Values.kafka.interBrokerProtocolVersion }}
      ssl.client.auth: required
      {{- range $key, $value := .Values.kafka.config }}
      {{ $key }}: {{ $value }}
      {{- end }}

These are not particularly chosen; they are merely intended to be sensible defaults for reasonable durability.

The ``log.message.format.version`` and ``inter.broker.protocol.version`` fields deserve extra explanation, however.
These need to be explicitly set to make it possible to upgrade Kafka's version.
For more on this, see `Strimzi documentation on these fields <https://strimzi.io/docs/operators/latest/full/deploying.html#ref-kafka-versions-str>`__.

Storage
*******

The Kafka cluster's storage (that is, the backing disks used to store alert packet data) is configured directly in the ``Kafka`` resource:

.. code-block:: yaml

    storage:
      type: jbod
      volumes:
        # Note that storage is configured per replica. If there are 3 replicas,
        # and 2 volumes in this array, each replica will get 2
        # PersistentVolumeClaims for the configured size, for a total of 6
        # volumes.
      - id: 0
        type: persistent-claim
        size: {{ .Values.kafka.storage.size }}
        class: {{ .Values.kafka.storage.storageClassName }}
        deleteClaim: false

The "``jbod``" storage type requests "just a bunch of disks" - a simple storage backend.
The requests for storage are handled through Kubernetes PersistentVolumeClaims, which request persistent disks from the Kubernetes controller.

On Google Kubernetes Engine, these end up requesting persistent disks; see `the GKE documentation <https://cloud.google.com/kubernetes-engine/docs/concepts/persistent-volumes>`__ for more detail.
By requesting disks with a StorageClass of "standard", these should be general purpose SSDs.

Note that these disks can be enlarged, but never shrunk.
This is a constraint of Strimzi in order to manage Kafka disk usage safely.

Node Pool
*********

The Kafka cluster is set to run on a dedicated Kubernetes *Node Pool*, which means that it runs on single-tenant hardware dedicated just to Kafka brokers.
This is configured through pod tolerations and affinities, as is standard in Kubernetes.

Using single-tenant hardware helps ensure that community brokers will receive stable levels of network connectivity to the Kafka brokers, and also helps avoid memory pressure issues if Kubernetes' scheduler oversubscribed pods onto nodes used by Kafka.

The Kafka node pool is labeled ``kafka=ok``; this label is used for all taints, tolerations, and affinities.
This node pool is created using Terraform in the `environments/deployments/science-platform/env/integration-gke.tfvars`_ file.

The 2018 Strimzi blog post "Running Kafka on dedicated Kubernetes nodes" :cite:`strimzi-kafka-nodes` provides a good guide on how this is implemented in more detail.

.. _environments/deployments/science-platform/env/integration-gke.tfvars: https://github.com/lsst/idf_deploy/blob/a4361659854d078ab823ee915a1136bc0fbd65ff/environment/deployments/science-platform/env/integration-gke.tfvars#L49-L64

TLS Certificate
~~~~~~~~~~~~~~~

The TLS certificate for the broker's external listener (see :ref:`listeners`) is configured through a ``Certificate`` custom resource.
This custom resource is used by the cert-manager :cite:`cert-manager` system which is already installed on the Kubernetes cluster.

This system works by provisioning LetsEncrypt TLS certificates automatically and storing them in TLS secrets.
The Strimzi blog post "Deploying Kafka with Let's Encrypt certificates" :cite:`kafka-letsencrypt` provides a detailed discussion of how this works, although it assumes the use of "ExternalDNS" to manage DNS records, which is different.
The Rubin Science Platform's DNS is managed manually by the SQuaRE team in Route53, so all DNS records were created manually.

The most important part of the ``Certificate`` resource is the ``dnsNames`` field which requests TLS certificates for specific hostnames.
In our Kafka installation, we need multiple such hostnames: one for each individual broker (``alert-stream-int-broker-0-int.lsst.cloud``, ``alert-stream-int-broker-1-int.lsst.cloud``, etc), and one for the cluster-wide bootstrap address (``alert-stream-int.lsst.cloud``).
As explained in :ref:`listeners`, these can only be fully configured once an IP address for an external load balancer has been provisioned, so this resource may fail when first created.

Users and Superusers
~~~~~~~~~~~~~~~~~~~~

Kafka Users are identities presented by clients and authenticated by the Kafka broker.
They have access boundaries which restrict which operations they can perform.
In the case of the Alert Distribution System, most users are limited to only working with a subset of topics.

The only exception is superusers who are granted global access to do anything.
These are administrative accounts which are only expected to be used by Rubin staff, and only in case of emergencies.

1Password, Vault, and Passwords
*******************************

User's passwords are set through the RSP-Vault 1Password vault in the LSST-IT 1Password account.
Each user gets a separate 1Password item with a name in the format "alert-stream idfint <username>", like "alert-stream idfint lasair-idint".

A username can be set in the 1Password item, but this is purely descriptive; the password is the only thing that is used.

The item uses a field named "generate_secrets_key" with a value of "alert-stream-broker <username>-password".
Through Rubin Science Platform's 1Password secret machinery, this will automatically generate a value in the ``alert-stream-broker-secrets`` Kubernetes Secret named "<username>-password" which stores the user's password; this can then be fed in to Kafka's configuration.

All most administrators really need to know, though, is:
 - Each Kafka user needs to have a separate item in the RSP-Vault 1Password vault.
 - The password stored in 1Password is authoritative.
 - Passwords can be securely distributed using 1Password's 'Private Link' feature.
 - The formatting of the 1Password item is persnickety and must be set exactly correctly.

Authentication
**************

Users authenticate using SCRAM-SHA-512 authentication, which is a username and password-based protocol.
The alert-stream-broker's `users.yaml`_ template configures each username, but lets passwords get generated separately and receives them through Kubernetes Secrets.
These passwords are then passed in to Kafka to configure the broker to expect them.

Access Restrictions
*******************

Users are granted read-only access to a configurable list of topics.
This access grants them the ability to read individual messages from the topics and to fetch descriptions of the topic configuration, but it grants them no access to publish messages or alter the topics in any way.

In addition, users are granted complete access to Kafka Consumer Groups which are prefixed with their username.
For example, the ``fink-idfint`` user may create, delete, or modify any groups named ``fink-idfint``, or ``fink-idfint-testing``, or ``fink-idfint_anythingtheylike``, but not any groups named ``antares-idfint`` or ``admin``.

The list of user identities to be created is maintained in Phalanx as a configuration value for the ``idfint`` environment in `values-idfint.yaml`_:

.. code-block:: yaml

  users:
    # A user for development purposes by the Rubin team, with access to all
    # topics in readonly mode.
    - username: "rubin-devel-idfint"
      readonlyTopics: ["*"]
      groups: ["rubin-devel-idfint"]

    # A user used by the Rubin team but with similar access to the community
    # broker users.
    - username: "rubin-communitybroker-idfint"
      readonlyTopics: ["alerts-simulated"]
      groups: ["rubin-communitybroker-idfint"]

    # The actual community broker users
    - username: "alerce-idfint"
      readonlyTopics: ["alerts-simulated"]
      groups: ["alerce-idfint"]

    - username: "ampel-idfint"
      readonlyTopics: ["alerts-simulated"]
      groups: ["ampel-idfint"]

    - username: "antares-idfint"
      readonlyTopics: ["alerts-simulated"]
      groups: ["antares-idfint"]

   # ... truncated

Explicitly listing every username like this would be clumsy for large numbers of users, but since there are a relatively small number of community brokers, this provides a simple mechanism.
Alternatives which hook into systems like LDAP are much, much more complicated to configure and might not have Strimzi support.

In the ``idfint`` environment, each user only gets access to the "alerts-simulated" topic which holds the alerts generated by the Alert Stream Simulator.

Strimzi Registry Operator
-------------------------

Strimzi Registry
----------------

Alert Stream Simulator
----------------------

Alert Database
--------------

Design Decisions
================

This section lists particular overall design decisions that went into the Alert Distribution System.

.. _single-namespace:

Single Namespace
----------------

All Strimzi and Kubernetes resources reside in the same namespace, with the exception of the Strimzi Operator and Strimzi Registry Operator.
This is done because it's the simplest way to allow internal authentication to the Kafka cluster using Kubernetes Secrets.

The Strimzi Operator creates Kubernetes Secrets for each ``KafkaUser`` associated with a Kafka cluster that it manages.
These Secrets hold all of the data required for a Kafka client to connect to the broker: TLS certificates, usernames, passswords - anything needed for a particular authentication mechanism.

The Secrets are created automatically, and will be updated or rotated automatically if the Kafka Cluster is changed.
In addition, they can be securely passed in to application code using Kubernetes' primitives for secret management, which gives us confidence that access is safe.
This hands-off system greatly simplifies the coordination processes that would be required if credentials were manually managed without Strimzi.

However, they come with a downside, which is that Secrets cannot be accessed across namespace boundaries; they must be resident in a single namespace and can only be used from there.
Strimzi chooses to create them in the same namespace as that of the ``Kafka`` resources.

Since we want to also use the Secrets for access from applications, this means that the applications need to all reside in the same namespace as the ``Kafka`` resource - effectively requiring that everything be in one namespace if it needs to access Kafka internally.

This isn't particularly consequential in practice, although it has a few downsides:

1. All applications need to be bundled together into one Phalanx service, resulting in a cluttered view with many, many resources in Argo CD's UI.
   This view can be hard to browse.
2. Applications may have access to more than is necessary, since Kubernetes Roles often grant access to resources within a namespace boundary.
   Bundling things into one namespace removes that protection.
   In practice, there aren't any Kubernetes permissions granted to any of the applications, so this may be a moot point at this time, but things may change as the system evolves.

As an alternative, the Kubernetes Secrets could be reflected into multiple namespaces using a custom Operator.
However, this would come at the cost of extra cluster-wide complexity.
If multiple systems on the cluster would take advantage of such an operator, it might be worthwhile overall.


.. .. rubric:: References

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
    :style: lsst_aa
