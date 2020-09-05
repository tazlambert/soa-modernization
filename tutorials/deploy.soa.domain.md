# Deploy SOA Domain to Kubernetes #

In this step we are going to deploy the Oracle SOA Image in Persistent Volume.

### Preparing the Oracle File Server for SOA Suite Kubernetes PV and PVC ###

We start from the home page of the Oracle Cloud Console, and click the burger button and direct it to File Storage and choose Mount Target that will be acted as NFS Server with its IP server that will be used further.

![alt text](images/201.jpg)

Click "Create File System"

![alt text](images/202.jpg)

You can create File System and Mount Targets with the default values. But here in case you want to rename the file System and mount targets, follow below steps. Note: Make Sure the Virtual Cloud Network in Mount Target refers to the one where your OKE.

![alt text](images/203.jpg)

Edit and change the File System name to say "sharevolume".  Next "Edit" and change the Mount Target name to sharevolume and make sure the Virtual Cloud Network selected is "soademo" and the Subnet as "oke_workers" ( i.e., subnet of the worker nodes)  the one which our  is using. Click "Create"

![alt text](images/204.jpg)

Once the File System is created, it lands at below page. Click on "sharevolume" link.

![alt text](images/205.jpg)

Click on Mount Commands which gives details on how to mount this file system on your instances:

![alt text](images/206.jpg)

Mount Command pop up gives details on what must be configured on security list to access the mount targets from instances. Note down the mount command and Ingress/Egresss Rules which need to be added.

![alt text](images/207.jpg)

This filesystem will be used for many purpose, in this case;
- grafana
- prometheus
- prometheus alert
- soa domain home

Now for SOA Domain, it will requires its mounted directory to be in full permission mode (777) to do that we need to mount the /sharevolume to the bastion and create root folder and give 777 permission:
```
sudo mkdir /mnt/sharevolume
sudo mount 10.0.10.9:/sharevolume /mnt/sharevolume
sudo mkdir /mnt/sharevolume/soa
sudo chmod -Rf 777 /mnt/sharevolume/soa
```

### Preparing the Oracle Registry (OCIR) to get SOA domains and Database Docker images ###

Follow the below steps to configure OCIR to push and pull images from OCIR. Create an "Auth token" which will be used as docker password to push/pull images from OCIR. Login to Console and navigate to User Settings, which is in the drop down under your OCI username in the top nav 

![alt text](images/208.jpg)

On User Details page, select "Auth Tokens" in the left nav and then Click the "Generate Token" button. Enter a Name and Click "Generate Token"

![alt text](images/209.jpg)
![alt text](images/210.jpg)

Token will get generated, copy the generated token. NOTE: It will only be displayed this one time, and you will need to copy it to a secure place for further use.

![alt text](images/211.jpg)

Using the Docker CLI to login to OCIR ( for phoenix : phx.ocir.io , ashburn: iad.ocir.io, hyderabad: hyd.ocir.io etc) and we assume that you already have docker image for SOA Domain and Oracle Database in your bastion or PC.

```
docker login hyd.ocir.io -u axr17p4hbhaw/oracleidentitycloudservice/lambertus.wardana@oracle.com
docker tag localhost/oracle/database-enterprise:12.2.0.1-slim hyd.ocir.io/axr17p4hbhaw/oracle/database-enterprise:12.2.0.1
docker tag localhost/oracle/soa-suite:12.2.1.4 hyd.ocir.io/axr17p4hbhaw/oracle/dsoa-suite:12.2.1.4
docker push hyd.ocir.io/axr17p4hbhaw/oracle/database-enterprise:12.2.0.1
docker push hyd.ocir.io/axr17p4hbhaw/oracle/soa-suite:12.2.1.4
```

### Preparing WebLogic Kubernetes Operator for SOA domains in OKE ###

