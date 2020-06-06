## Selecting Nodes for Jupyter Noterbook Pods

It might be important for your use case to be able to select a specific node or a set of nodes where a specific Jupyter Notebook images for specific users would get deployed. This tutorial shows you how to achieve that.

First, we will want to taint and label the node. We will pick the first worker node and add a taint and a label `my_memory_heavy_node`:

Get a node name:

```
NODE=$(oc get nodes | grep worker | head -1 | awk '{print $1}')
```

Let'see it's taints and labels there are any taints:

```
oc describe node ${NODE}
```

You should see `Taints: <none>` in the output and set of default lables.

It would be good to see if there are any pods running on the node:

```
oc get pods --all-namespaces -o wide --field-selector spec.nodeName=${NODE}
```

Now let's add the taint to make sure no additional containers get scheduled to the node unless we specifically allow that

```
oc adm taint nodes ${NODE} my_memory_heavy_node=yes:NoSchedule
```

Next let's add a label to the node to make sure we can use it to explicitly request our containers to be deployed on this node

```
oc label node ${NODE} my_memory_heavy_node="true"
```

Let's check the node info again:

```
oc describe node ${NODE}
```

At this point we have the node ready - no pod will be scheduled on the node unless it has the `my_memory_heavy_node:true:NoSchedule` node tolerations. By using the right profile we will now deploy a Jupyter Notebook container on this specific node. Adding the toleration and setting node affinity to make sure the pod is only schedulable on that node. As you can see in the file ./profiles/02-more-profiles.configmap.yaml, the profile `node tolerations` defines the toleration and affinity exactly as we set above, so when we deploy a `memory_heavy_notebook:latest`, it will and up on the given node.


First, make sure the image is available in the cluster

```
oc apply -f images/memory_heavy_notebook.imagestream.yaml
```

Then upload the profiles configuration

```
oc apply -f profiles/02-more-profiles.configmap.yaml
```

Let's create new rollout to make sure JupyterHub picked up all the changes

```
oc rollout latest jupyterhub
```

If you head over to JupyterHub now, you should see the `memory_heavy_notebook:latest` in the image list. Select it and click `Spawn`.

When you list all the pods running on the tainted node, you should see your notebook pod there.

```
oc get pods --all-namespaces -o wide --field-selector spec.nodeName=${NODE}
```

You will probably want to remove the taint at the end to make sure the node is fully used again.

```
oc adm taint nodes ${NODE} my_memory_heavy_node-
```