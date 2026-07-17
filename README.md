# Overview

In this short tutorial, we'll cover how to set Kubernetes policies across your clusters using a few common use-cases examples. You'll see a before and after of setting the policy, using an example application in a targeted namespace, proving that the policy works. Optionally, this guide will show you how to apply the policies across multiple clusters automatically, using NKP's Project concept.

The use-cases this covers:

- Prevent a pull from unapproved container registry sources
  - Justification: Centralized control of where container images lives (Harbor) gives more visibility and guardrails for Platform Operators
- Prevent `latest` tag
  - Justification: Enforce imutable images for application predictability.
- Enforce _Realistic_ Pod Disruption Budgets with multiple replicas required
  - Justification: During platform upgrades, worker nodes get replaced. Ensure application uptime.
- Required Label
  - Justification: Having an owner identified for billing and tracking purposes


# Assumptions

- NKP Management and at least one Workload Cluster has been installed
- `kubectl` access to the Workload Cluster
- `git` installed
- `podman` installed
- Internet Access

# Example App

Clone this repo

```bash
git clone https://github.com/ryandotclair/gatekeeper-policy-workshop.git
cd gatekeeper-policy-workshop
```

Let's create a specific namespace for our tests. If you happen to require a different namespace, you'll have to modify the example app (and this guides instructions).

```bash
kubectl create ns test-app
```

Every policy that we will be using, will target any namespce with the label of `policy.gatekeeper.workshop/enforce=true`

First, ensure you have no namespaces that use that label. If you do, you'll need to modify that namespaceSelector.matchLabels value for every `constraint.yaml` file in this repo.

```bash
kubectl get ns -l policy.gatekeeper.workshop/enforce=true
```

**IF IT RETURNS NOTHING** then run:
```bash
kubectl label ns test-app policy.gatekeeper.workshop/enforce=true
```

Verify ONLY the test-app namespace has that label in your cluster:
```bash
$ kubectl get ns -l policy.gatekeeper.workshop/enforce=true
NAME       STATUS   AGE
test-app   Active   5s
```

Create a copy of example-app, which we'll use for the workshop, and deploy the it. This app is a simple nginx server.
```bash
cp example-app.yaml.example example-app.yaml
```

Here's the contents of that deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-app
  namespace: test-app
  labels:
    app: example
spec:
  replicas: 1
  selector:
    matchLabels:
      app: example
  template:
    metadata:
      labels:
        app: example
    spec:
      containers:
      - name: web
        image: nginx:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 80
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
          requests:
            memory: "64Mi"
            cpu: "250m"
---
apiVersion: v1
kind: Service
metadata:
  name: example-service
  namespace: test-app
spec:
  selector:
    app: example
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
```
> Note: This assumes you have an available IP in your metalLB IP pool

As you can see, it's violating all the rules we've mentioned before:
- We're pulling from docker.io
- It's using the latest tag
- There's no PodDisruptionBudget
- It's a single replica
- There's no owner label

Let's deploy it!
```bash
kubectl apply -f example-app.yaml
```

# Gatekeeper Mental Model
Let's do a quick recap on [Gatekeeper](https://open-policy-agent.github.io/gatekeeper/website/docs/) before we dive in. There's a few things you should know.

First, Gatekeeper is an [admission controller](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/). Which means **enforcement** of these policies happen when something is created, updated, or deleted. So if you have something that was already deployed that wasn't compliant to your policies, you wouldn't see a violation until something changed.

There are two types of admission controllers that Gatekeeper supports: validating and mutating. In this workshop we only focus on validating (read: "no you can't do that"). Just know we can also do [mutating](https://open-policy-agent.github.io/gatekeeper/website/docs/mutation/), where we modify the resource as it passes through the kube-apiserver to make it compliant (ex: I deploy that 1 replica deployment above and it mutates to 2 replicas).

Gatekeeper uses two constructs with it's validating admission controller. The first is a `Constraint Template`, which defines the rules of the policy you want to create, using the `Rego` syntax. In each folder in this workshop has a `template.yaml` file, which is the the `Constraint Template` we will use for a given policy. For Platform Teams, I **HIGHLY** recommend updating each template's message (aka `msg`) to give a meaningful message back to the end user (ex: reference an internal KB article on how to be compliant, a Slack channel for help, etc).

What consumes that policy that you define as a Custom Resource is a `Constraint`. This is where you pass in the specifics (like what the label key should be, or what the registry repo name should be) as well as how it should behave when it sees a violation.
  - `deny` = Hard block the resource from being created. This is default value.
  - `warn` = Don't block it, but throw a warning to the user.
  - `dry-run` = Don't block, but note it.


# Use-Case 1: Lock Down Container Registry Source

Edit the `allowedrepos/contraint.yaml`, and add in your own details.

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: repo-is-openpolicyagent
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaceSelector:
          matchLabels:
            policy.gatekeeper.workshop/enforce: "true"
  parameters:
    repos:
      - "openpolicyagent/" # <-- This
```