Prepare the environment for SOA Domain creation:
```
kubectl create namespace soans
```
Configure the operator to manage domain namespace
```
cd ~/weblogic-kubernetes-operator
helm upgrade --reuse-values --namespace opns --set "domainNamespaces={soans}" --wait weblogic-kubernetes-operator kubernetes/charts/weblogic-operator
```
Create a Kubernetes secret containing the Administration Server credentials (username : weblogic and password: Welcome1) in the same Kubernetes namespace as the domain (soans):
```
cd ~/weblogic-kubernetes-operator/kubernetes/samples/scripts/create-weblogic-domain-credentials
./create-weblogic-credentials.sh -u weblogic -p Welcome1 -n soans -d soainfra -s soainfra-domain-credentials
```
Create a Kubernetes secret for the RCU in the same Kubernetes namespace as the domain (soans) with below details:
Schema user          : SOA1
Schema password      : Oradoc_db1                    
DB sys user password : Oradoc_db1
Domain name          : soainfra
Domain Namespace     : soans
Secret name          : soainfra-rcu-credentials 
```
cd ~/weblogic-kubernetes-operator/kubernetes/samples/scripts/create-rcu-credentials
./create-rcu-credentials.sh -u SOA1 -p Oradoc_db1 -a sys -q Oradoc_db1 -d soainfra -n soans -s soainfra-rcu-credentials
```
Create OCI image Registry secret to allow Kubernetes to pull you custome WebLogic image. Replace the registry server region code, username and auth token respectively.
WARNING!!! - be careful about username - docker-username parameter should have a value of YOUR_OCIR_NAME/YOUR_OCIR_USERNAME - don't skip YOUR_TENANCY_NAME please.
```
kubectl create secret docker-registry ocirsecret -n soans --docker-server=hyd.ocir.io --docker-username='axrtkaqgdfo8/oracleidentitycloudservice/john.p.smith@testing.com' --docker-password='xxxxxxxxxx' --docker-email='john.p.smith@testing.com'
```
Then now we need to create PV and PVC for this domain:
```
cd
cd weblogic-kubernetes-operator/kubernetes/samples/scripts/create-weblogic-domain-pv-pvc
mkdir output
vi create-pv-pvc-inputs.yaml
```
Edit the input file
```
# Copyright (c) 2018, 2020, Oracle Corporation and/or its affiliates.
# Licensed under the Universal Permissive License v 1.0 as shown at https://oss.oracle.com/licenses/upl.

# The version of this inputs file.  Do not modify.
version: create-weblogic-sample-domain-pv-pvc-inputs-v1

# The base name of the pv and pvc
baseName: domain

# Unique ID identifying a domain.
# If left empty, the generated pv can be shared by multiple domains
# This ID must not contain an underscope ("_"), and must be lowercase and unique across all domains in a Kubernetes cluster.
domainUID: soainfra

# Name of the namespace for the persistent volume claim
namespace: soans

# Persistent volume type for the persistent storage.
# The value must be 'HOST_PATH' or 'NFS'.
# If using 'NFS', weblogicDomainStorageNFSServer must be specified.
weblogicDomainStorageType: NFS

# The server name or ip address of the NFS server to use for the persistent storage.
# The following line must be uncomment and customized if weblogicDomainStorateType is NFS:
weblogicDomainStorageNFSServer: 10.0.10.9

# Physical path of the persistent storage.
# When weblogicDomainStorageType is set to HOST_PATH, this value should be set the to path to the
# domain storage on the Kubernetes host.
# When weblogicDomainStorageType is set to NFS, then weblogicDomainStorageNFSServer should be set
# to the IP address or name of the DNS server, and this value should be set to the exported path
# on that server.
# Note that the path where the domain is mounted in the WebLogic containers is not affected by this
# setting, that is determined when you create your domain.
# The following line must be uncomment and customized:
weblogicDomainStoragePath: /sharevoluem/soa

# Reclaim policy of the persistent storage
# The valid values are: 'Retain', 'Delete', and 'Recycle'
weblogicDomainStorageReclaimPolicy: Retain

# Total storage allocated to the persistent storage.
weblogicDomainStorageSize: 10Gi
```
After finish editing the input file then we can execute the process to create new PV n PVC for WebLogic image:
```
./create-pv-pvc.sh -i create-pv-pvc-inputs.yaml -o output/ -e
```
Expected output will be like below:
```
Input parameters being used
export version="create-weblogic-sample-domain-pv-pvc-inputs-v1"
export baseName="domain"
export domainUID="soainfra"
export namespace="soans"
export weblogicDomainStorageType="NFS"
export weblogicDomainStorageNFSServer="10.0.10.9"
export weblogicDomainStoragePath="/sharevolume/soa"
export weblogicDomainStorageReclaimPolicy="Retain"
export weblogicDomainStorageSize="10Gi"

Generating output//pv-pvcs/soainfra-domain-pv.yaml
Generating output//pv-pvcs/soainfra-domain-pvc.yaml
Checking if the persistent volume soainfra-domain-pv exists
The persistent volume soainfra-domain-pv does not exist
Creating the persistent volume soainfra-domain-pv
persistentvolume/soainfra-domain-pv created
Checking if the persistent volume soainfra-domain-pv is Available
Checking if the persistent volume claim soainfra-domain-pvc in NameSpace soans exists
No resources found in soans namespace.
The persistent volume claim soainfra-domain-pvc does not exist in NameSpace soans
Creating the persistent volume claim soainfra-domain-pvc
persistentvolumeclaim/soainfra-domain-pvc created
Checking if the persistent volume soainfra-domain-pv is Bound
The following files were generated:
  output//pv-pvcs/soainfra-domain-pv.yaml
  output//pv-pvcs/soainfra-domain-pvc.yaml

Completed
```
Create the PV and PVC using the configuration files created in previous step as shown below:
```
kubectl create -f  output/pv-pvcs/soainfra-domain-pv.yaml
kubectl create -f  output/pv-pvcs/soainfra-domain-pvc.yaml
```
### Deploy Oracle Database and SOA RCU on Kubernetes ###

