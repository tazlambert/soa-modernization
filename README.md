# Welcome to Oracle SOA Modernization Hands On Lab #

## About this hands-on Hands On Lab ##

This Hands On Lab main purpose is to show how to modernization of existing Oracle SOA Deployment so that SOA's user will be able to experience SOA's improvement in this agile era. This hands on lab is made based on the [WebLogic Kubernetes Operator fo SOA Documentation](https://oracle.github.io/weblogic-kubernetes-operator/samples/simple/domains/soa-domain/) also various references and has been tested by September 2020. If you have any questions or inquiries please email me to lambertus.wardana@oracle.com.

**Note:** This is a work in progress tutorial, and only meant for Non Production Environment.

The Hands On Lab will demonstrates **Installing Oracle SOA Domain on Oracle Kubernetes Engine** using [Oracle Cloud Infrastructure](https://docs.cloud.oracle.com/en-us/iaas/Content/GSG/Concepts/baremetalintro.htm), [Oracle Container Engine for Kubernetes](https://docs.cloud.oracle.com/en-us/iaas/Content/ContEng/Concepts/contengoverview.htm), [Oracle Registry](https://docs.cloud.oracle.com/en-us/iaas/Content/Registry/Concepts/registryoverview.htm), and lastly [WebLogic Kubernetes Operator](https://github.com/oracle/weblogic-kubernetes-operator).

This lab is designed for people with some experience with OCI, Kubernetes, Oracle WebLogic, Oracle SOA, and no experience with WebLogic Tooling, Container Registry, Docker and want to learn the core concepts and basics of how to run application on Kubernetes environment.

## Prerequisites ##

+ **[Oracle Cloud Infrastructure](https://cloud.oracle.com/en_US/cloud-infrastructure)** enabled account. The tutorial has been tested using [Trial account](https://myservices.us.oraclecloud.com/mycloud/signup) (as of April, 2020).
+ **Linux Desktop with Oracle Cloud Infrastructure CLI, kubectl, helm.** Preferred to have bastion host in OCI
+ **[Docker](https://hub.docker.com/signup) account.**
+ **[Oracle Support](https://support.oracle.com/portal/) account.**
+ **Oracle SOA Docker Image [Refer to this Dcumentations](https://oracle.github.io/weblogic-kubernetes-operator/userguide/managing-fmw-domains/soa-suite/#creating-a-soa-suite-docker-image).**

### Installing Oracle SOA Domain on Oracle Kubernetes Engine ###

This will demonstrate the process of installing Oracle SOA domain into Oracle Kubernetes Engine. The domain will contain demo Web Application which is a simple JSP page which shows WebLogic Domain's MBean attributes. During moving process, improvement process also being done by adding new monitoring and logging capability using WME and WLE. Below is the illustration

1. [Setup Oracle Kubernetes Engine to orchestrate the Docker image](tutorials/setup.oke.md)
2. [Setup WebLogic Kubernetes Operator to orchestrate SOA in Kubernetes](tutorials/setup.wko.md)
3. [Deploy SOA Domain to Kubernetes](tutorials/deploy.weblogic.md)
4. [Deploy Ingress Load Balancer for SOA in Kubernetes](tutorials/deploy.load.balancer.md)
5. [Update SOA deployment to improve existing SOA domain](tutorials/update.domain.docker.image.md)

## Components ##

By the time this hands on lab created below are the tools that being used and its version:

### WebLogic Tooling ###
+ **[Oracle Weblogic Deploy Tooling (WDT) v1.8.1](https://github.com/oracle/weblogic-deploy-tooling/releases/tag/weblogic-deploy-tooling-1.8.1)** 
+ **[Oracle Weblogic Image Tooling (WIT) v1.8.5](https://github.com/oracle/weblogic-image-tool/releases/tag/release-1.8.5)** 
+ **[Oracle Weblogic Monitoring Exporter (WME) v1.1.2](https://github.com/oracle/weblogic-monitoring-exporter/releases/tag/v1.1.2)** 
+ **[Oracle Weblogic Logging Exporter (WLE) v1.0.0](https://github.com/oracle/weblogic-logging-exporter/releases/tag/v1.0.0)** 
+ **[Oracle Weblogic Kubernetes Operator (WKO) v3.0.1](https://github.com/oracle/weblogic-kubernetes-operator/releases/tag/v3.0.1)** 
### Others Tooling ###
+ **[Oracle Command Line Interface (OCI) v2.12.9](https://github.com/oracle/oci-cli/releases/tag/v2.12.9)**
+ **[Kubectl v1.15.7](https://github.com/kubernetes/kubectl/releases/tag/kubernetes-1.15.7)**  
+ **[Helm v3.1.1](https://github.com/helm/helm/releases/tag/v3.1.1)** 
+ **[Kubernetes v1.16.8](https://docs.cloud.oracle.com/en-us/iaas/releasenotes/changes/03abf6f3-f311-47e5-ba18-c3eb19577a22/)** 
+ **[Kube Prometheus v0.3.0](https://github.com/coreos/kube-prometheus/releases/tag/v0.3.0)** 
+ **[Elasticsearch v6.8.0](https://github.com/elastic/elasticsearch/releases/tag/v6.8.0)** 
+ **[Kibana v6.8.0](https://github.com/elastic/kibana/releases/tag/v6.8.0)** 
