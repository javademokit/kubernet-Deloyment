SDLA Helm Chart Deployment
-----------------------------------
Cloning Repository for helm package:
 
1.clone the repository from gerrit like below:
	git clone https://gerrit.ext.net.nokia.com/gerrit/SDME/sdme && (cd sdme && curl -Lo `git rev-parse --git-dir`/hooks/commit-msg https://gerrit.ext.net.nokia.com/gerrit/tools/hooks/commit-msg; chmod +x `git rev-parse --git-dir`/hooks/commit-msg)
 
2. use the helmpackage under below location in the cloned repository for installation 
	sdme/packagingCLOUD/SDLA_CNF
 
Image creation :
 
1. download the tgz files which contains required images from repository like below:
	SDLA_CNF.tgz:
		wget https://artifactory-espoo1.ext.net.nokia.com/artifactory/sdmexpert-snapshots-local/sdme/vnf/sdla_cloud_native_poc/45-2021-08-20/SDLA_CNF.tgz
	DB_SIMULATOR.tgz:
		wget https://artifactory-espoo1.ext.net.nokia.com/artifactory/sdmexpert-snapshots-local/sdme/vnf/sdla_cloud_native_poc/54-2021-08-27/DB_SIMULATOR.tgz
 
2. upload the image using below command
	SDLA_CNF.tgz:
		gunzip -c SDLA_CNF.tgz | docker load
		below images will be uploaded:
			sdme_worker_cnf:<build_id>
			sdme_tomcat_loginapp:<build_id>
			sdme_tomcat_protea:<build_id>
	DB_SIMULATOR.tgz:
		gunzip -c DB_SIMULATOR.tgz | docker load
		below images will be uploaded:
			db_simulator:<build_id>
			db_simulator_haproxy:<build_id>
	Validation:
		validate whether images got uploaded by below command:
		SDLA_CNF images : docker images | grep sdme
		DB_SIMULATOR imges : docker images | grep db_simulator
 
3. Tag and push the images to registry one by one as per our requirement. We are using sdme_worker_cnf and sdme_tomcat_loginapp for our testing.
	sdme_worker_cnf:
		docker tag sdme_worker_cnf:<build_id> bcmt-registry:5000/sdme_worker_cnf:<build_id>
		docker push bcmt-registry:5000/sdme_worker_cnf:<build_id>
	sdme_tomcat_loginapp:
		docker tag sdme_tomcat_loginapp:<build_id> bcmt-registry:5000/sdme_tomcat_loginapp:<build_id>
		docker push bcmt-registry:5000/sdme_tomcat_loginapp:<build_id>
	sdme_tomcat_protea:
		docker tag sdme_tomcat_protea:<build_id> bcmt-registry:5000/sdme_tomcat_protea:<build_id>
		docker push bcmt-registry:5000/sdme_tomcat_protea:<build_id>
	db_simulator:
		docker tag db_simulator:<build_id> bcmt-registry:5000/db_simulator:<build_id>
		docker push bcmt-registry:5000/db_simulator:<build_id>
	db_simulator_haproxy:
		docker tag db_simulator_haproxy:<build_id> bcmt-registry:5000/db_simulator_haproxy:<build_id>
		docker push bcmt-registry:5000/db_simulator_haproxy:<build_id>
	Validation:
		validate whether images got tagged properly by below command:
			SDLA_CNF images : docker images | grep sdme
			DB_SIMULATOR imges : docker images | grep db_simulator

Steps to install SDLA CNF:

Changes needs to be done in steps and config files:
	1. We have to update the namespace details in the deployment steps and make sure that unique namespace is used.
	2. Update namespace and sdme_worker_cnf image detils in sdla_values.yaml.
	3. By defualt citm will deploy on the one of the controller node. If we want to deploy citm on any specific controller node or edge node  then we need to change the configuration in values.yaml of citm to make it deploy on edge nodes
	4. Make sure that namespace updated in sdla_values.yaml and other values.yaml muste be same with namespace used for deployment. 
	5. Below commands needs to be executed inside helmpackage folder which is cloned from repository.
	6. Template:- helm3 install <relaese name> -n <namespace> <helm chart path>

1. install rabbitmq
   helm3 install test-crmq -n <namespace> ./crmq/

   validation:
    helm3 ls -n <namespace> - to validate the deployment.
    kubectl get pods -n <namespace> - to validate the pods created. 3 pods will be created. 

2. install tomcat
   Before installation update loginapp/protea image details (which uploaded and tagged) and ZTS details in values.yaml.

   helm3 install test-tomcat -n <namespace> ./casf-tomcat/ 
   
   validation:
    helm3 ls -n <namespace> - to validate the deployment.
    kubectl get pods -n <namespace> - to validate the pods created. 1 pod will be created
    Check whether zts values got assigned the env variables got assigned in pod.
		kubectl exec -it <tomcat-pod-name> -n <namespace> bash - to login in to pod.
		printenv | grep ZTS - to view env values configured.
		example values:
			ZTS_UM=false
			ZTS_SERVER_IPS=10.93.74.214,10.93.74.216
			ZTS_REALM=sdlagui
 
