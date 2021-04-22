# Creating the Monitoring log stack EFK(Elastic + Fluentd + Kibana)

Those steps are to create a monitoring kubernetes logs, including the pods with the apps.

After you had all k8s cluster up running we can use those steps to spin the monitoring stack.

## Elastic

*elastic.yaml*

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: elasticsearch
spec:
  selector:
    matchLabels:
      component: elasticsearch
  template:
    metadata:
      labels:
        component: elasticsearch
    spec:
      containers:
        - name: elasticsearch
          image: docker.elastic.co/elasticsearch/elasticsearch:6.5.4
          env:
            - name: discovery.type
              value: single-node
          ports:
            - containerPort: 9200
              name: http
              protocol: TCP
          resources:
            limits:
              cpu: 500m
              memory: 4Gi
            requests:
              cpu: 500m
              memory: 4Gi

---
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
  labels:
    service: elasticsearch
spec:
  type: NodePort
  selector:
    component: elasticsearch
  ports:
    - port: 9200
      targetPort: 9200
```

So, this will spin up a [single node](https://www.elastic.co/guide/en/elasticsearch/reference/current/bootstrap-checks.html#single-node-discovery) Elasticsearch pod in the cluster along with a [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#nodeport) service to expose the pod to the outside world.

Create:

```
$ kubectl create -f elastic.yaml -n logging
```

deployment.extensions/elasticsearch created
service/elasticsearch created


Verify that both the pod and service were created:

```
$ kubectl get pods -n logging

NAME                          READY   STATUS    RESTARTS   AGE
elasticsearch-bb9f879-d9kmg   1/1     Running   0          75s


$ kubectl get service -n logging

NAME            TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
elasticsearch   NodePort   10.102.149.212   <none>        9200:30531/TCP   86s
```

Take note of the pod name – e.g, elasticsearch-bb9f879-d9kmg . Then, create a port-forward and make sure you can cURL that pod:

```
$ kubectl -n logging port-forward elasticsearch-bb9f879-d9kmg 7000:9200  

