---
reviewers:
- alicejw1
- austinshaefer
title: Configure Redis with a ConfigMap
content_type: tutorial
weight: 30
---

<!-- overview: introduce what this tutorial accomplishes - what it does/doesn't do and who should read it. -->

In this tutorial we show you how to set up your ConfigMap to use Redis (Remote Dictionary Server). Redis stores data in RAM to deliver fast access to data. To learn more about Redis, see the [Redis](https://redis.io/about/) documentation. 
<!-- alicejw1 suggests to write a concepts guide about Redis, and include a link to that doc in this tutorial:-->

## {{% heading "objectives" %}}

* Create a ConfigMap with Redis configuration values. To learn more about ConfigMaps, see: [ConfigMaps](/docs/concepts/configuration/configmap/).
* Create a Redis Pod that mounts and uses the created ConfigMap. To learn more about using Redis with Kubernetes, see: [Understanding Redis data store](docs/concepts/redis). 
* Verify that the Redis configuration was correctly applied.

<!-- This section is duplicate content that is repeated from the prerequisite Task in Configure Pods and Containers. See https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#before-you-begin. 
## {{% heading "prerequisites" %}}

{{< include "task-tutorial-prereqs.md" >}} {{< version-check >}}
-->
* Make sure you are using `kubectl` version 1.14 or any later versions.
* You need to create a ConfigMap before configuring Redis. See the following Task: [Create a ConfigMap](/docs/tasks/configure-pod-container/configure-pod-configmap/#create-a-configmap).

<!-- lessoncontent -->

## How to configure Redis using a ConfigMap

To configure a Redis cache using data stored in a ConfigMap, you need to modify the `config.yaml` file. 

### Step 1: Create a ConfigMap with an empty configuration block

Create a new ConfigMap, for example, `example-redis-config.yaml` file, and add an empty data block for `redis-config`. The following example shows an empty configuration block:

```shell
cat <<EOF >./example-redis-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-redis-config
data:
  redis-config: ""
EOF
```

### Step 2: Add the Redis POD manifest 

You need to explicitly apply the updated config.yaml file and the Redis pod manifest using the 'kubectl' CLI. To do so, run the following command in a shell script:

```shell
kubectl apply -f example-redis-config.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
```

After you apply the updated config and manifest YAML files, you should see several changes in the redis-pod.yaml manifest file:

* `spec.volumes[1]` creates a new volume `config` that is added to the `/redis-master` volumeMount.
* The `redis-config` key is contained in the `redis.conf` file in the `config` volume under the configMap `example-redis-config`.
* The data is now exposed inside the Pod in `data.redis-config` from the `example-redis-config`
ConfigMap as `/redis-master/redis.conf`.

The Redis POD YAML file should now look like this:

{{% code_sample file="pods/config/redis-pod.yaml" %}}

### Step 3: Examine the created objects

Run the following command to get and open the 'example-redis-config' ConfigMap:

```shell
kubectl get pod/redis configmap/example-redis-config 
```

Verify that you see the following output:

```
NAME        READY   STATUS    RESTARTS   AGE
pod/redis   1/1     Running   0          8s

NAME                             DATA   AGE
configmap/example-redis-config   1      14s
```

Run the following command to look at the 'example-redis-config' description:

```shell
kubectl describe configmap/example-redis-config
```

{{< note >}}
Because we left the `redis-config` key in the `example-redis-config` ConfigMap blank, you should see an empty `redis-config` key that looks similar to the following example.
{{< /note >}}

```shell
Name:         example-redis-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
redis-config:
```

### Step 4: Verify the current configuration

Run `kubectl exec` with the `redis-cli` tool to check the current configuration in the POD:

```shell
kubectl exec -it redis -- redis-cli
```

Verify that the maximum memory `maxmemory` is set to zero. From your local host, run the following command:

```shell
127.0.0.1:6379> CONFIG GET maxmemory
```

It should return the following zero value for `maxmemory`:

```shell
1) "maxmemory"
2) "0"
```

Verify the `maxmemory-policy` value is set to the default `noeviction`. Run the following command:

```shell
127.0.0.1:6379> CONFIG GET maxmemory-policy
```
It should return the following default `noeviction`:

```shell
1) "maxmemory-policy"
2) "noeviction"
```

### Step 5: Add configuration values to the ConfigMap

Run the following command to apply the updated ConfigMap:

```shell
kubectl apply -f example-redis-config.yaml
```

You should see configurations in the `example-redis-config` similar to the following example YAML file:

{{% code_sample file="pods/config/example-redis-config.yaml" %}}

Run `kubectl describe` to confirm that the ConfigMap was updated: 

```shell
kubectl describe configmap/example-redis-config
```

You should see the following configuration values now in the file:

```shell
Name:         example-redis-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
redis-config:
----
maxmemory 2mb
maxmemory-policy allkeys-lru
```

### Step 6: Verify configurations are applied to the Pod
<!-- alice left off here....need to rewrite the rest of the file -->

Verify that the configuration was applied to the Redis Pod using Redis CLI and `kubectl exec`. Run the following command:

```shell
kubectl exec -it redis -- redis-cli
```

Verify that `maxmemory` is still set to the default zero value. Run the following command:

```shell
127.0.0.1:6379> CONFIG GET maxmemory
```
You should see the following entry for `maxmemory`:

```shell
1) "maxmemory"
2) "0"
```

Run the following command to verify that `maxmemory-policy` remains at the `noeviction` default setting:

```shell
127.0.0.1:6379> CONFIG GET maxmemory-policy
```

You should see `noeviction` value for `maxmemory-policy`:

```shell
1) "maxmemory-policy"
2) "noeviction"
```

### Step 7: Delete and recreate the Pod

The configuration values have not changed because the Pod needs to be restarted to grab updated
values from the associated ConfigMap. Delete and recreate the Pod as follows:

```shell
kubectl delete pod redis
kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
```

Now check the configuration values one last time:

```shell
kubectl exec -it redis -- redis-cli
```

Look at the `maxmemory` value by running the following command:

```shell
127.0.0.1:6379> CONFIG GET maxmemory
```

It should return the updated value `2097152`:

```shell
1) "maxmemory"
2) "2097152"
```

Look at the `maxmemory-policy` value that was updated. Run:

```shell
127.0.0.1:6379> CONFIG GET maxmemory-policy
```

It should show the `allkeys-lru` value as follows:

```shell
1) "maxmemory-policy"
2) "allkeys-lru"
```
As a best practice to clean up your work, run the following command to delete the created resources:

```shell
kubectl delete pod/redis configmap/example-redis-config
```

## {{% heading "whatsnext" %}}

* Learn about ConfigMaps mounted as a Volume. See: [Update a ConfigMap mounted as a Volume](/docs/tutorials/configuration/updating-configuration-via-a-configmap/#rollout-configmap-volume).
* Learn about Pod environment variables. See: [Update Pod environment variables with a ConfigMap ](/docs/tutorials/configuration/updating-configuration-via-a-configmap/#rollout-configmap-env).
* Learn about multi-container Pods. See: [Update ConfigMap configurations in a multi-container Pod](/docs/tutorials/configuration/updating-configuration-via-a-configmap/#rollout-configmap-multiple-containers).
* Learn about sidecar containers in Pods. See: [Update configurations for a Pod with a sidecar container](/docs/tutorials/configuration/updating-configuration-via-a-configmap/#rollout-configmap-sidecar).
