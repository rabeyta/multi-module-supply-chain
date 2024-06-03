# multi-module-supply-chain

Purpose of this repo is to demonstrate how we can support a java mono repo utilizing gradle within [Tanzu Application Platform](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/index.html).

Verified on [v1.10.0](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.10/tap/overview.html)

The components involved are

* Cartographer Supply Chain 
  * simple supply chain that pulls source, determines if it should be built or not, tests and then builds an image
* Cartographer `ClusterSourceTemplate`
  * determines if the url and revision returned from the `source-provider` `ClusterSourceTemplate` should be built or ignored
  * output of this `ClusterSourceTemplate` will be used as input for the `source-tester` stage
* Tekton Task
  * performs the actual logic to determine if the workload should be built
* Tekton Pipeline 
  * runs the tests for the given module to be built
* Cartographer workload
  * defines the repo, branch and module to build along with other application specific configuration

# workload config

## build env

```yaml
spec:
  build:
    env:
      - name: BP_NATIVE_IMAGE
        value: "false"
      - name: BP_JVM_VERSION
        value: "17"
      - name: BP_GRADLE_BUILD_ARGUMENTS
        value: "--no-daemon -Dorg.gradle.welcome=never :tanzu-java-web-app-1:assemble"
      - name: BP_GRADLE_BUILT_MODULE
        value: "tanzu-java-web-app-1"
```
`BP_GRADLE_BUILD_ARGUMENTS` to instruct the build pack to run the assemble task for the specific module this workload defines.

`BP_GRADLE_BUILT_MODULE`