In my lab environment, I have a Harbor instanced deployed on my NKP Management Cluster. I'll be using that IP address (sanitized). Mine looks like this:

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: repo-is-openpolicyagent
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaceSelector:
          matchLabels:
            policy.gatekeeper.workshop/enforce: "true"
  parameters:
    repos:
      - "10.1.2.3:5000/"
```

Note: If we did NOT specify the namespace(s) match, it would apply to all namespaces in the cluster.

## Test

Let's "manually" test this for now. Later we will flux our way to automating all the things. 

Apply the admission controller policy, and then the policy that will use said controller:

```bash
# Assumes you are in ${NKP_GIT_REPONAME}/gatekeeper-policies
kubectl apply -f allowedrepos/template.yaml

kubectl apply -f allowedrepos/constraint.yaml
```

In checking the running pods in the namespace, you'll note that they still are running:

```bash
$ kubectl get pod -n test-app
NAME                               READY   STATUS    RESTARTS   AGE
example-app-6b84485b85-kgr5c   1/1     Running   0          23m
```

As mentioned before, this is because policy happens when a change occurs (like a deployment, scaling event, or reschedule event). Let's trigger an event by deleting a pod. Because a replicaset backs it, it will attempt to recreate it. However, you'll note that it doesn't come back.

```bash
$ kubectl delete pod -n test-app example-app-6b84485b85-kgr5c

$ kubectl get pod -n test-app
No resources found in test-app namespace.
```

This is because the policy stated that the image should be coming from our specific Harbor instance. You can see this in a few places. First place is in the namespace's events:

```bash
$ kubectl get events -n test-app
.
.
96s (x17 over 7m2s)   Warning   FailedCreate        ReplicaSet/example-app-6b84485b85   Error creating: admission webhook "validation.gatekeeper.sh" denied the request: [repo-is-openpolicyagent] container <web> has an invalid image repo <nginx:latest>, allowed repos are ["10.1.2.3:5000/"]
```

You can also find this in gatekeeper's audit pod. 

```bash
$ kubectl logs -n ${WORKSPACE_NAMESPACE} gatekeeper-audit-58f7486476-j9qgl | grep test-app
{"level":"info","ts":1784150099.5924163,"logger":"controller","msg":"container <web> has an invalid image repo <nginx:latest>, allowed repos are [\"10.1.2.3:5000/\"]","process":"audit","audit_id":"2026-07-15T21:14:56Z","details":{},"event_type":"violation_audited","constraint_group":"constraints.gatekeeper.sh","constraint_api_version":"v1beta1","constraint_kind":"K8sAllowedRepos","constraint_name":"repo-is-openpolicyagent","constraint_namespace":"","constraint_action":"deny","constraint_enforcement_actions":[],"constraint_annotations":{},"resource_group":"","resource_api_version":"v1","resource_kind":"Pod","resource_namespace":"test-app","resource_name":"example-app-6b84485b85-gx5lh","resource_labels":{"app":"example","pod-template-hash":"6b84485b85"}}
```

Which you can use NKP's Grafana Logging dashboard to key on as well!

To make this compliant, let's get that pod under Harbor's management.

First retag this image and push it to your Harbor instance

```bash
echo "Pulling..."
podman pull --platform linux/amd64 nginx:latest 
echo "Tagging..."
podman tag docker.io/library/nginx ${HARBOR_LOCATION}/${PROJECT}/nginx
echo "Pushing..."
podman push ${HARBOR_LOCATION}/${PROJECT}/nginx --tls-verify=false
echo "Done"
```

Then modify the `example-app.yaml` to reference `${HARBOR_LOCATION}/${PROJECT}/nginx:latest`, where the $ values are your values. For mine it's `10.1.2.3:5000/demo/nginx:latest`

Then `kubectl apply -f example-app.yaml`

You should see in the namespace, 1 new pod!

```bash
$ kubectl get pod
NAME                           READY   STATUS    RESTARTS        AGE
example-app-74b9c6fd57-8c9qh   1/1     Running   0               45s
```

Yay! We've now enfroced good behavior, getting our container images centrally managed by Harbor as a central place for us to manage our container images and benefit from it's CVE vulnerability scanning capabilities.


# Use-Case 2: Disallow Latest

Similar as last time, let's look at the current constraint.

```bash
$ cat disallowedtags/constraint.yaml

apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sDisallowedTags
metadata:
  name: container-image-must-not-have-latest-tag
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces:
      - "test-app"
  parameters:
    tags: ["latest"]
    exemptImages: ["openpolicyagent/opa-exp:latest", "openpolicyagent/opa-exp2:latest"]
```

As you'll note, we have the ability to exclude images. But let's leave that as is and apply it.

```bash
kubectl apply -f disallowedtags/template.yaml
kubectl apply -f disallowedtags/constraint.yaml
```

## Test
And like before,  you'll note the running pods aren't impacted

```bash
$ kubectl get po
NAME                           READY   STATUS    RESTARTS      AGE
example-app-74b9c6fd57-8c9qh   1/1     Running   0             15m
```

... And like before if you delete it, it won't come back.

```bash
$ kubectl delete pod example-app-74b9c6fd57-8c9qh -n test-app
pod "example-app-74b9c6fd57-ffxz6" deleted from test-app namespace

$ kubectl get pod -n test-app
No resources found in test-app namespace.
```

This is because of the latest tag we are using.

```bash
$ kubectl get events -n test-app
.
.
2m1s        Warning   FailedCreate        replicaset/example-app-74b9c6fd57   Error creating: admission webhook "validation.gatekeeper.sh" denied the request: [container-image-must-not-have-latest-tag] container <web> uses a disallowed tag <10.1.2.3:5000/demo/nginx:latest>; disallowed tags are ["latest"]
```

Let's fix it!

```bash
podman tag docker.io/library/nginx ${HARBOR_LOCATION}/${PROJECT}/nginx:v1
podman push ${HARBOR_LOCATION}/${PROJECT}/nginx:v1 --tls-verify=false
```

And then update the `example-app.yaml` file to reference the v1 tag (example: `image: 10.1.2.3:5000/demo/nginx:v1`), apply it.

```bash
kubectl apply -f example-app.yaml
```

You'll notice immediately a new pod.

```
$ kubectl get pod -n test-app
NAME                           READY   STATUS    RESTARTS   AGE
example-app-694978478c-t9k8h   1/1     Running   0          1s
```

Yay! We've now enforced good behavior, ensuring that all our images are tagged with uniqueness, such that we have immutable images for predictable deployments, especially as you promote your code from laptop, to dev, to qa/pre-prod, and finally to prod. Same image, same predictable and repeatable behavior.

# Use-Case 3: Enforce Pod Disruption Budget
As a quick recap on Pod Disruption Budget, this concept (which I highly recommend reading up on [disruptions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/) doc and this [page](https://kubernetes.io/docs/tasks/run-application/configure-pdb/) on it) is all about ensuring the application (pods) has an allocated disruption budget, such that Kubernetes is aware of what the app can handle. This is especially important for planned disruptions (like rolling upgrades).

This particular Constraint Template has a number of rules in it. Specifically it's look at ensuring:
- Your deployment has a PodDisruptionBudget tied to it
- Your deployment replicas isn't set to 1 (defeats the purpose)
- The PDB isn't 0 (defeats the purpose)
- The PDB value isn't greater than or equal to Deployment's replica count (defeats the purpose)

Go ahead and apply the template and constraints.

```bash
kubectl apply -f poddisruptionbudget/template.yaml
kubectl apply -f poddisruptionbudget/constraint.yaml
```

Our rules we are using, however, require an understanding of object relationships (and direction). To do this in Gatekeeper, I'm using the `data.inventory` store that Gatekeeper can sync from etcd. By default, nothing is captured. You'll need to configure Gatekeeper to sync the PDB data to that store. However, where to deploy this config map will be dependent on where Gatekeeper is running. On an NKP Workload Cluster, Gatekeeper runs in the Workspace Namespace. 

Modify the `poddisruptionsbudget/config.yaml` file and update the `namespace: gatekeeper-system` to reflect your namespace. 
> Note: Fastest way is to run `kubectl get po -A | grep gatekeeper` on your cluster to find the namespace.

**IMPORTANT**: _Before_ you deploy this config file, make sure it doesn't already exist. 

```bash
$ kubectl get -f poddisruptionbudget/config.yaml
Error from server (NotFound): configs.config.gatekeeper.sh "config" not found
```

If your's does NOT look like the above, you'll need to edit the `configs.config.gatekeeper.sh/config` object and merge the below into it:
```
sync:
    syncOnly:
      - group: "policy"
        version: "v1"
        kind: "PodDisruptionBudget"
