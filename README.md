# **Migration of Kubernetes cluster deployment state**


This document's primary purpose is to show how to migrate the deployment state from one Kubernetes cluster to another. 

### Dump from the source cluster

*mkdir ./cluster-dump*

First, get a list of all namespaces that are not kube-system or default and record them to a file on disk. This represents the list of namespaces that we want to migrate:

*kubectl get --export -o=json ns | \
jq '.items[] |
	select(.metadata.name!="kube-system") |
	select(.metadata.name!="default") |
	del(.status,
        .metadata.uid,
        .metadata.selfLink,
        .metadata.resourceVersion,
        .metadata.creationTimestamp,
        .metadata.generation
    )' > ./cluster-dump/ns.json*
    
    
For each of these namespaces, dump all services, controllers (rc,ds,replicaset,etc), secrets and daemonsets to a file on disk. Strip any non-portable fields from the objects. If you wish to migrate additional controller resource types (replicasets, deployments, etc), make sure to add them to the resource type list:   
    
*for ns in $(jq -r '.metadata.name' < ./cluster-dump/ns.json);do
    echo "Namespace: $ns"
    kubectl --namespace="${ns}" get --export -o=json svc,rc,secrets,ds | \
    jq '.items[] |
        select(.type!="kubernetes.io/service-account-token") |
        del(
            .spec.clusterIP,
            .metadata.uid,
            .metadata.selfLink,
            .metadata.resourceVersion,
            .metadata.creationTimestamp,
            .metadata.generation,
            .status,
            .spec.template.spec.securityContext,
            .spec.template.spec.dnsPolicy,
            .spec.template.spec.terminationGracePeriodSeconds,
            .spec.template.spec.restartPolicy
        )' >> "./cluster-dump/cluster-dump.json"
done*




### Restore to target cluster

Create the set of namespaces needed for your deployment state:

*kubectl create -f cluster-dump/ns.json*

Restore the resource state:

*kubectl create -f cluster-dump/cluster-dump.json*



*Ref* : https://coreos.com/kubernetes/docs/latest/cluster-dump-restore.html
