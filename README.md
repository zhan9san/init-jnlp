# init-jnlp

This is a workaround of [Jenkins issue: Default container selector for the template configuration in the UI](https://issues.jenkins.io/browse/JENKINS-64778)

[It should be noted that the main reason to use the global pod template definition is to migrate a huge corpus of existing projects (including freestyle) to run on Kubernetes without changing job definitions.](https://plugins.jenkins.io/kubernetes/#plugin-content-using-a-label)

As there is no way to change the default container to non-jnlp container if we
use Kubernetes plugin with `label`, we have do some magics in the jnlp
container.

The magic is that the Jenkins-related artifacts are downloaded in a shared
volume in the init container, like JDK, agent.jar and jenkins-agent, and then the
entrypoint in jnlp container is overriden by `jenkins-agent` in the shared
volume.

## How to introduce the magic

It is recommended to replace all download URLs to your internal URLs.

The Kubernetes plugin would inject some labels to pod template automatically.

```yaml
metadata:
  labels:
    jenkins: "slave"
    jenkins/label-digest: "590267eadb8beeaff73d0dee815c30ea86815a9c"
    jenkins/label: "helm"
```

Original yaml

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: jnlp
      image: customized-python-jnlp:3.12
      command:
        - /foo/bar
```

New yaml

You can modify the yaml configuration in Jenkins Cloud configuration manually.
There may be hundreds of templates in Jenkins Cloud configuration.

Or you can modify the yaml configurations [by gatekeeper](./gatekeeper/).

```yaml
apiVersion: v1
kind: Pod
spec:
  volumes:
    - name: shared-volume
      emptyDir: {}
  initContainers:
    - name: init
      image: busybox
      command:
        - /bin/sh
        - -c
        - wget -O /shared/init-jnlp https://raw.githubusercontent.com/zhan9san/init-jnlp/main/init-jnlp
      volumeMounts:
        - name: shared-volume
          mountPath: /shared
  containers:
    - name: jnlp
      image: customized-python-jnlp:3.12
      volumeMounts:
        - name: shared-volume
          mountPath: /shared
      command:
        - /shared/init-jnlp
```

### Pros

Decouple the business-related packages and the jenkins-related packages in
combined jnlp image.
If the Jenkins is upgraded, the JDK/agent version may be not supported in
original image. If they are decoupled, there is no need to rebuild the
business-related image.

### Cons

Each time a pod is created, the JDK/agent.jar need to be downloaded.

## Reference

* docker-agent: https://github.com/jenkinsci/docker-agent
* jenkins-agent: https://github.com/jenkinsci/docker-agent/blob/master/jenkins-agent
* jenkins-agent Dockerfile: https://github.com/jenkinsci/docker-agent/blob/master/debian/Dockerfile
