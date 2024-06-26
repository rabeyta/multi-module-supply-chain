# multi-module-supply-chain

Purpose of this repo is to demonstrate how we can optimize building a java mono repo utilizing gradle within [Tanzu Application Platform](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/index.html) supply chain v1 ([Cartographer](https://cartographer.sh/) based).

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

# repo

We need to update the multimodule project with a common task that can be called from the `AffectedModuleDetector` plugin to allow us to identify if a given module needs built or not. 

This Gradle task will be called from the Tekton task `detect-java-module` and it will allow the task to determine if it needs to return the new revision (impacted by new commit) or a prior one (not impacted by new commit)

## affected module detector config

Example `AffectedModuleDetector` config that defines a common task that can called for each affected modules when running

```groovy
affectedModuleDetector {
    baseDir = "${project.rootDir}"
    pathsAffectingAllModules = setOf("gradle/", "config/")
    logFilename = "output.log"
    logFolder = "${project.rootDir}/affectedModuleDetector"
    specifiedBranch = "main"
    compareFrom = "PreviousCommit" //default is PreviousCommit
    ignoredFiles = setOf(".*\\.md", ".*\\.txt", ".*README")
    includeUncommitted = true
    customTasks = setOf(
            AffectedModuleConfiguration.CustomTask("printProjectsImpacted", "printProjectName", "print name")
    )
}
```

## gradle task for each project
example task for a given module
``` groovy
tasks.register("printProjectName") {
    doLast {
        println("app-1")
    }
}
```

## gradle command to determine affected modules

```shell
gradle printProjectsImpacted -Paffected_module_detector.enable --no-daemon
```

# workload

## supply chain labels

```yaml
apiVersion: carto.run/v1alpha1
kind: Workload
metadata:
  name: tanzu-java-web-app-1
  labels:
    apps.tanzu.vmware.com/multi-module: "true"
    apps.tanzu.vmware.com/has-tests: "true"
    apps.tanzu.vmware.com/workload-type: web
```

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

the `module` param will be read in the [testing pipeline](./testing-pipelines/java-test.yaml) in addition to the module detector to instruct these components which gradle module it needs to focus on

# test pipeline

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

# detect module

## supply-chain 

Within the [supply chain](./supply-chain/multi-module-detect-sc.yaml), the module-detector `ClusterSourceTemplate` sits inbetween the source-provider and source-tester to act as a gatekeeper for changes.

The source-provider will emit url and revision for every commit of a given repository being watched.

The module-detector determines if the commit impacts this workload, and if it does, it outputs the input url and revision for the source-tester to test. 

If the commit does not impact the module, the prior successful output from source-tester is output from module-detector and no further work is done based on this commit. 

```yaml
  resources:
    - name: source-provider
      templateRef:
        kind: ClusterSourceTemplate
        name: source-template
      params:
        - name: serviceAccount
          default: #@ data.values.service_account
        - name: gitImplementation
          default: #@ data.values.git_implementation

    - name: module-detector
      templateRef:
        kind: ClusterSourceTemplate
        name: java-module-detector
      sources:
        - resource: source-provider
          name: source

    - name: source-tester
      templateRef:
        kind: ClusterSourceTemplate
        name: testing-pipeline
      sources:
        - resource: module-detector
          name: source
```

## module-detector-source-template

The goal of this [ClusterSourceTemplate](./templates/module-detector-source-template.yaml) is to produce a Tekton TaskRun that will output url and revision to be output from this template

```yaml
apiVersion: carto.run/v1alpha1
kind: ClusterSourceTemplate
metadata:
  name: java-module-detector
spec:
  urlPath: .status.results[?(@.name=="url")].value
  revisionPath: .status.results[?(@.name=="revision")].value

```

The template will gather inputs from the workload to build the params for the TaskRun and then create the CR


```yaml

#@ def merged_tekton_params():
#@   params = []
#@   if hasattr(data.values, "params") and hasattr(data.values.params, "testing_pipeline_params"):
#@     for param in data.values.params["testing_pipeline_params"]:
#@       params.append({ "name": param, "value": data.values.params["testing_pipeline_params"][param] })
#@     end
#@   end
#@   resources = data.values.workload.status.resources
#@   priorUrl = ""
#@   priorRevision = ""
#@   if len(resources) > 2 and hasattr(resources[2], "outputs"):
#@     priorUrl = resources[2].outputs[0].preview
#@     priorRevision = resources[2].outputs[1].preview
#@   end
#@   params.append({ "name": "source-url", "value": data.values.sources.source.url })
#@   params.append({ "name": "source-revision", "value": data.values.sources.source.revision })
#@   params.append({ "name": "prior-url", "value": priorUrl })
#@   params.append({ "name": "prior-revision", "value": priorRevision })
#@   params.append({ "name": "git-url", "value": data.values.workload.spec.source.git.url })
#@   params.append({ "name": "git-branch", "value": data.values.workload.spec.source.git.ref.branch })
#@   return params
#@ end

apiVersion: tekton.dev/v1
kind: TaskRun
metadata:
  generateName: #@ data.values.workload.metadata.name + "-module-detector-"
  labels: #@ merge_labels({ "app.kubernetes.io/component": "module-detector" })
spec:
  #@ if/end hasattr(data.values.workload.spec, "serviceAccountName"):
  serviceAccountName: #@ data.values.workload.spec.serviceAccountName
  params: #@ merged_tekton_params()
  taskRef:
    resolver: cluster
    params:
      - name: kind
        value: task
      - name: namespace
        value: tap-tasks
      - name: name
        value: detect-java-module
```

## detect-java-module-task

[module-detector-task](./tasks/module-detector-task.yaml)
```yaml
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: detect-java-module
  namespace: tap-tasks
spec:
  params:
    - name: source-url
    - name: source-revision
    - name: prior-url
    - name: prior-revision
    - name: git-url
    - name: git-branch
    - name: module
  results:
    - name: url
    - name: revision
```

The task will perform the following logic on the inputs.

1. if there is no prior information, it returns the input source-url and revision as this is a new run with no history
2. if the source and prior revisions are the same, source revision and url is returned with no further work needs done as this should be a no-op
3. clone the input repository url and checkout the input branch
4. run the gradle command to identify which modules are affected by the latest commit
5. if the input module is in the output of affected modules, the input source-url and revision are output as results. otherwise the prior url and revision are returned to enable no further work to be completed

# tap-gui

Output revision is the same for source provider, module detector and source tester when the commit should be acted upon
![tap-gui commit is built](docs/tap-gui-commit-is-built.png)

Output revision is not the same for source provider and module detector as the commit should not have been acted upon
![tap-gui commit is not built.png](docs/tap-gui_commit_is_not_built.png)

# infographic
![info-graphic.png](docs/info-graphic.png)
