# Jenkins S2I Example

An example demonstrating Jenkins S2I features for installing plugins, configuring jobs, Jenkins, etc and using slave pods for Jenkins jobs.

## About this fork

This is a fork of Siamak Sadeghianfar's original repository, with the following differences:

- It uses the `centos` base image, as a workaround for the original base `rhel` image throwing the error on build `nfs_wrapper is not installed"`
- A local standalone Nexus can be used as a Maven mirror to speed up builds (good for those long, cup-of-tea, toilet-break style Maven builds). Edit the file `slave/configuration/mirror-settings.xml` to suit your requirements. Then modify the build in Jenkins by going to the build > Configure > Build > Advanced > change Settings file path to `${JENKINS_HOME}/mirror-settings.xml`.

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
  $ oc create -f jenkins-slave-builder-template.yaml   # For converting any S2I to Jenkins slave
  $ oc create -f jenkins-master-s2i-template.yaml      # For creating pre-configured Jenkins master using Jenkins S2I
  ```

5. Build Jenkins slave image.

  ```
  $ oc new-app jenkins-slave-builder
  ```

4. Create Jenkins master. You can customize the source repo and other configurations through template parameters. Note that this example doesn't define any [persistent volume](https://docs.openshift.com/enterprise/3.2/architecture/additional_concepts/storage.html). You need to define storage in order to retain Jenkins data on container restarts. 

  ```
  $ oc new-app jenkins-master-s2i
  ```
