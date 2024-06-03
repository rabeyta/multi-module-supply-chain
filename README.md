# multi-module-supply-chain

Purpose of this repo is to demonstrate how we can support a java mono repo utilizing gradle within [Tanzu Application Platform](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/index.html) supply chain v1 ([Cartographer](https://cartographer.sh/) based).

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
* [AffectedModuleDetector](https://github.com/dropbox/AffectedModuleDetector) Gradle plugin to determine impacted modules from commits
* Gradle task that utilizes the `AffectedModuleDetector` plugin to identify impacted modules

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
`BP_GRADLE_BUILD_ARGUMENTS` to provide the build pack with commands to run in order to produce the artifact (jar) for the expected module associated with this workload configuration

`BP_GRADLE_BUILT_MODULE` instructs the build pack where to find the built artifact (jar)

## testing pipeline params

```yaml
spec:  
  params:
    - name: testing_pipeline_params
      value:
        module: "tanzu-java-web-app-1"
```

the `module` param will be read in the testing pipeline in addition to the module detector to instruct these components which gradle module it needs to focus on

# test logic

```yaml
spec:
  params:
    - name: source-url
    - name: source-revision
    - name: module
      default: ""
  tasks:
    - name: test
      params:
        - name: source-url
          value: $(params.source-url)
        - name: source-revision
          value: $(params.source-revision)
        - name: module
          value: $(params.module)
```

the module will be passed into the testing pipeline from the workload config and used to know which module to call the test task on. this enables the build to only run the specific module and not all modules with a given task.

sample script that handles maven and gradle with and without multiple modules.

```shell
if [ -f "mvnw" ]; then
    ./mvnw test
elif [ -f "build.gradle" ] || [ -f "build.gradle.kts" ]; then
    if [ ! -z "$(params.module)" ]; then
      gradle :$(params.module):test --no-daemon
    else
      gradle test --no-daemon
    fi
else
    echo "WARNING: No tests were run. This workload is not built with one of the currently supported frameworks (maven or gradle). If using another language/framework, update the image and the script sections of the 'pipeline.tekton.dev' resource in your namespace to match your language/framework."
fi
```