```

Go ahead and apply the config.yaml file once you've validated it doesn't exist and you've updated the namespace to reflect where gatekeeper pods are running in:

```bash
kubectl apply -f poddisruptionbudget/config.yaml
```

## Test
Here we are going to put this one through it's paces in a number of scenarios. This one can really highlight the flexibility of these policies. First, delete the deployment (as the rules are tied to the deployment).

```bash
kubectl delete deploy example-app -n test-app

kubectl apply -f example-app.yaml
```

You should see this message:
```
Error from server (Forbidden): error when creating "example-app.yaml": admission webhook "validation.gatekeeper.sh" denied the request: [pod-distruption-budget] Deployment <example-app> is missing a mandatory PodDisruptionBudget. You must deploy a PDB with matching selector labels in namespace <test-app>.
```

Awesome. What we're going to do is test this in a number of ways. First up, let's apply a PDB with `maxUnavailable: 0 ` (or in other words, "I don't allow any interuption"). We don't want this because we _should_ be allowing interruptions. 
```bash
kubectl apply -f poddisruptionbudget/bad.yaml
```

You should see this:
```
Error from server (Forbidden): error when creating "poddisruptionbudget/bad.yaml": admission webhook "validation.gatekeeper.sh" denied the request: [pod-distruption-budget] PodDisruptionBudget <my-example-app-pdb> has maxUnavailable of <0>, which allows 0 voluntary disruptions and will block node drains / cluster upgrades
```

Cool. Next up, let's use a PDB that has maxUnavailable of 1, _however_ it's not associated with the Deployment
```bash
kubectl apply -f poddisruptionbudget/bad2.yaml
```

That went through, because it's valid. What's not valid is if we do a deployment right now that doesn't reference that PDB. 

You'll note that we don't actually have a deployment right now.

This is because the Admission Controller had blocked the deployment earlier (but not the service) when you did the `kubectl apply -f example-app.yaml`.

```bash
$ kubectl get all -n test-app
NAME                      TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)        AGE
service/example-service   LoadBalancer   10.111.123.157   10.1.2.10   80:31052/TCP   29m
```

Go ahead and re-apply it. 
```bash
kubectl apply -f example-app.yaml
```

And there we are.
```
Error from server (Forbidden): error when creating "example-app.yaml": admission webhook "validation.gatekeeper.sh" denied the request: [pod-distruption-budget] Deployment <example-app> (2 replica(s)) is selected by PodDisruptionBudget <my-example-app-pdb> (minAvailable=<unset>, maxUnavailable=<2>). Require 1 <= disruptionsAllowed <= replicas-1 so drains can proceed without taking down every pod.
```

Cool. That works. Let's apply the good PDB, which has both `maxUnavailable: 1` and the right matching labels to our Deployment, and re-apply again. 

```bash
kubectl apply -f poddisruptionbudget/good.yaml
kubectl apply -f example-app.yaml
```

You should see this error:
```
Error from server (Forbidden): error when creating "example-app.yaml": admission webhook "validation.gatekeeper.sh" denied the request: [pod-distruption-budget] Deployment <example-app> (1 replica(s)) is selected by PodDisruptionBudget <my-example-app-pdb> (minAvailable=<unset>, maxUnavailable=<1>). Require 1 <= disruptionsAllowed <= replicas-1 so drains can proceed without taking down every pod.
[pod-distruption-budget] Deployment <example-app> has 1 replica(s); at least 2 replicas are required so a PodDisruptionBudget can allow node drains / cluster upgrades without taking down every pod
```
Now this is slightly different. We have a PDB, which is pointed at our deployment, but because we are using a replica of 1, this isn't ideal. This is because the app can experience downtime right now with just 1 replica. We want to enforce redundancy.

Go ahead and modify the `example-app.yaml` to use `replicas: 2` and deploy it.

```bash
$ kubectl apply -f example-app.yaml
deployment.apps/example-app created
service/example-service unchanged
```

Success!

Let's do one more test. This time, lets use `minAvailable: 2`, aka "the minimum of replicas I want always up is 2", which is the same number of replicas our Deployment has (read: similar as maxUnavailable = replica earlier). 

```bash
kubectl apply -f poddisruptionbudget/bad3.yaml
```

You should see this:
```
Error from server (Forbidden): error when creating "poddisruptionbudget/bad3.yaml": admission webhook "validation.gatekeeper.sh" denied the request: [pod-distruption-budget] PodDisruptionBudget <my-example-app-pdb> (minAvailable=<2>, maxUnavailable=<unset>) is unsafe for 2 matching replica(s). Require replicas >= 2 and 1 <= disruptionsAllowed <= replicas-1 so drains can proceed without taking down every pod.
```

Awesome sauce. So we now are protecting our app teams from doing something they'll regret later. This really leans into the concept of shared responsibilty model, and what I like to call "social contracts". I'm a big fan of setting the expectation up front that at any moment in time, a Worker Node can go away and be replaced. If you haven't designed the app to handle that scenario and have the right resilency settings in place, then everyone suffers. 


# Use-case 4: Required Labels
A common use-case is ensuring all namespaces have an owner label (or owner annotation). In this case we're going to use the label method. This helps with tracking who owns what, and while you could do this at the deployment level, generally namespace level is enough.

As you'll note in the `requiredlabels/constraint.yaml` file, not only are we looking for the owner label, but also a specific shape (ex: ryan.company.local)

```yaml
    labels:
      - key: owner
        allowedRegex: "^[a-zA-Z]+.company.local$"
