
# Table of Contents

1.  [Clone the project](#org54e81c8)
2.  [Create the namespace](#org2792e2f)
3.  [Create the broker](#orgb7ec4d0)
4.  [Create the trigger](#org3038aa2)
5.  [Deploy the function](#orgf28977c)
6.  [Deploy an upload service](#org814db6d)
7.  [Send and verify CloudEvents to encode a video segment](#org1c7a5aa)
    1.  [Send CloudEvent to the Broker](#org7a18784)
    2.  [Check that the encoded video has been uploaded](#org61dcc74)



<a id="org54e81c8"></a>

# Clone the project

    git clone https://routerdi1315.uv.es:10000/review-serverless/sample-deployment.git

You will need a working knative platform to deploy this example. This is out of scope of this
document, but a working testbed can be built using kind or minikube.


<a id="org2792e2f"></a>

# Create the namespace

All the resources will be created in a namespace named `sample-video-coding-ns`.

Create the namespace:

    kubectl create ns sample-video-coding-ns


<a id="orgb7ec4d0"></a>

# Create the broker

This is the file that defines the broker [broker.yml](source/broker.yml):

    apiVersion: eventing.knative.dev/v1
    kind: Broker
    metadata:
     name: video-coding-broker
     namespace: sample-video-coding-ns

Create the broker:

    kubectl apply -f source/broker.yml


<a id="org3038aa2"></a>

# Create the trigger

This is the file that defines the trigger [x264-trigger.yml](source/x264-trigger.yml):

    apiVersion: eventing.knative.dev/v1
    kind: Trigger
    metadata:
      name: x264-trigger
      namespace: sample-video-coding-ns
      annotations:
         description: "Trigger used to send events from the broker to the x264 encoder function"
    spec:
      broker: video-coding-broker
      filter:
        attributes:
          # This trigger activates if a CloudEvent with header ce-type has value x264
          type: x264
      subscriber:
        ref:
         apiVersion: serving.knative.dev/v1
         kind: Service
         name: x264-fn-events
        uri: /x264-fn-events

Create the trigger:

    kubectl apply -f source/x264-trigger.yml


<a id="orgf28977c"></a>

# Deploy the function

This is the file that defines the serverless function [x264-fn-events.yml](source/x264-fn-events.yml)

    kind: Service
    apiVersion: serving.knative.dev/v1
    metadata:
      name: x264-fn-events
      namespace: sample-video-coding-ns
    spec:
      template:
        metadata:
          name: x264-fn-events-v1
          annotations:
             # 1 request per pod (no concurrency)
             autoscaling.knative.dev/metric: "concurrency"
             autoscaling.knative.dev/target: "1"
             autoscaling.knative.dev/minScale: "1"
             autoscaling.knative.dev/maxScale: "10"
             autoscaling.knative.dev/target-utilization-percentage: "100"
        spec:
          containers:
            - image: cloudmedialab/x264-fn-events:1.0
              env:
                - name: SERVICE_PORT
                  value: "8080"
                - name: SERVICE_PATH
                  value: "/x264-fn-events"
                - name: HEALTH_PATH
                  value: "/health"
                - name: METRICS_PATH
                  value: "/metrics"
                - name: LOG_LEVEL
                  value: "DEBUG"
                - name: EVENT_CONSUMER_URL
                  value: ""
              resources:
                requests:
                  cpu: "3000m"
                  memory: "2.5Gi"
              livenessProbe:
                httpGet:
                  path: /health
              readinessProbe:
                httpGet:
                  path: /health

Deploy the function:

    kubectl apply -f source/x264-fn-events.yml


<a id="org814db6d"></a>

# Deploy an upload service

For this sample, we will deploy an upload service [upload-deployment.yml](source/upload-deployment.yml):

    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: upload-app
      namespace: sample-video-coding-ns
    spec:
      selector:
        matchLabels:
          app: upload-app
      template:
        metadata:
          labels:
            app: upload-app
        spec:
          containers:
          - name: upload-app
            image: cloudmedialab/upload-tomcat-service:2.0
            env:
              - name: SERVICE_PORT
                value: "8080"
              - name: BASE_URL
                value: "/upload"
              - name: HEALTH_PATH
                value: "/health"
              - name: METRICS_PATH
                value: "/metrics"
              - name: LOG_LEVEL
                value: "DEBUG"
              - name: GET_ENABLED
                value: "true"
                # Max size in MB of the body in the request
              - name: MAX_REQUEST_THRESHOLD
                value: "10"
            resources:
              requests:
                memory: "256Mi"
                cpu: "500m"
            ports:
            - containerPort: 8080

    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: upload-service
      namespace: sample-video-coding-ns
    spec:
      type: ClusterIP
      selector:
        app: upload-app
      ports:
      - port: 80
        targetPort: 8080

Create the deployment and the service:

    kubectl apply -f source/upload-deployment.yml


<a id="org1c7a5aa"></a>

# Send and verify CloudEvents to encode a video segment

After you have deployed the application, and have verified that the namespace, sample application and trigger are ready:

    kubectl get all -n sample-video-coding-ns

you can send a CloudEvent.


<a id="org7a18784"></a>

## Send CloudEvent to the Broker

You can send an HTTP request directly to the Knative Broker with the correct CloudEvent headers and body.

Deploy a curl pod in the namespace and start a shell in the container:

    kubectl --namespace sample-video-coding-ns run curl --image=radial/busyboxplus:curl -it

Run the following command in that container shell:

    curl -v "broker-ingress.knative-eventing.svc.cluster.local/sample-video-coding-ns/video-coding-broker" \
    -X POST \
    -H "Ce-Id: 4ccad752-1173-11ed-ba84-bfd6ca98d94b" \
    -H "Ce-specversion: 1.0" \
    -H "Ce-Type: x264" \
    -H "Ce-Source: curl" \
    -H "Content-Type: application/json" \
    -d '{
         "sourceUrl":"https://routerdi1315.uv.es/video/video.mp4",
         "crf":"40",
         "frameRate":"23.98",
         "videoNameDestination":"video-crf-35.mp4",
         "destUrl":"http://upload-service/upload",
         "bucketParamName":"bucket",
         "bucketParamValue":"encodedVideos/"
        }'
    exit


<a id="org61dcc74"></a>

## Check that the encoded video has been uploaded

Make a request from the curl container to the upload service to list  all the files the bucket:

    kubectl exec -it curl -n sample-video-coding-ns -- curl http://upload-service:80/upload?bucket=encodedVideos

You should see a response like this one:

    [
      {
        "name": "video-crf-35.mp4_times.txt",
        "isDir": false,
        "date": 1659358777783,
        "size": 151
      },
      {
        "name": "video-crf-35.mp4",
        "isDir": false,
        "date": 1659358777771,
        "size": 19082
      }
    ]
