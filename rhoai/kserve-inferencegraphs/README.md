# KServe InferenceGraph test on OpenShift AI
This test will try to deploy a KServe InferenceGraph on an OpenShift AI (version 2.6) using the models reported [here](https://kserve.github.io/website/master/modelserving/inference_graph/image_pipeline/#deploy-the-inferenceservices)

## Preliminary steps
- Create a Red Hat OpenShift cluster
- Install Red Hat ServiceMesh Operator
- Install Red Hat Serverless Operator
- Install Red Hat OpenShift AI (version 2.6)
- Create a Data Science Cluster using the following definition

```
apiVersion: datasciencecluster.opendatahub.io/v1
kind: DataScienceCluster
metadata:
  name: default-dsc
spec:
  components:
    codeflare:
      managementState: Removed
    dashboard:
      managementState: Managed
    datasciencepipelines:
      managementState: Managed
    kserve:
      managementState: Managed
      serving:
        ingressGateway:
          certificate:
            type: SelfSigned
        managementState: Managed
        name: knative-serving
    modelmeshserving:
      managementState: Managed
    ray:
      managementState: Removed
    trustyai:
      managementState: Managed
    workbenches:
      managementState: Managed
```

- Create a project ig-test and use it for the rest of the test

```
oc new-project ig-test --display-name 'InferenceGraph Test'
oc project ig-test
```


## Create a ServingRuntime

Since the InferenceService used in this test are pytorch based we need to create a ServingRuntime.

You can do it with the following command:

```
oc apply -f servingruntime.yaml
```

## Deploy the Inference Services

```
oc apply -f inferencesvcs.yaml
```

## Deploy the Inference Graph

```
oc apply -f inferencegraphs.yaml
```

## Test the InferenceGraphs

Before testing the InferenceGraph, first check if the graph is in the ready state and then get the router url for sending the request.

```
oc get ig  dog-breed-pipeline
NAME                 URL                                             READY   AGE
dog-breed-pipeline   http://dog-breed-pipeline.default.example.com   True    17h
```

You can test the inference graph by sending the cat and dog image data.


```
SERVICE_HOSTNAME=$(oc get inferencegraph dog-breed-pipeline -o jsonpath='{.status.url}' | cut -d "/" -f 3)
curl -v -H "Host: ${SERVICE_HOSTNAME}" -H "Content-Type: application/json" http://${SERVICE_HOSTNAME}:443 -d @./cat.json
```
Expected output

```
{"predictions": ["It's a cat!"]}
```

```
curl -v -H "Host: ${SERVICE_HOSTNAME}" http://${SERVICE_HOSTNAME} -d @./dog.json
```

Expected output

```
{"predictions": [{"Kuvasz": 0.9854059219360352, "American_water_spaniel": 0.006928909569978714, "Glen_of_imaal_terrier": 0.004635687451809645, "Manchester_terrier": 0.0011041086399927735, "American_eskimo_dog": 0.0003261661622673273}]}
```