```

Go ahead and apply as is:
```bash
kubectl apply -f requiredlabels/template.yaml
kubectl apply -f requiredlabels/constraint.yaml
```

## Test
Given this is a workshop, and we've narrowed the policy enforcement based on a given namespace label, we'll require passing in that label for Gatekeeper to act on it. Let's try a failure scenario to see what happens:
```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: my-new-namespace
  labels:
    policy.gatekeeper.workshop/enforce: "true"
EOF
```

As you'll note, we get the missing owner.
```
Error from server (Forbidden): error when creating "STDIN": admission webhook "validation.gatekeeper.sh" denied the request: [all-must-have-owner] All namespaces must have an `owner` label that points to your company username (me.company.local)
```

Let's try an owner but not using the proper syntax:
```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: my-new-namespace
  labels:
    policy.gatekeeper.workshop/enforce: "true"
    owner: "ryan"
EOF
```

Yup, we get the same error.
```
Error from server (Forbidden): error when creating "STDIN": admission webhook "validation.gatekeeper.sh" denied the request: [all-must-have-owner] All namespaces must have an `owner` label that points to your company username (me.company.local)
```

Last but not least, we add in a proper username.
```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: my-new-namespace
  labels:
    policy.gatekeeper.workshop/enforce: "true"
    owner: "ryan.company.local"
EOF
```

And you should see...
```
namespace/my-new-namespace created
```

Success!

Note: an alternative type of namespaceSelector option you can use (if this wasn't in a workshop environment) is something like "run the policy only if this label doesn't exist".
```yaml
spec:
  match:
    namespaceSelector:
      matchExpressions:
        - key: policy.gatekeeper.workshop/skip
          operator: DoesNotExist
```

Now you can always know who to reach out to when there's an issue. This policy is especially handy when Platform Teams give direct kubernetes access to dev teams. Having a clearly defined owner for every namespace, ideally at the team or email distribution list level (in case an individual moves on) allows the Platform team to always know who to reach out to when needed. This can also help other downstream systems that might benefit from this like chargeback, observability dashboards, oncall, etc. 

# Automate All The Things!
OK, so we've successfully tested each policy individually, now let's roll this out to a whole fleet of Kubernetes clusters. 

We can do this by using the NKP Project concept. 

Create a new NKP Project, I'll call mine "admin-policies", and I'll also override the namespace to match that.

Note: You can further automate this by dynamically applying this Project to any Kubernetes cluster that has been given a label (example: `environment=production`), which opens up interesting use-cases.

Next you'll want to fork my repository, update (or just copy over all your constraint files over) it, git/push those changes, and then reference your github repo in your Project's Continuous Deployment tab. We leverage FluxCD, which will pick up the `kustomization.yaml` file and apply each of those policies that it references (you can comment the lines you don't want to apply) to any cluster the Project lives on. And in this case, it will act on any namespace with the `policy.gatekeeper.workshop/enforce: "true"` label.

(▀̿Ĺ̯▀̿ ̿)

So cool.

# Clean Up

If you did the GitOps method, you'll just need to delete the GitOps source, and it will clean everything up from all the clusters it was deployed on. 

For the manual method, a simple way to clean it all up is using the -R flag. First make sure it deletes everything you'd expect
```bash
cd ..
kubectl delete -f ./gatekeeper-policy-workshop -R --dry-run=client
```

Then delete
```bash
kubectl delete -f ./gatekeeper-policy-workshop -R
```

Last but not least clean up the two namespaces
```bash
kubectl delete ns test-app
kubectl delete ns my-new-namespace
```

# Resources

I'd like to give a special shout out to the [Gatekeeper Library](https://github.com/open-policy-agent/gatekeeper-library). They've got a plethera of many other examples, and I heavily borrowed from this repo. 