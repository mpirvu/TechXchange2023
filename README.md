# TechXchange2023

This repository contains artifacts to demonstrate some of the benefits
of Semeru Cloud Compiler (aka OpenJ9 JITServer).
The architecture of the experiment is shown in this picture:

![architecture](DocImages/architecture.png)

The application used as a benchmark is AcmeAirEE8, a Java EE8 application that implements an airline reservation system and runs on top of OpenLiberty.
There are two deployments of this application: one that can offload JIT compilations to a Semeru Cloud Compiler service (the one depicted at the bottom)
and one that does not benefit from Semeru Cloud Compiler (the one depicted at the top).
We will refer to these two deployments as "AcmeAir Baseline" and "AcmeAir with Semeru Cloud Compiler" respectively.
Both deployments use the same database (mongodb).
Load is applied to both deployments using two identical JMeter containers.
Throughput values are sent by the JMeter instances to an influxdb database and from there to a grafana container to be displayed in a dashboard.
The two AcmeAir deployments together with the Semeru Cloud Compiler service and mongodb service are deployed in OpenShift and use Knative to scale up and down based on the load.
The rest of the pods (for JMeter, influxdb and grafana) run outside of OpenShift on a separate machine.



Login as root using the password provided in the Lab Guide:
```
su --login root
```

Clone the repository
```
cd /home/techzone/Lab-SCC
```
```
git clone https://github.com/mpirvu/TechXchange2023.git
```
```
cd TechXchange2023
```

> **NOTE**: If you prefer to use an IDE to view the project, you can run VSCode with admin privileges using the following command: 
> ```bash
> sudo code --no-sandbox --user-data-dir /home/techzone
> ```


Then follow the instructions below.

1. Find the IP of the current machine with `ifconfig`. It should be something like "10.xxx.xxx.xxx".
   Then use the `./searchReplaceIPAddress.sh` script to change all "9.46.81.11" addresses from bash scripts to the IP of the current machine.
   Use the following command to do this (after replacing xxx.xxx.xxx.xxx with the IP of the current machine)
   ```
   ./searchReplaceIPAddress.sh 9.46.81.11 xxx.xxx.xxx.xxx
   ```

2. Start the influxdb container in normal operation mode:
   ```
   ./startInflux.sh
   ```
   Verify the influxdb container is up and running:
   ```
   podman ps -a | grep influxdb
   ```

   Create another data bucket:
   ```
   podman exec influxdb influx bucket create -n jmeter2 -o IBM -r 1d
   ```

   Verify that two buckets called jmeter and jmeter2 exist:
   ```
   podman exec influxdb influx bucket list
   ```

3. Create the JMeter container:
   ```
   cd BuildImages/JMeterContext
   ```
   ```
   ./build_jmeter.sh
   ```
   ```
   cd ../..
   ```

4. Create the mongodb container
   ```
   cd BuildImages/MongoContext
   ```
   ```
   ./build_mongo.sh
   ```
   ```
   cd ../..
   ```

5. Create the two AcmeAir containers (with and without InstantON).
   ```
   cd BuildImages/LibertyContext
   ```
   ```
   ./build_acmeair.sh
   ```
   ```
   ./checkpoint.sh
   ```
   ```
   cd ../..
   ```

