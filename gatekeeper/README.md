# Gatekeeper

We only need to modify the pods that only contains one container whose name is `jnlp`.
Generally these pod is configured in Jenkins Cloud configurations.

If the pod is created by [Jenkinsfile](https://plugins.jenkins.io/kubernetes/#plugin-content-declarative-pipeline), the `jnlp` is decoupled already.

The full explanation can be found in [main README](../README.md).
