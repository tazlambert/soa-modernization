
# Deploy Ingress Load Balancer for SOA in Kubernetes #

The Oracle WebLogic Server Kubernetes Operator supports Oracle Cloud Infrastructure various Load Balancer, for Test and Develop purposes we use Traefik. This tutorial demonstrates how to configure Kubernetes to provisions Ingress Controller and Load Balancer.

### Install Treafik  ###

First we need to create a namespace for Traefik
```
kubectl create namespace traefik
```
Set up Helm for 3rd party services
```
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
```
Install the Traefik operator in the traefik namespace with the provided sample values
```
cd ~/weblogic-kubernetes-operator/
helm install traefik-operator stable/traefik --namespace traefik --values kubernetes/samples/charts/traefik/values.yaml  --set "kubernetes.namespaces={traefik}" --set "serviceType=LoadBalancer"
```
Verify the Traefik (load balancer) services
```
kubectl get service -n traefik
```
Please note the EXTERNAL-IP of the traefik-operator service. 
This is the public IP address of the load balancer that you will use to access the WebLogic Server Administration Console and SOA URLs.

### Configure Treafik for Ingress  ###

Configure Traefik to manage Ingresses created in SOA Domain namespace (soans)
```
cd ~/weblogic-kubernetes-operator/kubernetes/samples/charts
helm upgrade --reuse-values --set "kubernetes.namespaces={traefik,soans}" --wait traefik-operator stable/traefik --namespace traefik
```
In order to access the Administration Console deployed on WebLogic and SOA Application URLs, you have to configure a Traefik Ingress.
An OCI load balancer with external IP i.e.,  ${TRAEFIK_PUBLIC_IP} is already assigned during the Traefik install in the previous step.
The values.yaml is already updated with below values: ( Note: Here we will be using the ${TRAEFIK_PUBLIC_IP} to access the URLs and hence hostname is left empty)
```
cd ~/weblogic-kubernetes-operator/kubernetes/samples/charts/ingress-per-domain/
vi values.yaml
```
Put the values like below
```
type: TRAEFIK

# WLS domain as backend to the load balancer
wlsDomain:
  domainUID: soainfra
  soaClusterName: soa_cluster
  osbClusterName: osb_cluster
  soaManagedServerPort: 8001
  osbManagedServerPort: 9001
  adminServerName: adminserver
  adminServerPort: 7001

# Traefik specific values
traefik:
  # hostname used by host-routing
  hostname:
```
And change the traefik template:
```
cd ~/weblogic-kubernetes-operator/kubernetes/samples/charts/ingress-per-domain/templates
vi traefik-ingress.yaml
```
Change the value into
```
# Copyright (c) 2018, 2020, Oracle Corporation and/or its affiliates.
# Licensed under the Universal Permissive License v 1.0 as shown at https://oss.oracle.com/licenses/upl.

{{- if eq .Values.type "TRAEFIK" }}
---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: {{ .Values.wlsDomain.domainUID }}-traefik
  namespace: {{ .Release.Namespace }}
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
    - host: '{{ .Values.traefik.hostname }}'
      http:
        paths:
        - path: /console
          backend:
            serviceName: '{{ .Values.wlsDomain.domainUID }}-{{ .Values.wlsDomain.adminServerName | lower | replace "_" "-" }}'
            servicePort: {{ .Values.wlsDomain.adminServerPort }}
        - path: /em
          backend:
            serviceName: '{{ .Values.wlsDomain.domainUID }}-{{ .Values.wlsDomain.adminServerName | lower | replace "_" "-" }}'
            servicePort: {{ .Values.wlsDomain.adminServerPort }}
        - path: /weblogic/ready
          backend:
            serviceName: '{{ .Values.wlsDomain.domainUID }}-{{ .Values.wlsDomain.adminServerName | lower | replace "_" "-" }}'
            servicePort: {{ .Values.wlsDomain.adminServerPort }}
        - path: 
          backend:
            serviceName: '{{ .Values.wlsDomain.domainUID }}-cluster-{{ .Values.wlsDomain.soaClusterName | lower | replace "_" "-" }}'
            servicePort: {{ .Values.wlsDomain.soaManagedServerPort }}			
        - path: /soa-infra
          backend:
            serviceName: '{{ .Values.wlsDomain.domainUID }}-cluster-{{ .Values.wlsDomain.soaClusterName | lower | replace "_" "-" }}'
            servicePort: {{ .Values.wlsDomain.soaManagedServerPort }}			
        - path: /soa/composer
          backend:
            serviceName: '{{ .Values.wlsDomain.domainUID }}-cluster-{{ .Values.wlsDomain.soaClusterName | lower | replace "_" "-" }}'
            servicePort: {{ .Values.wlsDomain.soaManagedServerPort }}	
        - path: /integration/worklistapp
          backend:
            serviceName: '{{ .Values.wlsDomain.domainUID }}-cluster-{{ .Values.wlsDomain.soaClusterName | lower | replace "_" "-" }}'
            servicePort: {{ .Values.wlsDomain.soaManagedServerPort }}				
        - path: /servicebus
          backend:
            serviceName: '{{ .Values.wlsDomain.domainUID }}-{{ .Values.wlsDomain.adminServerName | lower | replace "_" "-" }}'
            servicePort: {{ .Values.wlsDomain.adminServerPort }}
        - path: /lwpfconsole
          backend:
            serviceName: '{{ .Values.wlsDomain.domainUID }}-{{ .Values.wlsDomain.adminServerName | lower | replace "_" "-" }}'
            servicePort: {{ .Values.wlsDomain.adminServerPort }}
        - path: /ess
          backend:
            serviceName: '{{ .Values.wlsDomain.domainUID }}-cluster-{{ .Values.wlsDomain.soaClusterName | lower | replace "_" "-" }}'
            servicePort: {{ .Values.wlsDomain.soaManagedServerPort }}
        - path: /EssHealthCheck
          backend:
            serviceName: '{{ .Values.wlsDomain.domainUID }}-cluster-{{ .Values.wlsDomain.soaClusterName | lower | replace "_" "-" }}'
            servicePort: {{ .Values.wlsDomain.soaManagedServerPort }}			
        - path: /osb
          backend:
            serviceName: '{{ .Values.wlsDomain.domainUID }}-cluster-{{ .Values.wlsDomain.osbClusterName | lower | replace "_" "-" }}'
            servicePort: {{ .Values.wlsDomain.osbManagedServerPort }}
{{- end }}
```
Verify List of Services deployed by the Ingress
```
Name:             soainfra-traefik
Namespace:        soans
Address:
Default backend:  default-http-backend:80 (<none>)
Rules:
  Host  Path  Backends
  ----  ----  --------
  *
        /console                   soainfra-adminserver:7001 (10.244.0.134:7001)
        /em                        soainfra-adminserver:7001 (10.244.0.134:7001)
        /weblogic/ready            soainfra-adminserver:7001 (10.244.0.134:7001)
                                   soainfra-cluster-soa-cluster:8001 (10.244.0.135:8001,10.244.0.137:8001)
        /soa-infra                 soainfra-cluster-soa-cluster:8001 (10.244.0.135:8001,10.244.0.137:8001)
        /soa/composer              soainfra-cluster-soa-cluster:8001 (10.244.0.135:8001,10.244.0.137:8001)
        /integration/worklistapp   soainfra-cluster-soa-cluster:8001 (10.244.0.135:8001,10.244.0.137:8001)
        /servicebus                soainfra-adminserver:7001 (10.244.0.134:7001)
        /lwpfconsole               soainfra-adminserver:7001 (10.244.0.134:7001)
        /ess                       soainfra-cluster-soa-cluster:8001 (10.244.0.135:8001,10.244.0.137:8001)
        /EssHealthCheck            soainfra-cluster-soa-cluster:8001 (10.244.0.135:8001,10.244.0.137:8001)
        /osb                       soainfra-cluster-osb-cluster:9001 (10.244.0.136:9001,10.244.1.6:9001)
Annotations:
  kubernetes.io/ingress.class:  traefik
Events:                         <none>
```
Then you can test if the service is up or not:
```
username: weblogic
password: Welcome1
SOA:
http://${TRAEFIK_PUBLIC_IP}/weblogic/ready
http://${TRAEFIK_PUBLIC_IP}/console
http://${TRAEFIK_PUBLIC_IP}/em
http://${TRAEFIK_PUBLIC_IP}/soa-infra/
http://${TRAEFIK_PUBLIC_IP}/soa/composer
http://${TRAEFIK_PUBLIC_IP}/integration/worklistapp

ESS:
http://${TRAEFIK_PUBLIC_IP}/ess
http://${TRAEFIK_PUBLIC_IP}/EssHealthCheck
	
OSB:
http://${TRAEFIK_PUBLIC_IP}/servicebus
http://${TRAEFIK_PUBLIC_IP}/lwpfconsole
```