The Oracle Database Docker images are supported only for non-production use. For more details, see My Oracle Support note: Oracle Support for Database Running on Docker (Doc ID 2216342.1). For production usecase it is suggested to use a standalone db. Below are steps to create the database in a container without using PV. 
```
cd ~/weblogic-kubernetes-operator/kubernetes/samples/scripts/create-oracle-db-service
./start-db-service.sh -i  hyd.ocir.io/axr17p4hbhaw/oracle/database-enterprise:12.2.0.1 -s image-secret -n soans
```
Once database is created successfully, you can use the database connection string, "oracle-db.soans.svc.cluster.local:1521/devpdb.k8s", as an rcuDatabaseURL parameter in the create-domain-inputs.yaml file.
```
Done ! The database is ready for use .
Oracle DB Service is RUNNING with NodePort [30011]
Oracle DB Service URL [oracle-db.soans.svc.cluster.local:1521/devpdb.k8s]
```
Run the RCU to create SOA schemas, to install SOA schemas, run the "create-rcu-schema.sh" script with below inputs:
-s <RCU PREFIX>   Here: SOA1 
-t <SOA domain type>  Here: soaessosb
-p <ImagePullSecret name> Here: image-secret
-d <DB connection string>  Here: oracle-db.soans.svc.cluster.local:1521/devpdb.k8s 
-i <SOASuite image>   Here: hyd.ocir.io/axr17p4hbhaw/oracle/soa-suite:12.2.1.4

```  
cd ~/weblogic-kubernetes-operator/kubernetes/samples/scripts/create-rcu-schema
./create-rcu-schema.sh -s SOA1 -t soaessosb -d oracle-db.soans.svc.cluster.local:1521/devpdb.k8s  -p image-secret -i hyd.ocir.io/axr17p4hbhaw/oracle/soa-suite:12.2.1.4 -n soans -q Oradoc_db1 -r Oradoc_db1
```

