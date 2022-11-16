# What is going on here?

This folder contains all the `pipeline as code` components needed to run the VIB CI pipeline for this project. 

[buildpacks/vib.json](buildpacks/vib.json) contains the main VIB pipeline. Here is some useful information about how those pipelines are usually structured:

* `Phases` are predefined ala Maven style. There are three phases: `package`, `verify`, `publish`. 
* Within each phase you can run many different `actions`. VIB actions are very similar to GitHub Actions. They do have an schema, they get some inputs and provide some outputs. There is a catalog of available actions at [our internal gitlab](https://gitlab.eng.vmware.com/marketplace-isv-services/verification-engine/puzzle). Note that each action is always associated to a specific lifecycle phase.
* Phases have a working `context defined`. That context can also be defined at a `global` level. The context provides some common input for all the actions within that phase. For example:
```json
"context": {
    "resources": {
        "url": "{SHA_ARCHIVE}",
        "path": "/containerize/buildpack"
    }
}
```
provides some context to the actions about where are the resources and which folder should the actions use to run. Note that `SHA_ARCHIVE` is a special variable (we need a better name for this one) that the GitHub Action will populate prior to sending the pipeline to VIB. This particular variable points to the zip/tarball URI for the event that triggered the GitHub workflow (e.g. PR, push to main, etc.)
* The GitHub Action replaces variables within curvy brackets (only ones starting with VIB_ENV_) from the environment. This provides a poor man's templating engine. So for example:
```json
{
    "action_id": "buildpacks",
    "params": {
        "application": {
            "details": {
                "name": "modern-spring-on-kubernetes",
                "tag": "buildpacks-{VIB_ENV_REF}"
            }
        },
        "env": {
            "BP_JVM_VERSION": "17.*"
        }
    }
}
```
Before sending the pipeline to VIB, the GitHub action will fetch the value for VIB_ENV_REF from the environment and replace it. Here in particular we are setting the tag for the container image that will be created by the buildpacks engine.
* Some actions need actionable code, like cypress tests, or goss scripts. That is provided through the context. For example here:
```json
{
"action_id": "goss",
    "params": {
        "resources": {
            "path": "goss"
        },
        "remote": {
            "workload": "deploy-modern-spring-on-kubernetes"
        }
    }
},
```
we are telling the goss action that the folder with the goss scripts is at the relative path `goss`. That is relative to the context itself which points to `/verify`. In summary, [this is the file being read](verify/goss).
* Several actions in the `verify` phase do actual verification on the containers or charts. For verifying a cloud native application, it has to be deployed. And for deploying something, you need a place. That is what we call a target platform. Today, target platforms can be pulled via API. The example here sets the target platform on the GitHub Workflow and uses `91d398a2-25c4-4cda-8732-75a3cfc179a1`. When I go to the VIB API, this is what I get for that:
```json
bash-3.2$ curl -s -H "Authorization: Bearer ${BEARER_TOKEN}" https://cp.bromelia.vmware.com/v1/target-platforms/91d398a2-25c4-4cda-8732-75a3cfc179a1 | jq .
{
  "architecture": "AMD_64",
  "id": "91d398a2-25c4-4cda-8732-75a3cfc179a1",
  "name": "GKE Kubernetes v1.22.x",
  "description": "Google Kubernetes Engine (Kubernetes v.1.22.x)",
  "kind": "GKE",
  "version": "1.22",
  "provider": "GCP",
  "environment_sizes": [
    {
      "default": true,
      "name": "S4",
      "description": "Small. Worker Nodes 2vCPU-8Gb",
      "worker_nodes_instance_type": "e2-standard-2",
      "worker_nodes_instance_vcpu": 2,
      "worker_nodes_instance_memory": 8,
      "worker_nodes_instance_count_min": 1,
      "worker_nodes_instance_count_max": 5,
      "worker_nodes_instance_count_default": 3,
      "master_nodes_instance_type": "Managed by GKE",
      "master_nodes_instance_vcpu": 0,
      "master_nodes_instance_memory": 0,
      "master_nodes_instance_count_min": 1,
      "master_nodes_instance_count_max": 1,
      "master_nodes_instance_count_default": 1
    },
...
```
I'm actually keeping the output intentionally short as each target platform has several different T-shirt sizes that you can choose from.

Let's look at the [GitHub Workflow](../.github/workflows/vib-ci-buildpacks.yaml) now.

* The GitHub Action needs some variables for it to be able to run. Those variables have defaults but we always try to make it explicit. This is the bare minimal needed:
```
env:
  CSP_API_URL: https://console.cloud.vmware.com
  CSP_API_TOKEN: ${{ secrets.CSP_API_TOKEN }}
  VIB_PUBLIC_URL: https://cp.bromelia.vmware.com
```

* What does not have a default is the `CSP_API_TOKEN`. That is a CSP token. To use VIB you need to be onboarded to an organization that has access to the service.
* Then, invoking the action is pretty simple:
```yaml
- uses: vmware-labs/vmware-image-builder-action@0.4.12
    name: VIB
    with:
        pipeline: buildpacks/vib.json
    env:
        VIB_ENV_REF: ${{ github.ref_name }}
```
* The action has [a few configuration parameters](https://github.com/vmware-labs/vmware-image-builder-action/blob/main/action.yml). It's still under continuous development.
* The GitHub Action stores all the results from the pipeline as GitHub Workflow run assets. That way you can download all reports, e.g. trivy/grype, action logs, everything from GitHub itself. You will find those assets on the GitHub Action summary panel at the very bottom.
* The GitHub Action can be run in [debug mode](https://github.com/vmware-labs/vmware-image-builder-action#enabling-debug-logging). That provides some crazy verbosity but can be good for learning the internals too. 

### Final remarks

The GitHub Action execution provides quite a lot of information. Exploring it provides nice insights. [Here is a sample action run](https://github.com/mpermar/modern-spring-on-kubernetes/actions/runs/3477476006/jobs/5813680660).

If you drill into the action run sections, you'll spot the execution graph. That URI is the entry point for pulling all the information about that pipeline execution. A sample execution graph URI looks like `https://cp.bromelia.vmware.com/v1/execution-graphs/0af8f3a1-6adf-4744-8289-3557d63a4ac5`. Note that clicking it won't work. You would need to be in the same CSP org for being able to view it. 

The execution graph content is what contains all the details about current progress and final execution. It contains all the detailed execution graph, with all the internal data, steps, whether the actions succeeded or not. It can be truly verbose and it is not intended for human reading. All the GitHub Action does after sending a pipeline is essentially polling the pipeline execution graph for progress and when it is completed pull all the information.  

Here is a sneak peak to the above execution graph:

```json
{
  "execution_graph_id": "2fe53d10-5e3e-4538-8f4a-91548b973419",
  "status": "SUCCEEDED",
  "organization_id": "58edc764-0d7d-4e57-b6cd-dfa727762024",
  "execution_time": 548,
  "created_at": "2022-11-15T20:42:23.666293Z",
  "started_at": "2022-11-15T20:42:25.211044Z",
  "tasks": [
    {
      "action_id": "buildpacks",
      "action_version": "0.98.1",
      "status": "SUCCEEDED",
      "execution_time": 179,
      "started_at": "2022-11-15T20:42:25.211044Z",
      "phase": "PACKAGE",
      "task_id": "fb474649-d1aa-4dc4-84b0-018b415975bf",
      "previous_tasks": [],
      "next_tasks": [
        "6e1bd432-b159-4ee1-bc72-42e69b775a8d",
        "782cb2cc-951a-4515-a9f2-e6d74b6fcf90",
        "7084190a-58d8-4f95-b238-8a08e74dc595"
      ],
      "preconditions": [],
      "params": {
        "resources": {
          "path": "/containerize/buildpack",
          "url": "https://github.com/mpermar/modern-spring-on-kubernetes/tarball/fc4924b55b73814cacc1f2727d33587bb1525841"
        },
        "application": {
          "credentials": [
            {
              "authn": {
                "username": "cp-storage-rw-token"
              },
              "url": "oci://vmwaresaas.jfrog.io/content-platform-docker"
            }
          ],
          "details": {
            "name": "modern-spring-on-kubernetes",
            "tag": "buildpacks-fc4924b55b73814cacc1f2727d33587bb1525841"
          }
        },
        "env": {
          "BP_JVM_VERSION": "17.*"
        }
      }
    },
    {
      "action_id": "grype",
      "action_version": "0.98.1",
      "status": "SUCCEEDED",
      "execution_time": 110,
      "started_at": "2022-11-15T20:45:26.777962Z",
      "phase": "VERIFY",
```

# References

* My friend Dani [has a much better version](https://github.com/dani8art/modern-spring-on-kubernetes) of this repo with more ellaborated workflows. 
* [VIB API](http://developers.eng.vmware.com/apis/content-platform/latest/).
* [GitHub Action](https://github.com/vmware-labs/vmware-image-builder-action/).
* [VIB Actions source code](https://gitlab.eng.vmware.com/marketplace-isv-services/verification-engine/puzzle)
