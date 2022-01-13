:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

   We describe the current implementation of the Rubin Alert Distribution System at the interim data facility.
   An overview is provided of the system's concepts and components, and then each is described in detail.
   This document aims to be a point-in-time record of what exists, and to explain implementation decisions made during construction.


Overview
========

Rubin's Alert Distribution System is implemented in the integration environment at the interim data facility (the "IDF").
The implementation runs on the shared Rubin Science Platform Kubernetes cluster in the IDF.

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
Somewhat unconventionally, all of the alert stream resources are in one namespace, "alert-stream-broker", which will be explained in a later section.

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
pOn the Rubin Science Platform, Terraform is used to provision non-Kubernetes resources, such as Google Cloud Storage buckets.

Principal Components
====================

Strimzi Operator
----------------

Kakfa Cluster
-------------

Strimzi Registry Operator
-------------------------

Strimzi Registry
----------------

Alert Stream Simulator
----------------------

Alert Database
--------------


.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
    :style: lsst_aa