6. Push the images for AcmeAir and mongodb to the OCP private repository

   **IMPORTANT**: Login to the OCP console, using the desktop URL provided in the TechZone OCP server reservation.

   From the OCP console UI, click the username in the top right corner, and select `Copy login command`.

   ![ocp-cli-login](DocImages/ocp-cli-login.png)

   Press `Display Token` and copy the `Log in with this token` command.

   ![ocp-cli-token](DocImages/ocp-cli-token.png)

   Paste the command into your terminal window. You should receive a confirmation message that you are logged in.

   Once logged in, create and switch to the "sccproject-[Your initial]" namespace:

   > **NOTE**: If you are working on a cluster that is shared with others, please ensure that you are using a unique project name. We recommend using the format sccproject- followed by your initials. For example, sccproject-rm.
   
   ```
   oc new-project sccproject-[Your initial]

   oc project sccproject-[Your initial]
   ```

   After that, log in to the OCP registry:
   ```
   oc registry login --insecure=true
   ```

   Enable the default registry route in OCP to push images to its internal repos

   ```
   oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge
   ```

   Log Podman into the OCP registry server

   First we need to get the `TOKEN` that we can use to get the password for the registry.

   ```
   oc get secrets -n openshift-image-registry | grep cluster-image-registry-operator-token
   ```

   Take note of the `TOKEN` value, as you need to substitute it in the following command that sets the registry password.

   ```
   export OCP_REGISTRY_PASSWORD=$(oc get secret -n openshift-image-registry cluster-image-registry-operator-token-<INPUT_TOKEN> -o=jsonpath='{.data.token}{"\n"}' | base64 -d)
   ```

   Now set the OCP registry host value.

   ```
   export OCP_REGISTRY_HOST=$(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')
   ```

   Finally, we have the values needed for podman to login into the OpenShift registry server.

   ```
   podman login -p $OCP_REGISTRY_PASSWORD -u kubeadmin $OCP_REGISTRY_HOST --tls-verify=false
   ```

   Now tag and push the images for AcmeAir and mongodb to the OCP private repository.

   - mongo-acmeair image

   > ```bash
   > podman tag localhost/mongo-acmeair:5.0.17 $(oc registry info)/$(oc project -q)/mongo-acmeair:5.0.17
   > 
   > podman push $(oc registry info)/$(oc project -q)/mongo-acmeair:5.0.17 --tls-verify=false
   > ```

   - liberty-acmeair-ee8 image

   > ```bash
   > podman tag localhost/liberty-acmeair-ee8:23.0.0.6 $(oc registry info)/$(oc project -q)/liberty-acmeair-ee8:23.0.0.6
   > 
   > podman push $(oc registry info)/$(oc project -q)/liberty-acmeair-ee8:23.0.0.6 --tls-verify=false
   > ```

   - liberty-acmeair-ee8 instantOn image

   > ```bash
   > podman tag localhost/liberty-acmeair-ee8:23.0.0.6-instanton $(oc registry info)/$(oc project -q)/liberty-acmeair-ee8:23.0.0.6-instanton
   > 
   > podman push $(oc registry info)/$(oc project -q)/liberty-acmeair-ee8:23.0.0.6-instanton --tls-verify=false
   > ```

   - Verify the images have been pushed to the OpenShift image repository

   > ```bash
   > oc get imagestream
   > ```

7. Start the grafana container:
   ```
   ./startGrafana.sh
   ```

