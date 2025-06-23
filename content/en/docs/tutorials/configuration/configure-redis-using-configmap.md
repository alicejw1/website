---
reviewers:
- alicejw1
- austin.shaefer
title: Configure Redis using a ConfigMap
content_type: tutorial
weight: 30
---

<!-- overview -->

<<!--alicejw1 - introduce what this tutorial accomplishes - what it does/doesn't do and who should read it. -->
This tutorial is the second


This page provides a real world example of how to configure Redis using a ConfigMap and builds upon the [Configure a Pod to Use a ConfigMap](/docs/tasks/configure-pod-container/configure-pod-configmap/) task. 

To keep your app configuration separate from the image content, you need to create a ConfigMap. 
To learn more about ConfigMaps, see: [ConfigMaps](/docs/concepts/configuration/configmap/).

## {{% heading "objectives" %}}


* Create a ConfigMap with Redis configuration values
* Create a Redis Pod that mounts and uses the created ConfigMap
* Verify that the configuration was correctly applied.



## {{% heading "prerequisites" %}}


{{< include "task-tutorial-prereqs.md" >}} {{< version-check >}}

* The example shown on this page works with `kubectl` 1.14 and above.
* Understand [Configure a Pod to Use a ConfigMap](/docs/tasks/configure-pod-container/configure-pod-configmap/).

<!-- lessoncontent -->

## Example: Configure Redis using a ConfigMap

To configure a Redis cache using data stored in a ConfigMap, you need to do a few configurations. Create the configuration block in your YAML file. Apply 

### Step 1: Create a ConfigMap with an empty configuration block

Add the following information to your config.yaml file:

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

Run the following command to look at the 'example-redis-config' description:

```shell
kubectl describe configmap/example-redis-config
```

### Step 4: Verify the current configuration

Run `kubectl exec` with the `redis-cli` tool to check the current configuration in the POD:

```shell
kubectl exec -it redis -- redis-cli
```

Verify that the maximum memory `maxmemory` is set to zero. From your local host, run 'CONFIG GET maxmemory' as follows:

```shell
127.0.0.1:6379> CONFIG GET maxmemory
```

It should return the zero value for 'maxmemory':

```shell
1) "maxmemory"
2) "0"
```

Next, verify the `maxmemory-policy` value. Run the following command:

```shell
127.0.0.1:6379> CONFIG GET maxmemory-policy
```
You should see its default value `noeviction`, as follows:

```shell
1) "maxmemory-policy"
2) "noeviction"
```

### Step 5: Add configuration values to the ConfigMap

Run the following command to apply the updated ConfigMap:

```shell
kubectl apply -f example-redis-config.yaml
```

You should see configurations in the `example-redis-config` similar to the following:

{{% code_sample file="pods/config/example-redis-config.yaml" %}}

Run `kubectl describe` to confirm that the ConfigMap was updated. 

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

### Step 6: Verify configurations are applied to the POD
<!-- alice left off here....need to rewrite the rest of the file

Check the Redis Pod again using `redis-cli` via `kubectl exec` to see if the configuration was applied:

```shell
kubectl exec -it redis -- redis-cli
```

Check `maxmemory`:

```shell
127.0.0.1:6379> CONFIG GET maxmemory
```

It remains at the default value of 0:

```shell
1) "maxmemory"
2) "0"
```

Similarly, `maxmemory-policy` remains at the `noeviction` default setting:

```shell
127.0.0.1:6379> CONFIG GET maxmemory-policy
```

Returns:

```shell
1) "maxmemory-policy"
2) "noeviction"
```

The configuration values have not changed because the Pod needs to be restarted to grab updated
values from associated ConfigMaps. Let's delete and recreate the Pod:

```shell
kubectl delete pod redis
kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
```

Now re-check the configuration values one last time:

```shell
kubectl exec -it redis -- redis-cli
```

Check `maxmemory`:

```shell
127.0.0.1:6379> CONFIG GET maxmemory
```

It should now return the updated value of 2097152:

```shell
1) "maxmemory"
2) "2097152"
```

Similarly, `maxmemory-policy` has also been updated:

```shell
127.0.0.1:6379> CONFIG GET maxmemory-policy
```

It now reflects the desired value of `allkeys-lru`:

```shell
1) "maxmemory-policy"
2) "allkeys-lru"
```

Clean up your work by deleting the created resources:

```shell
kubectl delete pod/redis configmap/example-redis-config
```

## {{% heading "whatsnext" %}}


* Learn more about [ConfigMaps](/docs/tasks/configure-pod-container/configure-pod-configmap/).
* Follow an example of [Updating configuration via a ConfigMap](/docs/tutorials/configuration/updating-configuration-via-a-configmap/).
