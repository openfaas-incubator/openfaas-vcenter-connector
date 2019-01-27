# vcenter-connector

vcenter-connector is an OpenFaaS event connector built to consume events from vCenter from vCenter.

With this project your functions can subscribe to events generated by the changes (i.e. events) in your vCenter installation - for instance a VM being created, turned on or deleted. This allows you to extend vCenter's functionality by writing functions to execute each time an event is fired. An example may be tagging a VM with the date it was last turned on or applying a tag showing which user made a change to an object. 

## Status

This project uses the [OpenFaaS Connector SDK](https://github.com/openfaas-incubator/connector-sdk).

The code is under **active development** and only suitable for early adopters. For the initial version the vCenter user credentials need to be stored in plain-text in your YAML files, but this will move to using [OpenFaaS secrets](https://docs.openfaas.com/reference/secrets/) in the next version.

**Note:** Currently only a pre-built Docker image is available (based on [PR#11](https://github.com/openfaas-incubator/vcenter-connector/pull/11)) and only [VM events](https://code.vmware.com/doc/preview?id=4206#/doc/vim.event.VmEvent.html) are reportable for now.

## Example: vCenter Tagging Function

### Pre-reqs:

* [OpenFaaS](https://docs.openfaas.com/) running on a local or remote Kubernetes cluster (e.g. [kind](https://blog.alexellis.io/get-started-with-openfaas-and-kind/))
* An installation of vCenter (tested against 6.5)
* A vCenter user/service account with sufficient rights to perform the (tagging) action of the example function
* `docker` to run tools like `govc` if not installed on your machine already
* `git` to clone the function example
* `faas-cli` to deploy the function
* `kubectl` to deploy the connector

**Note:** Make sure that your OpenFaaS environment can reach vCenter as the tagging function performs API calls against vCenter.

### How it works:

Functions can subscribe to events in vCenter through the "topic" [annotations](https://docs.openfaas.com/reference/yaml/#function-annotations) applied through your `stack.yml` file. Based on these events a function can take action, e.g. tag a VM, run post-processing scripts, audit to an external system, etc.

### Get started with the vCenter Tagging Function example

In the following example we will subscribe to the event "vm.powered.on" by adding an annotation to our function of "vm.powered.on". The function will then add a specific tag to any VM when it is powered on.

**Note:** In a DRS-enabled cluster the event is called `drs...` and the example would not work as it's a different event type. If you run this example in a DRS cluster you can use `vm.powered.off` throughout the example below as a workaround.

1) Create a category/tag to be attached to a VM when it is powered on. Since we need the unique tag ID (i.e. vSphere URN) we will use [govc](https://github.com/vmware/govmomi/tree/master/govc) for this job. You can also use vSphere APIs (REST/SOAP) to retrieve the URN.

```bash
# Run pre-built govc Docker image
docker run --rm -it embano1/govc:0ee42d3 sh

# Test connection to vCenter, ignore TLS warnings
export GOVC_INSECURE=true
export GOVC_URL='https://vcuser:vcpassword@vcenter.ip' 
./govc tags.ls

# If the connection is successful create a demo category/tag to be used by the function
./govc tags.category.create democat1
urn:...
./govc tags.create -c democat1 demotag1
urn:vmomi:InventoryServiceTag:019c0a9e-0672-48f5-ac2a-e394669e2916:GLOBAL
```
2) Take a note of the `urn:...` for `demotag1` as we will need it for the next steps
3) In a separate terminal download the example function

```bash
git clone https://github.com/embano1/pytagfn
cd pytagfn
```

4) Configure the Python tagging function `stack.yaml`. 

> **Note:** The example cloned from Github already has the annotation to subscribe to VM power on events. More details in the [README](https://github.com/embano1/pytagfn/blob/master/README.md).

```yaml
environment:
    VC: vcenter.ip                      # FQDN/IP, must be reachable/resolvable from OpenFaaS
    VC_USERNAME: VCUSER                  # WIP: migration to secrets
    VC_PASSWORD: VCPASSWORD              # WIP: migration to secrets
    # Replace TAG_URN example below with the one you created with govc above
    TAG_URN: urn:vmomi:InventoryServiceTag:019c0a9e-0672-48f5-ac2a-e394669e2916:GLOBAL 
    TAG_ACTION: attach                   # this function also supports detach
```

5) Deploy the function

```bash
faas-cli deploy
Deploying: pytag-fn.

Deployed. 202 Accepted.
URL: http://127.0.0.1:8080/function/pytag-fn
```

6) Download and deploy the OpenFaaS vCenter Connector deployment manifest in a separate

> **Note:** The deployment assumes you have basic authentication configured for OpenFaaS on Kubernetes as per [this guide](https://github.com/openfaas/faas-netes/blob/67f61a468bc73833e53b626fa5243f5d539a9e00/yaml/README.md#L5). Thus, the deployment assumes a secret `gateway-basic-auth` to be available (`volumes` section in the YAML). If you don't use authentication for the gateway, remove the volumes section as the deployment would fail, not being able to mount the secret to the deployment.

```bash
git clone https://github.com/openfaas-incubator/vcenter-connector
cd vcenter-connector

# In yaml/kubernetes/connector-dep.yml modify the container args "-vcenter" (following URL scheme), "-vc-user" and "-vc-pass" accordingly
# Note: If you are not running your vCenter connector in the same cluster as OpenFaaS edit the -gateway flag

# Deploy the connector to Kubernetes
kubectl -n openfaas create -f yaml/kubernetes/connector-dep.yml

# Check the logs of the pod whether it connected successfully to vCenter and OpenFaaS
kubectl -n openfaas logs deploy/vcenter-connector -f
```

7) Generate a "Power On" event

In the next step we will power on a VM to trigger an event in vCenter ("VM powered on..."). This event will be received by the connector and forwarded to the OpenFaaS function subscribed to the corresponding event type (`vm.powered.on`). The function will then add the tag we created above (`demotag1`) to the VM.

```bash
# Note: This can be done in vCenter UI or via govc
# Pick a VM that is powered off and does not have already "demotag1", then in the running govc container power on the VM
./govc vm.power -on /Datacenter-North/vm/Nested-Pod/vesxi67-2

# Verify that the tag was correctly attached
./govc tags.attached.ls demotag1
VirtualMachine:vm-267
```

### Troubleshooting

If your VM did not get the tag attached, verify:

- vCenter IP/username/password
- Permissions of the vCenter user
- Whether the components can talk to each other (connector to vCenter and OpenFaaS, function to vCenter)
- Check the logs:

```bash
kubectl -n openfaas logs vcenter-connector-<...> -f

# Successful log message in the OpenFaaS vCenter connector
2019/01/25 23:39:09 Message on topic: vm.powered.on
2019/01/25 23:39:09 Invoke function: pytag-fn
2019/01/25 23:39:10 Response [200] from pytag-fn
```
- Enable debugging in the function with `write_debug` and `read_debug` env vars in `stack.yaml`

```bash
kubectl -n openfaas-fn logs pytag-fn-<...> -f

# Successful log message in the OpenFaaS tagging function
2019/01/25 23:48:55 Forking fprocess.
2019/01/25 23:48:55 Query
2019/01/25 23:48:55 Path  /

{"status": "200", "message": "successfully attached tag on VM: vm-267"}
2019/01/25 23:48:56 Duration: 1.551482 seconds
```

## License

MIT

## Acknowledgements

Thanks to VMware's Doug MacEachern for the awesome [govmomi](https://github.com/vmware/govmomi) project providing Golang bindings for vCenter and the [vcsim simulator tool](https://github.com/vmware/govmomi/blob/master/vcsim/README.md).

Thanks to Karol Stępniewski for showing me a demo of events being consumed in OpenFaaS via vCenter over a year ago at KubeCon in Austin. Parts of his "event-driver" originally developed in the Dispatch project have been adapted for this OpenFaaS event-connector including a method to convert camel case event names into names separated by a dot. I wanted to include this for compatibility between the two systems.

Other acknowledgements: Michael Gasch (VMware) and Ivana Yocheva (VMware).