Expected output will be:
```
Component schemas created:
-----------------------------
Component                                    Status         Logfile

Common Infrastructure Services               Success        /tmp/RCU2020-06-01_15-36_974422936/logs/stb.log
Oracle Enterprise Scheduler                  Success        /tmp/RCU2020-06-01_15-36_974422936/logs/ess.log
Oracle Platform Security Services            Success        /tmp/RCU2020-06-01_15-36_974422936/logs/opss.log
SOA Infrastructure                           Success        /tmp/RCU2020-06-01_15-36_974422936/logs/soainfra.log
User Messaging Service                       Success        /tmp/RCU2020-06-01_15-36_974422936/logs/ucsums.log
Audit Services                               Success        /tmp/RCU2020-06-01_15-36_974422936/logs/iau.log
Audit Services Append                        Success        /tmp/RCU2020-06-01_15-36_974422936/logs/iau_append.log
Audit Services Viewer                        Success        /tmp/RCU2020-06-01_15-36_974422936/logs/iau_viewer.log
Metadata Services                            Success        /tmp/RCU2020-06-01_15-36_974422936/logs/mds.log
WebLogic Services                            Success        /tmp/RCU2020-06-01_15-36_974422936/logs/wls.log

Repository Creation Utility - Create : Operation Completed
[INFO] Modify the domain.input.yaml to use [oracle-db.soans.svc.cluster.local:1521/devpdb.k8s] as rcuDatabaseURL and [SOA1] as rcuSchemaPrefix
```  

### Deploy Oracle SOA Domain on Kubernetes ###