$ curl http://localhost:7000
{
  "name" : "pqinS-H",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "GMoqy0pVTx6mcYJASZzdJg",
  "version" : {
    "number" : "6.5.4",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "d2ef93d",
    "build_date" : "2018-12-17T21:17:40.758843Z",
    "build_snapshot" : false,
    "lucene_version" : "7.5.0",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

## Kibana

*kibana.yaml*

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
spec:
  selector:
    matchLabels:
      run: kibana
  template:
    metadata:
      labels:
        run: kibana
    spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:6.5.4
        env:
        - name: ELASTICSEARCH_URL
          value: http://elasticsearch:9200
        - name: XPACK_SECURITY_ENABLED
          value: "true"
        ports:
        - containerPort: 5601
          name: http
          protocol: TCP

---

apiVersion: v1
kind: Service
metadata:
  name: kibana
  labels:
    service: kibana
spec:
  type: NodePort
  selector:
    run: kibana
  ports:
  - port: 5601
    targetPort: 5601
``` 

Like before, this deployment will spin up a single Kibana pod that gets exposed via a NodePort service. Take note of the two environment variables:

1. `ELASTICSEARCH_URL` - URL of the Elasticsearch instance
2. `XPACK_SECURITY_ENABLED` - enables [X-Pack security](https://www.elastic.co/guide/en/x-pack/current/elasticsearch-security.html)

Refer to the [Running Kibana on Docker](https://www.elastic.co/guide/en/kibana/current/docker.html) guide for more info on these variables.

Create:

```
$ kubectl create -f kibana.yaml -n logging
```

Verify:

```
$ kubectl get pods -n logging

NAME                          READY   STATUS    RESTARTS   AGE
elasticsearch-bb9f879-d9kmg   1/1     Running   0          17m
kibana-7f6686674c-mjlb2       1/1     Running   0          60s


$ kubectl get service -n logging

NAME            TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
elasticsearch   NodePort   10.102.149.212   <none>        9200:30531/TCP   17m
kibana          NodePort   10.106.226.34    <none>        5601:32683/TCP   74s
```

Test this in your browser at `http://localhost:KIBANA_PORT_FORWARD`.

```
$ kubectl -n logging port-forward kibana-7f6686674c-mjlb2 7001:5601
```
![](https://raw.githubusercontent.com/jcebidanes/efk-kubernetes/6b9eb756ca8c0da325f92dc796b6abb066a6d890/assets/aa7292d4-2d90-43fe-b3b2-51e015d05bec.png)

## Fluentd

In this example, we’ll deploy a Fluentd logging agent to each node in the Kubernetes cluster, which will collect each container’s log files running on that node. We can use a DaemonSet for this.

![](https://raw.githubusercontent.com/jcebidanes/efk-kubernetes/6b9eb756ca8c0da325f92dc796b6abb066a6d890/assets/98fcee21-acea-40c8-ae81-a8c57a0537b5.png)

First, we need to configure RBAC (role-based access control) permissions so that Fluentd can access the appropriate components.

*fluentd-rbac.yaml*

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd
  namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluentd
  namespace: kube-system
rules:
  - apiGroups:
      - ""
    resources:
      - pods
      - namespaces
    verbs:
      - get
      - list
      - watch

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: fluentd
roleRef:
  kind: ClusterRole
  name: fluentd
  apiGroup: rbac.authorization.k8s.io
subjects:
  - kind: ServiceAccount
    name: fluentd
    namespace: kube-system
```

In short, this will create a ClusterRole which grants get, list, and watch permissions on pods and namespace objects. The ClusterRoleBinding then binds the ClusterRole to the ServiceAccount within the kube-system namespace. Refer to the [Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) guide to learn more about RBAC and ClusterRoles.

Create:

```
$ kubectl create -f fluentd-rbac.yaml
```

Now, we can create the DaemonSet.

*fluentd-daemonset.yaml*

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
    version: v1
    kubernetes.io/cluster-service: "true"
spec:
  selector:
    matchLabels:
      name: fluentd
  template:
    metadata:
      labels:
        name: fluentd
        k8s-app: fluentd-logging
        version: v1
        kubernetes.io/cluster-service: "true"
    spec:
      serviceAccount: fluentd
      serviceAccountName: fluentd
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1.3-debian-elasticsearch
        env:
          - name:  FLUENT_ELASTICSEARCH_HOST
            value: "elasticsearch.logging"
          - name:  FLUENT_ELASTICSEARCH_PORT
            value: "9200"
          - name: FLUENT_ELASTICSEARCH_SCHEME
            value: "http"
          - name: FLUENT_UID
            value: "0"
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

Be sure to review [Kubernetes Logging with Fluentd](https://docs.fluentd.org/) along with the [sample Daemonset](https://github.com/fluent/fluentd-kubernetes-daemonset/blob/master/fluentd-daemonset-elasticsearch-rbac.yaml). Make sure `FLUENT_ELASTICSEARCH_HOST` aligns with the `SERVICE_NAME.NAMESPACE` of Elasticsearch within your cluster.

Deploy:

```
$ kubectl create -f fluentd-daemonset.yaml
```

If you’re running Kubernetes as a single node, this will create a single Fluentd pod in the kube-system namespace.

```
$ kubectl get pods -n kube-system

coredns-576cbf47c7-mhxbp                1/1     Running   0          120m
coredns-576cbf47c7-vx7m7                1/1     Running   0          120m
etcd-minikube                           1/1     Running   0          119m
fluentd-kxc46                           1/1     Running   0          89s
kube-addon-manager-minikube             1/1     Running   0          119m
kube-apiserver-minikube                 1/1     Running   0          119m
kube-controller-manager-minikube        1/1     Running   0          119m
kube-proxy-m4vzt                        1/1     Running   0          120m
kube-scheduler-minikube                 1/1     Running   0          119m
kubernetes-dashboard-5bff5f8fb8-d64qs   1/1     Running   0          120m
storage-provisioner                     1/1     Running   0          120m
```

Take note of the logs:

```
$ kubectl logs fluentd-kxc46 -n kube-system
```

You should see that Fluentd connect to Elasticsearch within the logs:

```
Connection opened to Elasticsearch cluster =>
  {:host=>"elasticsearch.logging", :port=>9200, :scheme=>"http"}
```

To see the logs collected by Fluentd in Kibana, click “Management” and then select “Index Patterns” under “Kibana”.

![](https://raw.githubusercontent.com/jcebidanes/efk-kubernetes/6b9eb756ca8c0da325f92dc796b6abb066a6d890/assets/da417fec-57fa-441f-bfdb-1927262a9759.png)

Click the “Create index pattern” button. Select the new Logstash index that is generated by the Fluentd DaemonSet. Click “Next step”.

![](https://raw.githubusercontent.com/jcebidanes/efk-kubernetes/6b9eb756ca8c0da325f92dc796b6abb066a6d890/assets/6760f6c7-e5fa-4644-baaf-e932d0f0106f.png)

Set the “Time Filter field name” to “@timestamp”. Then, click “Create index pattern”.

![](https://raw.githubusercontent.com/jcebidanes/efk-kubernetes/6b9eb756ca8c0da325f92dc796b6abb066a6d890/assets/382bd1ac-20f8-41ac-9b19-87f962194ad2.png)

Back in “Discover” on the Kibana dashboard, add the following filter:

![](https://raw.githubusercontent.com/jcebidanes/efk-kubernetes/6b9eb756ca8c0da325f92dc796b6abb066a6d890/assets/8eb19a22-722c-41d9-a20c-58971ef3f713.png)

On this example was deployed a simple application , you should be able to see the it works log in the stream.

![](https://raw.githubusercontent.com/jcebidanes/efk-kubernetes/6b9eb756ca8c0da325f92dc796b6abb066a6d890/assets/2b4cf4c2-5bc1-4315-a340-c6a6e1edd3c6.png)

Done!


This page was initially create using the [Michael Herman](https://mherman.org/blog/logging-in-kubernetes-with-elasticsearch-Kibana-fluentd/ "Michael Herman Homepage") article.  
