# Spinnaker-Chaos-Templates

Consists of [custom (kubernetes) jobs](https://spinnaker.io/guides/operator/custom-job-stages/#custom-job-stages) to implement predefined chaos stages in a spinnaker pipeline. 
The jobs are expressed as individual `orca-local.yml` definitions that can be used as is. 

These kubernetes jobs make use of the [chaos-ci-lib](https://github.com/mayadata-io/chaos-ci-lib) docker image and executes a specific chaos experiment,
The job essentially implements a wrapper logic that does the following actions:

- Pulls the chaosexperiment manifest from [chaos-charts](https://github.com/litmuschaos/chaos-charts) 
- Modifies the chaosengine with tunables supplied to the run job as environment variables
- Launches the experiment by creating the chaosengine, monitors experiment completion & examines the verdict

The above actions are performed as a BDD test implemented using the popular Ginkgo, Gomega test framework

## Sample Custom Job

The custom jobs are defined within the `orca-local.yaml` file, that serves as a profile which will be read by the `halyard` microservice, thereby ensuring the predefined spinnaker 
stage is available while creating new pipelines. It describes the metadata of the stage along with tunables that will be propagated to the kubernetes job. 

An example template running a custom job to perform the `generic/pod-delete` chaos experiment is provided below: 

```yaml
job:
  preconfigured:
    kubernetes:
      - label: Pod-Delete-Chaos
        type: chaosExperiment
        description: Kill an application pod
        cloudProvider: kubernetes
        account: spinnaker
        credentials: spinnaker
        application: hello
        waitForCompletion: true
        parameters:
          - name: APPLICATION_NAMESPACE
            label: Namespace of AUT
            description: Namespace where chaos will occur
            mapping: manifest.spec.template.spec.containers[0].env[0].value
            defaultValue: "spinnaker"
          - name: APPLICATION_LABEL
            label: Label of AUT
            description: Label by which app is filtered
            mapping: manifest.spec.template.spec.containers[0].env[1].value
            defaultValue: "name=hello"
        manifest:
          apiVersion: batch/v1
          kind: Job
          metadata:
            name: run-pod-delete-chaos
            namespace: spinnaker
          spec:
            backoffLimit: 0
            template:
              spec:
                restartPolicy: Never
                containers: 
                  - name: run-pod-delete-chaos
                    image: mayadata-io/chaos-ci-lib:latest
                    env:
                      - name: APP_NS
                        value: $(APPLICATION_NAMESPACE)
                      - name: APP_LABEL
                        value: "$(APPLICATION_LABEL)"
                    command: ["/bin/bash"]
                    args:
                    - -c 
                    - ./pod-delete
```

## Updating the Custom Job Template

- Exec inside the halyard pod  
- Update the orca-local.yml (if present) template within `/home/spinnaker/.hal/default/profiles` with the new custom job. Else create newly 
- Update the service by running `hal deploy apply`. This will restart the spin-orca microservices pod to read the latest orca profile


Post this, the experiment-specific stage will appear in the list/drop-down on `Deck` (spinnaker UI)

 


 
