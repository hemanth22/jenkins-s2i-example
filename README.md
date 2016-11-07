# Jenkins S2I Example

An example for OpenShift, demonstrating Jenkins S2I features for installing plugins, configuring jobs, Jenkins, etc and using slave pods for Jenkins jobs.

This example uses:

- An ephemeral (i.e. non-persistent) Jenkins, using the `jenkins` ImageStream that is supplied with OpenShift v3.
    - Jenkins [pipeline-utility-steps plugin](https://github.com/jenkinsci/pipeline-utility-steps-plugin), to be able to read a Maven POM within a `Jenkinsfile`
- The `os-sample-java-web` sample OpenShift app, from the [OpenShiftDemos repository](https://github.com/OpenShiftDemos), to demonstrate building a simple Java (Tomcat) web application.

## About this fork

This fork of [Siamak Sadeghianfar's original repository](https://github.com/siamaksade/jenkins-s2i-example) (thanks Siamak!) has the following differences:

- The original `jdk8` slave image is moved to `slaves/jdk8` and now uses the `centos` base image, which provides a workaround for the original base `rhel` image throwing the error on build `nfs_wrapper is not installed"`
- Extra slave image, see below.

In `master`, the configuration of the Jenkins master defines the following Jenkins jobs:

- A Maven build, `tasks-build`
- A pipeline build, `os-sample`, using a `Jenkinsfile` in the application's source repository, which demonstrates a Maven build and deployment to an artifact repository (Nexus).

In `slave`, the following Jenkins slave images are defined:

- `jdk8`: the slave image included in the original repository
- `fabric8`: uses the [fabric8 maven-builder](https://github.com/fabric8io/maven-builder) builder image, for Java Maven builds.

### Extending

Extend this repo further by:

- Creating further Jenkins slave images in the `slaves` directory, depending on your requirements (e.g. for building NodeJS)
- Defining new Jenkins Jobs in the `master/configuration/jobs` directory. These Jobs will be added to Jenkins on startup.

## installation

1. create a new openshift project, where the jenkins server will run:

  ```
  $ oc new-project ci --display-name="CI/CD"
  ```

2. Give the Jenkins Pod service account rights to do API calls to OpenShift. This allows us to do the Jenkins Slave image discovery automatically.

  ```
  $ oc policy add-role-to-user edit -z default -n ci
  ```

3. Install the provided OpenShift templates:

  ```
  $ oc create -f https://raw.githubusercontent.com/monodot/jenkins-s2i-example/master/jenkins-slave-builder-template.yaml
  $ oc create -f https://raw.githubusercontent.com/monodot/jenkins-s2i-example/master/jenkins-master-s2i-template.yaml
  ```

4. Build the `jdk8` Jenkins slave image (this is the default).

  ```
  $ oc new-app jenkins-slave-builder
  ```

5. Build the additional `fabric8/maven-builder` Jenkins slave image (note how the `IMAGE_NAME` of the builder image is passed as a parameter):

  ```
  $ oc new-app jenkins-slave-builder -p SLAVE_REPO_CONTEXTDIR=slaves/fabric8,SLAVE_LABEL=fabric8,IMAGE_NAME=fabric8/maven-builder
  ```

6. Create Jenkins master. You can customize the source repo and other configurations through template parameters. Note that this example doesn't define any [persistent volume](https://docs.openshift.com/enterprise/3.2/architecture/additional_concepts/storage.html). You need to define storage in order to retain Jenkins data on container restarts. 

  ```
  $ oc new-app jenkins-master-s2i
  ```

If you open the Jenkins admin console, you will see that two jobs have been defined, `os-sample` and `tasks-build`.

## To run the sample app build pipeline

This demonstration pipeline builds a Java app using Maven inside a Jenkins slave pod, deploys to Nexus, and then uses the [ews-bin-deploy](https://github.com/monodot/ews-bin-deploy) S2I builder to directly `curl` in the binary artifact (the WAR file) from Nexus into the Red Hat JBoss `jboss-webserver30-tomcat8-openshift` supported image.

**Before continuing**, you will need a running instance of Nexus in the `ci` project in OpenShift. However, if your instance of Nexus is running outside OpenShift, you can create an endpoint and service within the `ci` project which will proxy your Nexus and make it available to OpenShift at the address `http://nexus.ci.svc.cluster.local:8081/nexus`. To do this, see [this Gist](https://gist.github.com/monodot/1b59560df4c2e1882686eb91eb2ebfda).

Once Nexus is running, follow these instructions to build and deploy the app `os-sample-java-web` using a Jenkins pipeline:

1. In OpenShift, create a new project `build` to hold the build configuration:

  ```
  $ oc new-project build
  ```

2. Create Kubernetes resources for the sample application. This will automatically start a S2I build before we've even had chance to deploy our compiled artifact into Nexus, but don't worry about it:

  ```
  $ oc create -f https://raw.githubusercontent.com/monodot/os-sample-java-web/master/openshift-build.json
  $ oc new-app os-sample-java-web-template
  ```

3. Grant the default service account in the ci namespace access to the build namespace:

  ```
  $ oc policy add-role-to-user edit system:serviceaccount:ci:default -n build
  ```

4. Create a new project `run` to hold the deployment configuration:

  ```
  $ oc new-project run
  ```

5. Create the deployment resources for the sample app. This will create a new deploymentConfig that watches the ImageStreamTag `os-sample-java-web:dev-release`.

  ```
  $ oc create -f https://raw.githubusercontent.com/monodot/os-sample-java-web/master/openshift-run.json
  $ oc new-app os-sample-java-web
  ```

6. Finally, give the service account in the `run` namespace permission to pull images from the `build` namespace:

  ```
  $ oc policy add-role-to-user system:image-puller system:serviceaccount:run:default -n build
  ```

6. Now initiate a build from Jenkins by going to `http://jenkins-ci.rhel-cdk.10.1.2.2.xip.io/`, logging on, finding the included _os-sample_ project and clicking Build. The application will build in Maven and then deploy automatically into OpenShift in the `run` namespace.

### To specify another artifact repository for deployment

To override the URL for the deployment artifact repository:

1. In Jenkins, go to build > Configure.

2. Tick 'Prepare an environment for the run'

3. In _Properties Content_, add your URL into `MAVEN_OPTS`, e.g.:

  ```
  MAVEN_OPTS=-DaltDeploymentRepository=internalSnapshots::default::http://10.1.2.1:8081/nexus/content/repositories/snapshots/
  ```
