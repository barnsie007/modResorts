# How to deploy your applications using WebSphere Base in Containers

Additional documentation and information can be found [here](https://github.com/WASdev/ci.docker.websphere-traditional#readme)

## A binary project
The following section describes how to deploy a binary based project to Red Hat&trade; OpenShift. For the binary project, you will provide binary files for the application and any dependencies.

## Generated Artifacts
| Artifact | Description |
| --- | --- |
| Dockerfile | Dockerfile to build the application container image |
| src/config/install_app.py | Jython script to install the application |
| src/config/server_config.py | server configuration Jython script file |
| lib/ | Directory for external library dependencies, such as database driver library |
| deploy/RCO/ | Directory for the application deployment manifest files using RCO operator|

You can migrate your application in three steps. This document will reference the following variables:

- `<BINARY_FILE_LOCATION>` is the location of the binary files for your application, including any dependencies and shared library files.
- `<APPLICATION_FILE>` is the binary file for your application.
- `<DEPENDENCY_FILE_LOCATION>` is the location of any dependencies and shared library files that your application requires.
- `<APP_CONTEXT_ROOT>` is the context root for your application. If not defined elsewhere, for example in ibm-web-ext-xml.
- `<APPLICATION_NAME>` is the name of the application.
- `<CONTAINER_ID>` is the container ID for your docker image. To get this value, enter: docker ps
- `<IMAGE_REFERENCE>` is the reference for this image, including the registry, repository and tag, e.g. docker.io/myspace/myappimage:1.0.0
- `<MIGRATION_ARTIFACTS_HOME>` is the location where you have unzipped the Transformation Advisor artifacts, or cloned the repository.
- `<OCP_PROJECT>` is the name of the OpenShift project where you want to install the application.

### Notes

- The generated `src/config/server_config.py` script file may contain password variable but its value has been stripped off from
the original source environment.  You need to provide it manually. For example,

  ```
  # The following variables are used to replace sensitive data in the configuration for the application.
  # The values for these variables were not collected because the includeSensitiveData option was not specified.
  # ============================================================
  myhost_a1CellManager01_db2_password_1=''
  # ============================================================
  ```

## Deploy to Red Hat OpenShift Container Platform using Runtime Component Operator


### STEP ONE: build tWAS docker image with Java application
In this step you will containerize your tWAS installation. You will create a tWAS image that has your migrated application installed and working, and then test the image to confirm that it is operating correctly.

`NOTE: If you are using podman instead of docker simply replace the word docker in each command with podman`

### Step 1 Prerequisites
- Docker or podman is installed.
    - Download Docker: https://www.docker.com/get-started
    - Download podman: https://podman.io/getting-started/installation
- The machine where you complete this task requires access to the internet to download the tWAS base image.
- If using docker ensure the docker service is running. If itâs not, start it:
```
service docker start
```

### Step 1 Tasks
1. Add your application file to the migration bundle, and remove the placeholder file:
```
cp <BINARY_FILE_LOCATION>/<APPLICATION_FILE> <MIGRATION_ARTIFACTS_HOME>/target/
rm <MIGRATION_ARTIFACTS_HOME>/target/*.placeholder
```
2. Add your dependency file(s) to the migration bundle, and remove the placeholder file(s):
```
cp <DEPENDENCY_FILE_LOCATION>/* <MIGRATION_ARTIFACTS_HOME>/lib
rm <MIGRATION_ARTIFACTS_HOME>/lib/*.placeholder
```
     NOTE: The placeholder files provide the name(s) of all depednecies for your application

NOTE: The migration artifacts assist you in the migration of your application. Depending on the nature and complexity of your application, additional configuration may be required to fully complete this task. For example, you may need to complete extra configuration to connect your application with a user registry, and or to configure user security role bindings. Consult the product documentation for more details.

3. Go to where your migration artifacts are located and build your image from the docker file:
```
cd <MIGRATION_ARTIFACTS_HOME>
docker build --no-cache -t "<IMAGE_REFERENCE>" .
```
Note: The migration bundle includes a pom.xml file that allows the application and all dependencies to be pulled from Maven.  You need to enable this option in the Dockerfile and update the placeholders in the pom.xml file with correct values.
The details of how to do this can be found in the '[Add dependencies using Maven](#add-dependencies-using-maven)' section

4. Run the image and confirm that it is working correctly:
```
docker run âp 9080:9080 <IMAGE_REFERENCE>
```
5. If everything looks good, the image has been started and mapped to the port 9080. You can access it from your browser with this link:
   `http://<CURRENT_MACHINE_IP>:9080/<APP_CONTEXT_ROOT>`
   **Optional:** Check your image when it is up and running by logging into the container:
```
docker exec âti <CONTAINER_ID> bash
```
This allows you to browse the file system of the container where your application is running.


### STEP TWO: Deploy your image to Red Hat OpenShift
In this step you will deploy the image you have created to Red Hat OpenShift. These instructions relate to OpenShift 4+ and have been validated on OpenShift 4.10.

### Step 2 Prerequisites

- Access to either a public or private Red Hat OpenShift 4+ environment.
- Image registry access
    - You will need push your migrated application image to a location that is accessible to the OpenShift cluster. You may use a publicly available registry (e.g. Dockerhub), or your own private registry.
        The migration artifacts generated by Transformation Advisor (specifically the application-cr.yaml file) need to be updated with the image reference that you are using.
        If you do not have a suitable registry available you can create your own in Dockerhub or Podman to use until a suitable registry is found.
      - How to create your own registry in Dockerhub can be found [here](https://www.docker.com/blog/how-to-use-your-own-registry-2/)
      - How to create your own registry in Podman can be found [here](https://www.redhat.com/sysadmin/simple-container-registry)
- A copy of the migration artifacts that you downloaded from Transformation Advisor available on the machine.
- Runtime Component Operator
    - In order to deploy your application on tWAS image on OpenShift, you must first have installed the Runtime Component operator on your cluster.
        - The Runtime Component Operator is available in the Application Runtime Catalog. Go to the Operators...OperatorHub UI and click the **Application Runtime Catalog**. You can use the search to quickly find the Runtime Component operator.
        - Click on the Runtime Component operator UI and follow the instructions to install.
        - You will be given the option of installing the operator cluster wide which will be capable of installing Runtime Component instances across all namespaces, OR, you can choose to install the operator in a given namespace. Choose either option depending on your preference.
        - For full details on the Runtime Component operator, please consult the documentation [here](https://github.com/application-stacks/runtime-component-operator/tree/main/doc/).

### Step 2 Tasks
1. Push your tWAS image to a registry that is accessible to the OpenShift cluster. e.g.
    ```
    docker push <IMAGE_REFERENCE>
    ```
   where `<IMAGE_REFERENCE>` is the image reference you tagged the image with when building, e.g. `docker.io/myspace/myappimage:1.0.0`

   Note:
    - You may need to log in to that image registry in order to successfully push.
    - If your registry requires authenticated pulls, you will need to set up your cluster with a pull secret. See [docs](https://docs.openshift.com/container-platform/4.6/openshift_images/managing_images/using-image-pull-secrets.html) for more information.


2. Log in to the OpenShift cluster and create a new project in OpenShift.

    ```
    oc login -u <USER_NAME>
    ```

   If you have not already created a namespace when installing the Liberty operator create one now.
    ```
    oc new-project <OCP_PROJECT>
    ```

3. Deploy your image with its accompanying operator using the following instructions:

   a. Change to the deploy directory in `<MIGRATION_ARTIFACTS_HOME>`:

    ```
    cd <MIGRATION_ARTIFACTS_HOME>/deploy/RCO
    ```

   b. Update the IMAGE_REFERENCE in the `deployment.yaml` file to match the application image reference you pushed in Task 1.

   c. Create the Runtime Component instance for your tWAS image using the provided artifact:
    ```
    oc apply -f deployment.yaml
    ```

   d. You can view the status of your deployment by running oc get deployments. If you donât see the status after a few minutes, query the pods and then fetch the tWAS pod logs:
    ```
    oc get pods
    oc logs <pod>
    ```

   e. You can now access your application by navigating to the `Networking...Routes` UI in OpenShift and selecting the namespace where you deployed.
   &nbsp;
4. **Optional:** If you wish to delete your tWAS instance from OpenShift, you can uninstall using the OpenShift UI. Navigate to the `Operators...Installed operators` in OpenShift to Find the Runtime Component operator. Click on the `All instances` tab to find your tWAS instance and uninstall as required.

## Add dependencies using Maven

### Overview
The migration bundle includes a pom.xml file that allows the application and all dependencies to be pulled from Maven.  You need to enable this option in the Dockerfile and update the placeholders in the pom.xml file with correct values.


### Tasks
1. Navigate to the migration bundle
   ```
    cd <MIGRATION_ARTIFACTS_HOME>
    ```
2. Edit the pom.xml file and update the placeholder values with the appropriate values
    1. Set the correct values for the ```<dependency>``` element
    2. Set the correct values for each of the ```<artifactItem>``` elements
       &nbsp;
3. Edit the Dockerfile so that dependencies are pulled in during the image build
    1. Uncomment the line ```RUN mvn process-resources ```


# Managing keystores during deployment
```
When deploying to OpenShift, if your application is configured to use non default keystores then the default route will not work without configuration changes
```
## Configuring keystores in a container
_NOTE: If you plan to deploy to OpenShift you can use the Operator to generate the necessary certificates, see the next section for details._

Transformation Advisor does not automatically migrate keystore information in the migration bundle.
When running in a container using the default Dockerfile produced by Transformation Advisor the Liberty server will output messages indicating that the non-default keystore files can not be found.

## Configuring keystores in an OpenShift deployment
When deploying to OpenShift you can use the Liberty Operator to generate all necessary certificates or use your own certificates.


### References

https://github.com/WASdev/ci.docker.websphere-traditional#readme

https://github.com/WASdev/ci.docker.websphere-traditional/tree/main/samples/hello-world/openshift
