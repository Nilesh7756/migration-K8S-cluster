# migration-K8S-cluster
Migration of Kubernetes cluster deployment state


This document's primary purpose is to show how to migrate the deployment state from one Kubernetes cluster to another. 

#Dump from the source cluster

mkdir ./cluster-dump


kubectl get --export -o=json ns | \
jq '.items[] |
	select(.metadata.name!="kube-system") |
	select(.metadata.name!="default") |
	del(.status,
        .metadata.uid,
        .metadata.selfLink,
        .metadata.resourceVersion,
        .metadata.creationTimestamp,
        .metadata.generation
    )' > ./cluster-dump/ns.json
    
    
for ns in $(jq -r '.metadata.name' < ./cluster-dump/ns.json);do
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
done




#Restore to target cluster
Create the set of namespaces needed for your deployment state:

kubectl create -f cluster-dump/ns.json

Restore the resource state:

kubectl create -f cluster-dump/cluster-dump.json

