## Using Incremental Learning Job in Helmet Detection Scenario

This document introduces how to use incremental learning job in helmet detection scenario.
Using the incremental learning job, our application can automatically retrains, evaluates,
and updates models based on the data generated at the edge.

### Prepare Model
In this example, we need to prepare base model and deploy model in advance.
download [models](https://kubeedge.obs.cn-north-1.myhuaweicloud.com/examples/helmet-detection/model.tar.gz), including base model and deploy model.

```
cd /
wget https://kubeedge.obs.cn-north-1.myhuaweicloud.com/examples/helmet-detection/models.tar.gz
tar -zxvf models.tar.gz
```{{execute HOST2}}

### Prepare for Inference Worker
In this example, we simulate a inference worker for helmet detection, the worker will upload hard examples to `HE_SAVED_URL`, while
it inferences data from local video. We need to make following preparations:  

Make sure following localdirs exist.  

```
mkdir -p /incremental_learning/video/
mkdir -p /incremental_learning/he/
mkdir -p /data/helmet_detection
mkdir /output
```{{execute HOST2}}

Download [video](https://kubeedge.obs.cn-north-1.myhuaweicloud.com/examples/helmet-detection/video.tar.gz), unzip video.tar.gz, and put it into `/incremental_learning/video/`.

```
cd /incremental_learning/video/
wget https://kubeedge.obs.cn-north-1.myhuaweicloud.com/examples/helmet-detection/video.tar.gz
tar -zxvf video.tar.gz
```{{execute HOST2}}


### Create Incremental Job
In this example, `$WORKER_NODE` is a custom node, you can fill it which you actually run.

```
WORKER_NODE="node1" 
```{{execute}}

Create Dataset

```
kubectl create -f - <<EOF
apiVersion: sedna.io/v1alpha1
kind: Dataset
metadata:
  name: incremental-dataset
spec:
  url: "/data/helmet_detection/train_data/train_data.txt"
  format: "txt"
  nodeName: $WORKER_NODE
EOF
```{{execute}}

Create Initial Model to simulate the initial model in incremental learning scenario.

```
kubectl create -f - <<EOF
apiVersion: sedna.io/v1alpha1
kind: Model
metadata:
  name: initial-model
spec:
  url : "/models/base_model"
  format: "ckpt"
EOF
```{{execute}}

Create Deploy Model

```
kubectl create -f - <<EOF
apiVersion: sedna.io/v1alpha1
kind: Model
metadata:
  name: deploy-model
spec:
  url : "/models/deploy_model/saved_model.pb"
  format: "pb"
EOF
```{{execute}}

Start The Incremental Learning Job

* incremental learning supports hot model updates and cold model updates. Job support
  cold model updates default. If you want to use hot model updates, please to add the following fields:

```yaml
deploySpec:
  model:
    hotUpdateEnabled: true
    pollPeriodSeconds: 60  # default value is 60
```

* create the job:

```
IMAGE=kubeedge/sedna-example-incremental-learning-helmet-detection:v0.3.1

kubectl create -f - <<EOF
apiVersion: sedna.io/v1alpha1
kind: IncrementalLearningJob
metadata:
  name: helmet-detection-demo
spec:
  initialModel:
    name: "initial-model"
  dataset:
    name: "incremental-dataset"
    trainProb: 0.8
  trainSpec:
    template:
      spec:
        nodeName: $WORKER_NODE
        containers:
          - image: $IMAGE
            name:  train-worker
            imagePullPolicy: IfNotPresent
            args: ["train.py"]
            env:
              - name: "batch_size"
                value: "32"
              - name: "epochs"
                value: "1"
              - name: "input_shape"
                value: "352,640"
              - name: "class_names"
                value: "person,helmet,helmet-on,helmet-off"
              - name: "nms_threshold"
                value: "0.4"
              - name: "obj_threshold"
                value: "0.3"
    trigger:
      checkPeriodSeconds: 60
      timer:
        start: 02:00
        end: 20:00
      condition:
        operator: ">"
        threshold: 500
        metric: num_of_samples
  evalSpec:
    template:
      spec:
        nodeName: $WORKER_NODE
        containers:
          - image: $IMAGE
            name:  eval-worker
            imagePullPolicy: IfNotPresent
            args: ["eval.py"]
            env:
              - name: "input_shape"
                value: "352,640"
              - name: "class_names"
                value: "person,helmet,helmet-on,helmet-off"                    
  deploySpec:
    model:
      name: "deploy-model"
    trigger:
      condition:
        operator: ">"
        threshold: 0.1
        metric: precision_delta
    hardExampleMining:
      name: "IBT"
      parameters:
        - key: "threshold_img"
          value: "0.9"
        - key: "threshold_box"
          value: "0.9"
    template:
      spec:
        nodeName: $WORKER_NODE
        containers:
        - image: $IMAGE
          name:  infer-worker
          imagePullPolicy: IfNotPresent
          args: ["inference.py"]
          env:
            - name: "input_shape"
              value: "352,640"
            - name: "video_url"
              value: "file://video/video.mp4"
            - name: "HE_SAVED_URL" 
              value: "/he_saved_url"
          volumeMounts:
          - name: localvideo
            mountPath: /video/
          - name: hedir
            mountPath: /he_saved_url
          resources:  # user defined resources
            limits:
              memory: 2Gi
        volumes:   # user defined volumes
          - name: localvideo
            hostPath:
              path: /incremental_learning/video/
              type: DirectoryorCreate
          - name: hedir
            hostPath:
              path:  /incremental_learning/he/
              type: DirectoryorCreate
  outputDir: "/output"
EOF
```{{execute}}

1. The `Dataset` describes data with labels and `HE_SAVED_URL` indicates the address of the deploy container for uploading hard examples. Users will mark label for the hard examples in the address.
2. Ensure that the path of outputDir in the YAML file exists on your node. This path will be directly mounted to the container.

### Check Incremental Learning Job
Query the service status:

```
kubectl get incrementallearningjob helmet-detection-demo -o yaml
```{{execute}}