3. install citm
   note:- by default it will deploy on controller node port 17447, if you want to deploy with diffrent configuration you can make respective changes in values.yaml file

   kubectl get pod --all-namespaces -l app=citm-ingress ( use this command to check number of citm installed on cluster and make changes according to it).


   helm3 install test-citm -n <namespace> ./citm-ingress/ 

   validation:
     helm3 ls -n <namespace> - to validate the deployment.
     kubectl get pods -n <namespace> - to validate the pods created. 2 pods will be created

4. install psp
	note:- PSP resource is not belongs to namespace rather its cluster resource, so in a cluster we cant install 2 psp's with same name.
       so use the release name which is not already being used.

   	helm3 install test-psp -n <namespace> ./psp/ -f sdla_values.yaml

	validation:
     	helm3 ls -n <namespace> - to validate the deployment.

5. install common chart ( contains sdme rpm, creating soft link and also for creating pvc)
   	Before installing update namespace in the values.yaml in common package.
   
	helm3 install test-common -n <namespace> ./common/ -f sdla_values.yaml
	
	validation:
     helm3 ls -n <namespace> - to validate the deployment.

6. install manager
   	helm3 install test-manager -n <namespace> ./manager/ -f sdla_values.yaml

	validation:
     helm3 ls -n <namespace> - to validate the deployment.
     kubectl get pods -n <namespace> - to validate the pods created. 1 pod will be created
     
7. install event-manager
   	helm3 install test-eventmanager -n <namespace> ./event-manager/ -f sdla_values.yaml

	validation:
     helm3 ls -n <namespace> - to validate the deployment.
     kubectl get pods -n <namespace> - to validate the pods created. 1 pod will be created
     
8. install syslogserver
   	helm3 install test-syslog -n <namespace> ./SyslogService/ -f sdla_values.yaml
	
	validation:
     helm3 ls -n <namespace> - to validate the deployment.
     kubectl get pods -n <namespace> - to validate the pods created. 1 pod will be created
     
9. install scheduler
   helm3 install test-scheduler -n <namespace> ./scheduler/ -f sdla_values.yaml

	validation:
     helm3 ls -n <namespace> - to validate the deployment.
     kubectl get pods -n <namespace> - to validate the pods created. 1 pod will be created
     
10. install job-manager
   helm3 install test-jobmanager -n <namespace> ./job-manager/ -f sdla_values.yaml
	
	validation:
     helm3 ls -n <namespace> - to validate the deployment.
     kubectl get pods -n <namespace> - to validate the pods created. 1 pod will be created
     
11. install migration workers
    helm3 install test-migration -n <namespace> ./migration/ -f sdla_values.yaml

	validation:
     helm3 ls -n <namespace> - to validate the deployment.
     kubectl get pods -n <namespace> - to validate the pods created. 17 pods will be created
 


12. install sdmetest
	Note:- no need to deploy sdmetest pod if you are using zts
    
	helm3 install test-sdme -n <namespace> ./sdmetest/ -f sdla_values.yaml
	
	validation:
     helm3 ls -n <namespace> - to validate the deployment.
     kubectl get pods -n <namespace> - to validate the pods created. 1 pod will be created

13. Install db  simulator
	Note :- no need to deploy simulator if we use real onends and sdl.
    
    helm3 install  test-simulator -n <namespace> ./simulator/
	
	validation:
     helm3 ls -n <namespace> - to validate the deployment.
     kubectl get pods -n <namespace> - to validate the pods created. 2 pod will be created (one for oneNDS and one for SDL)

Undeployment Steps:
-------------------------------------------------
Template:- helm3 delete -n <namespace> <relaese name>


1. helm3 delete -n <namespace> test-sdme
2. helm3 delete -n <namespace> test-migration
3. helm3 delete -n <namespace> test-jobmanager
4. helm3 delete -n <namespace> test-scheduler
5. helm3 delete -n <namespace> test-syslog
6. helm3 delete -n <namespace> test-eventmanager
7. helm3 delete -n <namespace> test-manager
8. helm3 delete -n <namespace> test-common
9. helm3 delete -n <namespace> test-psp
10. helm3 delete -n <namespace> test-citm
11. helm3 delete -n <namespace> test-tomcat
12. helm3 delete -n <namespace> test-crmq
13. helm3 delete -n <namespace> test-simulator


SDLA GUI :
--------------------------------------------

curl http://<IngressInstalledIP>:<citm http port>/loginapp/