To deploy SOA domain you need to create a domain resource definition which contains the necessary parameters for the operator to start the SOA domain properly. You can modify the provided sample in the local repository or better if you make a copy first.
```
cd ~/weblogic-kubernetes-operator/kubernetes/samples/scripts/create-soa-domain/domain-home-on-pv
vi create-domain-inputs.yaml
```
Use your favourite text editor to modify domain resource definition values. If necessary remove comment leading character (#) of the parameter to activate. Always enter space before the value, after the colon.
```
domainType: soaessosb
initialManagedServerReplicas: 1
image: hyd.ocir.io/axr17p4hbhaw/oracle/soa-suite:12.2.1.4
imagePullSecretName: image-secret
rcuDatabaseURL: oracle-db.soans.svc.cluster.local:1521/devpdb.k8s
```
Run the create-domain.sh script to create a domain:
```
cd ~/weblogic-kubernetes-operator/kubernetes/samples/scripts/create-soa-domain/domain-home-on-pv
./create-domain.sh -i create-domain-inputs.yaml -o output
```
Expected output will be:
```
Domain soainfra was created and will be started by the WebLogic Kubernetes Operator

The following files were generated:
  output/weblogic-domains/soainfra/create-domain-inputs.yaml
  output/weblogic-domains/soainfra/create-domain-job.yaml
  output/weblogic-domains/soainfra/domain.yaml

Completed
```
Once the create-domain.sh is success, it generates the "output/weblogic-domains/soainfra/domain.yaml" using which you can create the Kubernetes resource domain which starts the domain and servers as shown below:
```
cd ~/weblogic-kubernetes-operator/kubernetes/samples/scripts/create-soa-domain/domain-home-on-pv
kubectl create -f output/weblogic-domains/soainfra/domain.yaml
```
Verify that Kubernetes Domain Object (name: soainfra)  is created:
```
kubectl get domain -n soans
NAME       AGE
soainfra   28s
```
Once you create the domain, "introspect pod" will get created. This inspects the Domain Home and then start the "soainfra-adminserver" pod.  Once the "soainfra-adminserver" pod comes up successfully, then the managed server pods are started in parallel. Watch the "soans" namespace for the status of domain creation with below command:
```
kubectl get po -n soans -o wide -w
```
Verify that SOA Domain server pods and services are created and in READY state.
```
$  kubectl get all -n soans
NAME                                             READY   STATUS      RESTARTS   AGE
pod/oracle-db-78b7566996-vsg89                   1/1     Running     0          27h
pod/rcu                                          1/1     Running     0          26h
pod/soainfra-adminserver                         1/1     Running     0          23m
pod/soainfra-create-soa-infra-domain-job-9n89n   0/1     Completed   0          23h
pod/soainfra-osb-server1                         1/1     Running     0          19m
pod/soainfra-soa-server1                         1/1     Running     0          19m

NAME                                   TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)          AGE
service/oracle-db                      LoadBalancer   10.96.6.36      129.146.236.86   1521:30011/TCP   27h
service/soainfra-adminserver           ClusterIP      None            <none>           7001/TCP         23h
service/soainfra-cluster-osb-cluster   ClusterIP      10.96.237.199   <none>           9001/TCP         23h
service/soainfra-cluster-soa-cluster   ClusterIP      10.96.166.96    <none>           8001/TCP         23h
service/soainfra-osb-server1           ClusterIP      None            <none>           9001/TCP         23h
service/soainfra-osb-server2           ClusterIP      10.96.174.34    <none>           9001/TCP         19m
service/soainfra-osb-server3           ClusterIP      10.96.250.216   <none>           9001/TCP         19m
service/soainfra-osb-server4           ClusterIP      10.96.253.13    <none>           9001/TCP         19m
service/soainfra-osb-server5           ClusterIP      10.96.72.25     <none>           9001/TCP         19m
service/soainfra-soa-server1           ClusterIP      None            <none>           8001/TCP         23h
service/soainfra-soa-server2           ClusterIP      10.96.128.229   <none>           8001/TCP         19m
service/soainfra-soa-server3           ClusterIP      10.96.251.159   <none>           8001/TCP         19m
service/soainfra-soa-server4           ClusterIP      10.96.149.141   <none>           8001/TCP         19m
service/soainfra-soa-server5           ClusterIP      10.96.49.11     <none>           8001/TCP         19m

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/oracle-db   1/1     1            1           27h

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/oracle-db-78b7566996   1         1         1       27h

NAME                                             COMPLETIONS   DURATION   AGE
job.batch/soainfra-create-soa-infra-domain-job   1/1           4m51s      23h
```

#### Testing REST API ####

Below is the sample if we access from REST API, in this case to find what is the domain name that have just deployed:
```
./restClient.sh 10.0.10.15 operator/latest/domains
```
Below is the expected results:
```
-----BEGIN CERTIFICATE-----
MIIDGDCCAgCgAwIBAgIEH6YWnzANBgkqhkiG9w0BAQsFADAcMRowGAYDVQQDExF3
ZWJsb2dpYy1vcGVyYXRvcjAeFw0yMDA1MTAwNjE0MjFaFw0zMDA1MDgwNjE0MjFa
MBwxGjAYBgNVBAMTEXdlYmxvZ2ljLW9wZXJhdG9yMIIBIjANBgkqhkiG9w0BAQEF
AAOCAQ8AMIIBCgKCAQEAmJvo6Hfj783ztbxgTAXkcmT9eIt0asKgj+RKVWXWD3Sa
F+EsuL4XQATPsYc84Ak9hKQkR3WaSVA7vkEz7C6R++8w6TTHtYA25ebqZHQ0MSuR
XamFoF/7zJFL4YZRPR52/aPw6MwROtPKP1CHTlVgiuB9yf5nlgorLomThRagX8If
3pzmVZ80aibyViy611KIuJxZn2JfwRl2LAifxC+RtUtsQtjbE+tH1+b2Z4FQdR0E
SWpwd4j4u/YAPmBjT/0tw/Y1xGzWcHejpRWMgUBNyhzGoee1lhDRW+1LiVjb94mm
0i5yy/8ZvUOST27bUcOt4G2OEaNpAWTWVo9Fvzyf9wIDAQABo2IwYDALBgNVHQ8E
BAMCA/gwMgYDVR0RBCswKYIJbG9jYWxob3N0hwR/AAABhwSBksSLhwSTmmdphwQK
AAoPhwQKAAoQMB0GA1UdDgQWBBRgfUINa1OZ+jGY4nWT7doR98B1+DANBgkqhkiG
9w0BAQsFAAOCAQEAQ8d1izy6Xsqh5u4sWU2XB/+2tQVlpXfW0KFA8TKQRxfJIINR
PG82CyGIqcGf6pVvzVmHVjcv3jRrcQLu5UNfLhDi71UyC4wBsTxG7UjYJcKi5H3j
5XojxeWodFRNS28/mMWFl4glZh/efXJSu3fwAfIDh5e0yCEtrxYx45DbfhT3eetM
JvKd7NfQBWKC5kpb0WAaLicMs8ufOMWW0237sAks8haTctzUMfnXlwEd5ic4H8FO
u0bhI+0JVZu9JA+BQoolSerlbTx0fnYzbLDRLQuSFzAPxcjCrRiU8aelEqRBy3JF
LjgDVQtZ4xkOZeDC4ddgHc9hipQLvu4XVQUIZg==
-----END CERTIFICATE-----
Ready to call operator REST APIs
* About to connect() to 10.0.10.15 port 31001 (#0)
*   Trying 10.0.10.15...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* Connected to 10.0.10.15 (10.0.10.15) port 31001 (#0)
* Initializing NSS with certpath: sql:/etc/pki/nssdb
*   CAfile: /tmp/operator.cert.pem
  CApath: none
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* SSL connection using TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
* Server certificate:
*       subject: CN=weblogic-operator
*       start date: May 10 06:14:21 2020 GMT
*       expire date: May 08 06:14:21 2030 GMT
*       common name: weblogic-operator
*       issuer: CN=weblogic-operator
> GET /operator/latest/domains/ HTTP/1.1
> User-Agent: curl/7.29.0
> Host: 10.0.10.15:31001
> Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJ3ZWJsb2dpYy1vcGVyYXRvci1ucyIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJ3ZWJsb2dpYy1vcGVyYXRvci1zYS10b2tlbi1seng3dCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJ3ZWJsb2dpYy1vcGVyYXRvci1zYSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjA0ZTcwNWQwLWZiZTgtNDgzYi05NjMwLTgwNTQ2MDM1ZmQ5YyIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDp3ZWJsb2dpYy1vcGVyYXRvci1uczp3ZWJsb2dpYy1vcGVyYXRvci1zYSJ9.ybBcexm36bvVgVm62gjxAHhcfRq_k_um163oEvdcPk-sPXy8DMUk413BsvI9EptAUkuEMx8VJodAf7IkyiiB4rDLvhfbiBN9rtLP1LUPu_oScfCiPEKgQDqqdnC5UBtOENRL1s83otL5hIjB3hhIkhvKaW5lSzeJnEUxAIb_qOw2_fKiM5j_KnHlJQZRjjGlJ28WYJ-IGflfc0Qu69OScDsSI_gXml296EJXYln_wRfc7V6lwcOGuexiwE7j3HAdnLvcLdZxNeu_sAbbhudJi1TiDZEtHopAtjiRSHwXkcqW2hN7uOvlXBYO0CsQRtKaljPutm9QUED864_a64D0EA
> Accept:application/json
>
< HTTP/1.1 200 OK
< Content-Type: application/json
< Content-Length: 396
<
{ [data not shown]
100   396  100   396    0     0   1100      0 --:--:-- --:--:-- --:--:--  1100
* Connection #0 to host 10.0.10.15 left intact
{
  "links": [
    {
      "rel": "self",
      "title": "",
      "href": "/operator/latest/domains"
    },
    {
      "rel": "canonical",
      "title": "",
      "href": "/operator/latest/domains"
    },
    {
      "rel": "parent",
      "title": "",
      "href": "/operator/latest"
    }
  ],
  "items": [
    {
      "links": [
        {
          "rel": "self",
          "title": "",
          "href": "/operator/latest/domains/soainfra-domain"
        },
        {
          "rel": "canonical",
          "title": "",
          "href": "/operator/latest/domains/soainfra-domain"
        }
      ],
      "domainUID": "soainfra-domain"
    }
  ]
}
```
