#### RHEL Skopeo Jenkins Slave

`Skopeo` - A command utility for various operations on container images and image repositories.

https://github.com/projectatomic/skopeo/

#### Build image

We extend the `jenkins-slave-base-rhel7` image and add `skopeo` CLI.

```
-- build from Dockerfile

oc project openshift
oc new-build --name=jenkins-skopeo-slave --strategy=docker --binary
oc start-build jenkins-skopeo-slave --from-file=. --follow

-- build from a template

oc process -f ./jenkins-slave-image-mgmt-template.yml | oc apply -n openshift -f-
oc new-app --template=openshift/jenkins-slave-image-mgmt -n ci-cd
```

The template is maintained upstream here:

https://github.com/redhat-cop/containers-quickstarts/tree/master/jenkins-slaves/jenkins-slave-image-mgmt

#### Handling Proxies

Note that the RHEL Dockerfile unsets any proxies that may be configured in the build defaults, as they can interfere with with JNLP communications to jenkins jnlp service.

```
ENV HTTP_PROXY='' http_proxy='' HTTPS_PROXY='' https_proxy='' NO_PROXY='' no_proxy=''
```

#### Use within Jenkins

For existing Jenkins servers, the slave can be added by using the following steps.

1. Login to Jenkins
2. Click on Manage Jenkins and then Configure System
3. Under the Cloud section, locate the Kubernetes Plugin. Click the   Add Kubernetes Pod Template
4. Enter the following details
   - Name: jenkins-skopeo-slave
   - Labels: jenkins-skopeo-slave
   - Timeout in seconds for Jenkins connection: 100   

Using the oc command line, run 
```
oc get is jenkins-skopeo-slave -n openshift --template='{{ .status.dockerImageRepository }}'
A value similar to 172.30.1.1:5000/openshift/jenkins-skopeo-slave should be used.
```

   
5. Add Container Template   
   - Name: jnlp
   - Docker image: 172.30.1.1:5000/openshift/jenkins-skopeo-slave:latest
   - Always pull image: (tick yes)
   - Working directory: /tmp
   - Arguments to pass to the command: ${computer.jnlpmac} ${computer.name}
   - Allocate pseudo-TTY: (deselect)
6. Click Save to apply the changes

Note: For pipeline developers, you can also configure the agent `kubernetes` plugin for in your individual pipeline, but we prefer to centralise this e.g.

```
    agent {
              kubernetes {
                label 'promotion-slave'
                cloud 'openshift'
                serviceAccount 'jenkins'
                containerTemplate {
                  name 'jnlp'
                  image "docker-registry.default.svc:5000/${NAMESPACE}/jenkins-slave-image-mgmt:latest"
                  alwaysPullImage true
                  workingDir '/tmp'
                  args '${computer.jnlpmac} ${computer.name}'
                  command ''
                  ttyEnabled false
                }
              }
    }
```

#### Use with Jenkins pipeline

```
node('jenkins-skopeo-slave') { 

  stage('Inspect Image') {
    sh """
    set +x       
    skopeo inspect docker://docker.io/fedora
    """
  }
}
```

```
// call like this so we dont print credentials to logs
sh "skopeo --debug copy --src-tls-verify=false --dest-tls-verify=false --src-creds=${SRC_CREDS} --dest-creds=${DEST_CREDS} ${SRC_REGISTRY}/${SRC_PROJECT}/${APP_NAME}:${SRC_TAG} ${DEST_REGISTRY}/${DEST_IMAGE_PROJECT}/${APP_NAME}:${DEST_DARK_TAG}"
```

##### Create Service Account for Clusters

To promote images from one cluster to another, it is convenient to create service accounts in the source project and destination project using least privilege:

```
-- source cluster project
oc project ci-cd
oc create sa skopeo
oc adm policy add-cluster-role-to-user system:image-builder system:serviceaccount:$(oc project -q):skopeo
oc serviceaccounts get-token skopeo

-- destination cluster project
oc project openshift
oc create sa skopeo
oc adm policy add-cluster-role-to-user system:image-builder system:serviceaccount:$(oc project -q):skopeo
oc adm policy add-role-to-user edit system:serviceaccount:$(oc project -q):skopeo
oc serviceaccounts get-token skopeo
```

We can use the `TOKENS`, store them in Secrets or Jenkins Credential Bindings and use them when promoting images e.g.

```
skopeo --debug copy --src-tls-verify=false --dest-tls-verify=false --src-creds="${SRC_CREDS}" --dest-creds="${DEST_CREDS}" "${srcPullSpec}" "${destPullSpec}"
```

#### Confingure Jenkins Slaves

Auto-configure our jenkins slave pods using configmap

```
oc create -f JenkinsSlaveConfigMap.yml -n cicd --dry-run -o yaml | oc apply -n cicd  --force -f-
```
