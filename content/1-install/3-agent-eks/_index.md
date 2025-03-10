---
title: "EKS: Install the Sysdig Agents"
chapter: true
weight: 4
---

## Requirements

- EKS cluster (deployed in the last step [here](/0-prerequisites/3-cloud9.html)) deployment completed.
- Kubectl configured with access to the EKS cluster.
- Helm.


## Install the Sysdig Agent

The next steps will deploy the Sysdig Agent in all the nodes of
the EKS deployed during the prerequisites step.

1. Log into Sysdig Secure, and browse to
   **Integrations > Data Sources > Sysdig Agents**,
   then [**Connect a Kubernetes Cluster**](https://secure.sysdig.com/#/data-sources/agents?setupModalEnv=Kubernetes).
   Insert a cluster name of your choice (for example `aws-workshop`) and copy the resulting command.

    ![Install with Helm](/images/1-installation/agentInstall.png)

2. The **Sysdig's Admission Controller (AC) for Kubernetes** is not deployed by default.
   This component enables the [Kubernetes Audit Logging Capabilities of Sysdig Secure](https://docs.sysdig.com/en/docs/sysdig-secure/secure-events/kubernetes-audit-logging/#kubernetes-audit-logging)
   among other features (for example, vulnerability management for your k8s images).

   In your IDE, add the next parameters to the Helm command copied above
   and update the *secureAPIToken* value with your **API Token** available in
   your account [**Settings**](https://us2.app.sysdig.com/secure/#/settings/user)
   (remember to add the trailing `\` at the end of the new options):

    ```
    --set admissionController.enabled=true \
    --set admissionController.sysdig.secureAPIToken=9345f4k3-f4k3-f4k3-f4k3-f4k308c706d0 \
    --set admissionController.features.k8sAuditDetections=true \
    --set scanner.enabled=false \
    ```

3. Execute the resulting command in your terminal.
   The Sysdig Agents are being deployed now on each of the nodes of the cluster.

    ```bash
    helm repo add sysdig https://charts.sysdig.com
    helm repo update
    helm install sysdig-agent --namespace sysdig-agent --create-namespace \
        --set global.sysdig.accessKey=9345f4k3-f4k3-f4k3-f4k3-f4k308c706d0 \
        --set global.sysdig.region=eu1 \
        --set nodeAnalyzer.secure.vulnerabilityManagement.newEngineOnly=true \
        --set global.kspm.deploy=true \
        --set nodeAnalyzer.nodeAnalyzer.benchmarkRunner.deploy=false \
        --set global.clusterConfig.name=aws-workshop \
        --set admissionController.enabled=true \
        --set admissionController.sysdig.secureAPIToken=94be24b3-eaf9-4951-996d-79545cfe1089 \
        --set admissionController.features.k8sAuditDetections=true \
        --set scanner.enabled=false \
        sysdig/sysdig-deploy
    ```


## Sysdig Secure Policies

When the installation is complete, visit the [**Runtime Policies** section](https://secure.sysdig.com/#/policies)
and filter the list of policies by type: *Kubernetes Audit*.
Then, enable the *Sysdig K8s Activity Logs* and *Sysdig K8s Notable Events* policies.
These policies will alert about each and every K8s Audit Event in your EKS cluster.

![Policies](/images/1-installation/Policies.png)

After this, filter too by:
- `AWS CloudTrail` and enable all the available policies.
- `Workload` and enable all the available policies.

Create a new policy, select type `Worload Policy`.
From the rule library, add the rule: `Write below etc`
name it `aws-workshop-test-policy`, add a description `foo`
and enable Captures.


## Review installation

### Agents

Check that the _Agent_ installation was successful in the 
**Integrations > Data Sources >** [**Sysdig Agents** section](https://secure.sysdig.com/#/data-sources/agents).

![Agents](/images/1-installation/agents.png)

The EKS cluster will also be visible in the 
[**Managed Kubernetes**](https://us2.app.sysdig.com/secure/#/data-sources/managed-kubernetes)
section.

![Managedk8s](/images/1-installation/managedk8s.png)


Alternatively, you can check the logs of the agent pod:

```
kubectl logs -l app.kubernetes.io/name=agent -n sysdig-agent --tail=-1 | grep "Sending scraper version" 
```


### Admission Controller

Check that the _Admission Controller_ installation was successful by
generating an event that will be registered in the k8s API:

1. Trigger some events with:
    ```bash
    kubectl exec -n sysdig-agent \
        $(kubectl -n sysdig-agent get pod -l app=sysdig-agent \
            --output=jsonpath={.items..metadata.name} \
        | cut --delimiter " " --fields 1) -c sysdig -- ls /bin/ > /dev/null

    kubectl run nginx --image nginx --privileged
    ```

2. Then, visit the
   [**Events** section](https://secure.sysdig.com/#/events?groupBy=policy&last=86400&severities=high%2Cmedium%2Clow%2Cnone)
    and it will show up after you select the *Info* level.

    ![Event triggered](/images/1-installation/events.png)


3. Alternatively, check the admission-controller pod logs:

    ```bash
    kubectl logs -f -n sysdig-agent \
        -l app.kubernetes.io/component=webhook \
        --tail=-1 --follow=false
    ```

    Every action against the K8s API Server will generate an entry in the logs
    (based on the configured logging level).
    You should see something like this:

    ```logs
    ...
    {"level":"info","component":"console-notifier","time":"2022-12-15T15:20:11Z","message":"Pod started with privileged container (user=kubernetes-admin pod=222nginx111 ns=default images=nginx)"}
    ...
    ```