8. Using the grafana UI in a local browser (http://localhost:3000), configure grafana to get data from influxdb.

   Note: Initial credentials are admin/admin. You can skip the password change.

   Note: There are two pre-configured datasources called InfluxDB and InfluxDB2. However, the IP address of these datasources needs to be adjusted.

   To configure the datasources, select the gear (Configuration) from the left side menu, then select "Data sources", then select the data source you want to configure (InfluxDB or InfluxDB2).
   For the "URL" field of the data source, change the existing IP (9.46.81.11) to the IP machine that runs InfluxDB (this was determined in Step 1 using `ifconfig`).

   ![grafana-source](DocImages/GrafanaSource.png)

   Then press "Save & Test" (at the bottom of the screen)to validate the connection.
   Make sure you change both data sources.


9. Display the pre-loaded dashboard in grafana UI

   From the left side menu select the "Dashboard" icon (4 squares), then select "Browse", then select "JMeter Load Test".
   Enable automatic refresh of the graphs by selecting `10 sec` in the top right menu of grafana dashboard.


10. Deploy the services in OCP

    1. Go to the Knative directory:
       ```
       cd Knative
       ```

    2. Validate that yaml files have the correct images specified:
       ```
       grep "image:" *.yaml
       ```
       The image should start with `image-registry.openshift-image-registry.svc:5000/` followed by the name of the project where the images were pushed (`default`) and followed by the image name and tag.

    3. Deploy mongodb:

      > **NOTE**: If you are working on a cluster that is shared with others, please ensure that you are using a unique project name. We recommend using the format sccproject- followed by your initials. For example, sccproject-rm.

       ```
       kubectl apply -f Mongo.yaml
       ```
       and validate that the pod is running with:
       ```
       kubectl get pods | grep mongodb
       ```

    4. Restore the mongo database:
       ```
       ./mongoRestore.sh
       ```

    5. Deploy Semeru Cloud Compiler:
       ```
       kubectl apply -f JITServer.yaml
       ```
       and verify it started sucessfully with:
       ```
       kubectl get pods | grep jitserver
       ```

    6. Deploy the default AcmeAir instance:

      **IMPORTANT**: Please ensure to fill in all [Your initial] fields with the namespace used in the creation step above before proceeding to apply the YAML file.

       ```
       kubectl apply -f AcmeAirKN_default.yaml
       ```
       A message should appear in the console saying that the service was created.

    7. Deploy the AcmeAir instance with Semeru Cloud Compiler:

      **IMPORTANT**: Please ensure to fill in all [Your initial] fields with the namespace used in the creation step above before proceeding to apply the YAML file.

       ```
       kubectl apply -f AcmeAirKN_SCC.yaml
       ```
       Note: if you want to deploy the AcmeAir instance with Semeru Cloud Compiler and InstantON instead, then follow these steps:

       1. Edit the KNative permissions to allow to add Capabilities (if not already done)

          To confirm whether the `containerspec-addcapabilities` is enabled, you can inspect the current configuration of `config-features` by executing the command 

            > ```bash 
            > kubectl -n knative-serving get cm config-features -oyaml | grep -c "kubernetes.containerspec-addcapabilities: enabled" && echo "true" || echo "false"
            > ```

            > **IMPORTANT**: If the command returns true, it indicates that the Knative 'containerspec-addcapabilities' feature is already enabled. Please skip the step regarding editing Knative permissions. However, if it returns false, please proceed with the subsequent step to enable the feature.

            >>  ### Edit the Knative permissions to allow to the ability to add Capabilities

            >>  ```bash
            >>  kubectl -n knative-serving edit cm config-features -oyaml
            >>  ```

            >>  Add in the following line just bellow the “data” tag at the top:
            >>  ```yaml
            >>  kubernetes.containerspec-addcapabilities: enabled
            >>  ```

            >> **IMPORTANT**: to save your change and exit the file, hit the escape key, then type `:x`. If you received the message `Edit cancelled, no changes made`, it may indicate that the Knative service has already been edited and the same changes applied. 

       2. Create a Service Account named `instanton-sa-[Your Initial]`:
          ```
          oc create serviceaccount instanton-sa-[Your Initial]
          ```

       3. Create a Security Context Constraint named `cap-cr-scc`:
          ```
          oc apply -f scc-cap-cr.yaml
          ```

       4. Add the `instanton-sa` Service Account to the `cap-cr-scc` Security Context Constraint:
          ```
          oc adm policy add-scc-to-user cap-cr-scc -z instanton-sa-[Your Initial]
          ```

       5. Deploy the AcmeAir instance with Semeru Cloud Compiler and InstantON:
            
          **IMPORTANT**: Please ensure to fill in all [Your initial] fields with the namespace used in the creation step above before proceeding to apply the YAML file.

          ```
          kubectl apply -f AcmeAirKN_SCC_InstantON.yaml
          ```

    8. Verify that 4 pods are running:
       ```
       kubectl get pods
       ```
       Note: Knative will terminate the AcmeAir pods automatically after about two minutes of inactivity. This does not affect the experiment.

11. Apply external load
    1. Find the external address of the two AcmeAir services. Use
       ```
       kubectl get all | grep http
       ```
       and extract the part that comes after "http://" or "https://" and use it as the address of the service.
       It should be something like `acmeair-baseline-default.apps.ocp.ibm.edu` and `acmeair-scc-default.apps.ocp.ibm.edu` (or `acmeair-sccio-default.apps.ocp.ibm.edu` for the service with InstantON)
       The same information can be obtained with `kn` if installed:
       ```
       kn service list
       ```

    2. Verify that the `runJMeter.sh` script contains these service addresses for the JHOST environment variable passed to the JMeter containers:

       **IMPORTANT**: Please ensure to fill in all fields marked as [Your initial] with the namespace used in the creation step above, and fill in all fields marked as [OCP server name] with the OCP server address you created with TechZone (see the example in the comments mentioned in the runJMeter.sh file) before proceeding to run the runJMeter.sh file.

       ```
       cat runJMeter.sh | grep JHOST
       ```
       **Note**: if you selected to start the `AcmeAirKN_SCC_InstantON` service instead of `AcmeAirKN_SCC`, then edit `runJMeter.sh` to comment out the second container invocation (the one with JHOST="acmeair-scc-default.apps.ocp.ibm.edu") and remove the comment from the third container invocation (the one with JHOST="acmeair-sccio-default.apps.ocp.ibm.edu").

    3. Launch jmeter containers:
       ```
       ./runJMeter.sh
       ```
       This will launch two JMeter containers that will generate load for the two AcmeAir services.

       After 10 seconds or so, validate that there are no errors with
       ```
       podman logs jmeter1
       ```
       ```
       podman logs jmeter2
       ```

    4. Go to the grafana dashboard you configured before and watch the throughput results for the two services.

       The AcmeAirEE8 service with Semeru Cloud Compiler should reach peak throughput much faster than the baseline service.


12. Cleanup
    1. Delete the services you created:
       ```
       kubectl delete -f AcmeAirKN_default.yaml
       kubectl delete -f AcmeAirKN_SCC.yaml
       kubectl delete -f AcmeAirKN_SCC_InstantON.yaml
       kubectl delete -f JITServer.yaml
       kubectl delete -f Mongo.yaml
       ```
    2. Stop the grafana and influxdb containers:
       ```
       podman stop grafana
       podman stop influxdb
       ```