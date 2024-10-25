## Setting up the ELK stack on a Kubernetes instance

To set up the ELK stack on Kubernetes, we will be relying on Helm charts. Therefore, make sure you have Helm set up on your machine. If you need a quick refresher on Helm, check out the [Helm section](../Helm101/what-is-helm.md). However, a deep understanding of Helm is not required here. The stack we will be installing is FileBeat, Logstash, Elasticasearch, and Kibana. All of these will ultimately run on the same namespace within containers (pods). Since that will produce container logs, we will be using them as an input data source.

You will also need a Kubernetes cluster ready since that is where everything will be getting deployed to. Once again, I recommend [Minikube](https://minikube.sigs.k8s.io/docs/start/) to set up a cluster on your local machine, but feel free to use any cluster that is available to you.

### Process Overview

Here's a big picture of how this is going to work:

- The container creates logs which get placed in a path
- filebeat reads the logs from the path and sends them to logstash
- logstash transforms and filters the logs before sending them forward to elasticsearch
- Kibana queries elasticsearch and visualizes the log data

Note that logstash here is optional. You could very well send logs directly from filebeat to elasticsearch. However, in a real-world situation, rarely, you wouldn't want to use GROK patterns to match specific patterns that allow Kibana to show the data in a better-formatted manner, or filter logs based on the type of log, or add index patterns that allow the data to be retrieved much faster. If you have a large number of logs, logstash will also act as a buffer and prevent sending all the logs at once to elasticsearch, which would overload it. Therefore, we will be sending the data to logstash first in this example.

As you can imagine, all of the above components are quite complex and require multiple resources to be deployed to get up and running. Therefore, we will be using Helm charts for each of the above components that automatically set up all the required resources for us. We could also use the single Kubernetes manifest file that elastic provides us, but the Helm chart allows for better flexibility.

### Setting up

To keep everything logically grouped together, we will be installing all the components in a namespace called `kube-logging`. So create it:

```
kubectl create ns kube-logging
```

The first component we will look into is filebeat. To set up filebeat, go ahead and get the relevant Helm chart from [its artifacthub page](https://artifacthub.io/packages/helm/elastic/filebeat?modal=install). Then use the provided commands to install the Helm chart:

```
helm repo add elastic https://helm.elastic.co
```

```
helm install filebeat  elastic/filebeat --version 8.5.1 -n kube-logging
```

If your kubeconfig file is set properly and it is pointed to your cluster, the command should run with no issues. You can then open up a terminal instance and run:

```
kubectl get po -n kube-logging
```

This will show you the filebeat pods that have started running in the kube-logging namespace. While the default configuration is meant to work out of the box, it won't fit our specific needs, so let's go ahead and customize it. To get the values.yaml, head back over to the Helm chart page on [Artifact hub](https://artifacthub.io/packages/helm/elastic/filebeat/7.6.1) and click on the "Default Values" option. Download this default values file.

We will start by modifying this file. The values file provides configuration to support running filebeat as both a DaemonSet as well as a regular Deployment. By default, the DaemonSet configuration will be active, but we don't need that right now, so let's go ahead and disable it. Under the `daemonset` section, change `enabled: true` to `enabled: false`.  You can then skip the rest of the DaemonSet section and head over to the `deployment` section. Then set change enabled to true to get filebeat deployed as a deployment. You might notice that the daemonset section and the deployment section are pretty much the same, so you only need to do the configuration for one section for it to work.

If you have used filebeat before, you would be well aware of the `filebeat.yaml`. This is the file where you specify the filebeat configurations, such as where to get the logs from, and where to send the logs to. In this case, we will be declaring the yaml within the values.yaml. There is a basic filebeat.yml defined within the values file to help you get started. We will be changing everything here. Replace this block with the below code:

```
filebeat.yml: |
    filebeat.inputs:
    - input_type: log
      paths:
        - /var/log/containers/*.log
      document_type: mixlog
    output.logstash:
      hosts: ["logstash-logstash:5044"]
```

The path provided above will get all the logs produced by the container. We will be marking these logs as type "mixlog" and filtering these out based on this type later in the flow. The last piece of configuration is `output.logstash` which tells where the log should be sent to. Here, we specify logstash instead of elasticsearch. Note that since this is loading a regular filebeat container, you are allowed to do additional configuration that you would normally do with a pod. For example, if instead of just getting the container logs of the same pod-like we are doing here, you wanted to get the logs from a different pod, that would be possible. To do so, you would have to copy the logs from your source container to a network volume mount (such as EFS) which would mean that the volume now has your source pods' logs. To mount these volumes, you would use the `extraVolumes` and `extraVolumeMounts` options that you see in the Helm values file. You can then mount this volume into your filebeat pod and change the filebeat.yml to point at the mounted volume:

```
paths:
  - /different_pod_logs/*.log
```

The filebeat configuration is now complete. We will now handle the second part of the log flow: logstash. As with filebeat, we will be using Helm charts to get logstash up on the Kubernetes cluster. We will also be changing the values file in the same way.

To start, head over to the logstash chart on [Artifact Hub](https://artifacthub.io/packages/helm/elastic/logstash). As with filebeat, download the values file. While in filebeat we had the filebeat.yml to set our configuration, in logstash we have a `logstash.conf`. We will be declaring the logstash conf in the yaml as we did before. Remove any default logstash conf values that may exist, and replace them with:

```
logstashPipeline:
 logstash.conf: |
   input {
     beats {
      type => mixlog
      port => 5044
     }
   }
   filter {
      if [type] == "mixlog" {
        grok {
          match => {
            "message" => "%{TIMESTAMP_ISO8601:timestamp}"
          }
        }
      }
    }
   output { elasticsearch { hosts => "http://elasticsearch:9200" } }
```

In the above config, we declare that we will be getting inputs from filebeat from port 5044 (which is the port filebeat exposes). This port will send logs of type "mixlog" which we declared in the filebeat.yml. Now, logstash will start gathering all logs that filebeat sends from port 5044, which we can either redirect to Elasticsearch or perform some processing on. In this case, we will be using GROK patterns to do the filtering. GROK patterns are very similar to regex patterns and can be used in other log-matching services such as fluentd. In the above config, we declare that for any log to go through, it must have a timestamp. All sorts of filters can applied, and you can find a comprehensive list of all of them [here](https://www.elastic.co/guide/en/logstash/current/filter-plugins.html). There are also other things logstash can do to your logs apart from filtering them. The final thing we do in the config is to redirect the logs to elasticsearch. For this, we specify the host and port of elasticsearch, and that is all it takes.

Note that all of these steps assume that you are running the entire stack on one single namespace. If, for example, you have logstash in one namespace and elasticsearch in another namespace, this above configuration will not work. This is because we are simply referring to elasticsearch with "http://elasticsearch:9200" without providing any namespaces, which means elasticsearch will automatically assume that it should look within the same namespace. If you need to specify a different namespace, you would have to use the full svc name: elasticsearch.different-namespace.svc.cluster.local:9200.

The filebeat configuration is now complete, and we can move on to the next phase of the flow: elasticsearch. As with the previous two components, you can set up elasticsearch using the relevant Helm chart. Go to the page in the [Artifact Hub](https://artifacthub.io/packages/helm/elastic/elasticsearch) and get the provided install command:

```
helm install my-elasticsearch elastic/elasticsearch --version 8.5.1
```

Unlike with the previous two modules, the elasticsearch chart can be installed as-is and work properly. This is because elasticsearch is being used here as somewhat of a data store. No data is being transformed or filtered, and since the data is already being fed in by filebeat, there is no need to set up custom volume mounts. Installing the chart will create an elasticsearch statefulset that will create two pods. It will also create a service that runs on port 9200. This way, you have a fully functional elasticsearch istance that logstash can reach. 

However, you also have a second option if you require more flexibility over the elasticsearch cluster. Instead of using Helm, go ahead and use [this yaml](https://gist.github.com/harsh4870/ccd6ef71eaac2f09d7e136307e3ecda6). If you are fine with the level of customizability the Helm chart provides, you can skip this part and head straight to the Kibana section. Once you have created a file from it, we can modify the file so that we have all our fine-grained configs. You might notice that this file only contains a StatefulSet. However, for elasticsearch to be exposed to other services such as logstash and Kibana, it needs a service. So create a file called `elasticsearch_svc.yaml` and add this into it:

```
kind: Service
apiVersion: v1
metadata:
  name: elasticsearch
  namespace: kube-logging
  labels:
    app: elasticsearch
spec:
  selector:
    app: elasticsearch
  clusterIP: None
  ports:
    - port: 9200
      name: rest
    - port: 9300
      name: inter-node
```

The above yaml will allow services to call elasticsearch for port 9200. As you can see, it is fairly straightforward. So let's move on to the StatefulSet. The only config you need to set is the `cluster.name` config which is where you need to set the name of your elasticsearch cluster. Apart from that, everything else is filled in. In cases where a large amount of data is going to be flowing into your cluster, you can increase the `ES_JAVA_OPTS` value to something higher than 512 MB. You also have fine-grained control over things such as the DNS policy and init commands used when starting the elasticsearch cluster. Once you have finished doing all the necessary configurations, you can go ahead and deploy the elasticsearch cluster with the `kubectl apply` command. You are almost done with the ELK stack setup.

The final part of the stack is Kibana. Now that you have all the necessary pieces in place to capture, filter, and store your logs in elasticsearch, it is time to start making use of this data by visualizing it. As with the other modules, we can use Helm to install Kibana:

```
helm install my-kibana elastic/kibana --version 8.5.1
```

You should also take the values yaml and make modifications to it. Specifically, you might notice that the values file has the elasticsearch host defined as "https://elasticsearch-master:9200". Change this to "https://elasticsearch:9200" since that is the name of the service that runs elasticsearch. Apart from that, there are no major modifications that need doing, and simply installing the chart should get Kibana to start working. While installing this may get Kibana up and running, you won't yet be able to access it from your browser. This is due to the Helm chart having a cluster IP setup. You can use port forwarding to forward your localhost port to the Kibana port and access Kibana this way.

```
kubectl port-forward <kibana-pod-name> 5601:5601
```

This will allow you to access Kibana at http://localhost:5601. You could also edit the svc and set the type to "NodePort" which would allow you to access the port using the IP of the pod. If you are running a managed cluster on something like EKS, you will have to open the port from the security group or firewall to allow external access to Kibana. However, all this is not necessary in a testing setup, so we will be sticking to port forwarding.

Now that Kibana is accessible, you might notice that the dashboard is empty. This is because Kibana lacks any indexes. Since we are using logstash and the logstash format, we can simply ask Kibana to index any entries that come formatted with "logstash-*". To set this index, head over to the main menu > stack management > index patterns and choose "Create index pattern". Once you type in the proper index, Kibana will search elasticsearch and match the index pattern with the logs. If no results are found, then something has gone wrong with the process. There might be some issue with either filebeat, logstash, or elasticsearch. If everything is working fine, you should see several logs matching the pattern, and you can go ahead and add the index. With the index added, go back to the dashboard and you should see the logs being presented here. Note that since we used a grok pattern, Kibana will try to fit the data into the provided pattern. In this case, we captured the timestamp so you should see a field called timestamp being separated from the rest of the message. Depending on what your log is, you could use grok patterns to capture various parts of the log. For example, if you are logging any Java errors you get from a Java application, you should be able to separately capture the method from which the error originated, the stack trace, and the severity of the error and have them separated by grok patterns. Afterward, when they are displayed by Kibana, you should be able to see the logs being separated by the fields you specified grok patterns with. You can then use this for further filtering from within Kibana. So for example, if your grok pattern matched a field called "severity", you can use "severity: high" in the Kibana search bar to only get the high severity logs.

That brings us to the end of setting up an end-to-end ELK setup on your Kubernetes cluster. So now, let us discuss some potential problems with the setup about Kubernetes.

Firstly, Kubernetes has a highly distributed nature, where each pod runs a separate application. This in itself is not a problem. Even if you have hundreds of pods running a hundred different applications, filebeat is more than capable of handling all those open log files that are being constantly written into and passing them on to logstash. Logstash then manages the in-flow of logs to make sure elasticsearch isn't overwhelmed. The problems start appearing if you have a sudden surge in the number of pods. This is not normal when it comes to everyday Kubernetes use cases. However, if you were using autoscaling jobs that massively scaled up and down depending on the workload, this could happen. One good example is with [KEDA](../Keda101/what-is-keda.md). KEDA looks at whatever metric you specify and massively scales jobs (basically pods) up and down to handle the demand. If each of these jobs writes a log to a common data source (such as EFS or other network file system), you could potentially have hundreds of open log files that are concurrently being written into. At this point, a single instance of filebeat may be unable to keep track of all these log files and either end up skipping some logs or stop pushing logs entirely. The solution for this is to either have multiple replicas of filebeat or to launch filebeat as a sidecar container for each pod that comes up. However, this is beyond the scope of this tutorial. The setup you have created now is enough to push logs for any regular applications.

With that, we are done with the logging section of this tutorial.