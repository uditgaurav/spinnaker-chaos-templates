# Spinnaker-Chaos-Templates

Consists of [custom (kubernetes) jobs](https://spinnaker.io/guides/operator/custom-job-stages/#custom-job-stages) to implement predefined chaos stages in a spinnaker pipeline. 
The jobs are expressed as individual `orca-local.yml` definitions that can be used as is. 

These kubernetes jobs make use of the [chaos-ci-lib](https://github.com/mayadata-io/chaos-ci-lib) docker image and executes a specific chaos experiment,
The job essentially implements a wrapper logic that does the following actions:

- Pulls the chaosexperiment manifest from [chaos-charts](https://github.com/litmuschaos/chaos-charts) 
- Modifies the chaosengine with tunables supplied to the run job as environment variables
- Launches the experiment by creating the chaosengine, monitors experiment completion & examines the verdict

The above actions are performed as a BDD test implemented using the popular Ginkgo, Gomega test framework

## Sample Chaos Custom Job

The custom jobs are defined within the `orca-local.yaml` file, that serves as a profile which will be read by the `halyard` microservice, thereby ensuring the predefined spinnaker 
stage is available while creating new pipelines. It describes the metadata of the stage along with tunables that will be propagated to the kubernetes job. 

An example template running a custom job to perform the `generic/pod-delete` chaos experiment is provided below: 

**Note**: The chaos jobs need the litmus chaos operator & CRDs to be installed. You can refer to the install & uninstall templates for litmus in [here](./templates/litmus-infra-ops)

```yaml
job:
  preconfigured:
    kubernetes:
      - label: Pod-Delete-Chaos
        type: ChaosExperiment
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
          - name: CHAOS_DURATION
            label: Chaos duration
            description: The time duration for chaos insertion (in sec)
            mapping: manifest.spec.template.spec.containers[0].env[2].value
            defaultValue: "30"
        manifest:
          apiVersion: batch/v1
          kind: Job
          metadata:
              name: run-pod-delete-chaos
          spec:
            backoffLimit: 0
            template:
              spec:
                restartPolicy: Never
                containers:
                  - name: run-pod-delete-chaos
                    image: mayadataio/chaos-ci-lib:ci
                    env:
                      - name: APP_NS
                        value: $(APPLICATION_NAMESPACE)
                      - name: APP_LABEL
                        value: $(APPLICATION_LABEL)
                      - name: TOTAL_CHAOS_DURATION
                        value: "$(CHAOS_DURATION)"
                    command: ["/bin/bash"]
                    args:
                    - -c
                    - ./pod-delete
```

## Updating the Custom Job Template

- Exec inside the halyard pod  

```console
kubectl exec -it halyard-0 bash
  
bash-5.0$ cd ~/.hal/default/profiles/
bash-5.0$ ls
front50-local.yml  gate-local.yml     orca-local.yml     settings-local.js  
```

- Update the orca-local.yml (if present) template within `/home/spinnaker/.hal/default/profiles` with the new custom job. Else create this file.
 
- Update the service by running `hal deploy apply`. This will restart the spin-orca microservices pod to read the latest orca profile

```console
bash-5.0$ hal deploy apply

+ Get current deployment
  Success
+ Prep deployment
  Success
Validation in default.features:
- WARNING Field Features.artifacts not supported for Spinnaker
  version 2.21.1: Artifacts are now enabled by default.
? You no longer need this.

- WARNING Field Features.artifactsRewrite not supported for
  Spinnaker version 2.21.1: The artifacts rewrite UI is now enabled by
  default.
? You no longer need this.

Validation in default:
- WARNING Version "2.21.1" was patched by "2.21.4". Please upgrade
  when possible.
? https://www.spinnaker.io/community/releases/versions/

Validation in default.stats:
- INFO Stats are currently ENABLED. Usage statistics are being
  collected���Thank you! These stats inform improvements to the product, and that
  helps the community. To disable, run `hal config stats disable`. To learn more
  about what and how stats data is used, please see
  https://www.spinnaker.io/community/stats.

+ Preparation complete... deploying Spinnaker
+ Get current deployment
  Success
+ Apply deployment
  Success
+ Deploy spin-redis
  Success
+ Deploy spin-clouddriver
  Success
+ Deploy spin-front50
  Success
+ Deploy spin-orca
  Success
+ Deploy spin-deck
  Success
+ Deploy spin-echo
  Success
+ Deploy spin-gate
  Success
+ Deploy spin-rosco
  Success
+ Run `hal deploy connect` to connect to Spinnaker.
```

- Wait for the spinnaker orca pod to restart & enter ready state

```
bash-5.0$ kubectl get pods -l app.kubernetes.io/name=orca -n spinnaker
NAME                         READY   STATUS    RESTARTS   AGE
spin-orca-6b84d44768-jtgml   1/1     Running   0          50m
```

- Refresh the Deck (Spinnaker UI) via a Relogin 

- Post this, the experiment-specific stage will appear in the list/drop-down for addition of new stages

![image](https://user-images.githubusercontent.com/21166217/97577372-59504400-1a15-11eb-8e04-3fe46ea85d20.png)



 
