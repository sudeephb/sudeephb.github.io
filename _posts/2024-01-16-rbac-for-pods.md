---
title: "Kubernetes - Access control, but for pods"
date: 2024-01-16T09:13:30+05:45
categories:
  - Kubernetes
---

If you've ever used Linux, you've probably used, or at least heard of, RBAC. RBAC stands for Role Based Access Control, and it provides exactly that.

It's intuitive to think of RBAC as allowing or restricting users to do stuff. But, what if you wanted a Kubernetes pod to be able to do something on the Kubernetes cluster. What if you wanted it to be able to not do some things? Kubernetes RBAC also provides features for it.

First, you'd need to create a role that defines what the pod should be able to do. Whatever you don't explicitly specify will be restricted. 

Let's create a role named `scholar-role` in the existing namespace `library` that allows the pods to see information about the available pods. Let's also give the pod ability to see other pod's logs in the same namespace.

```
# scholar-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: scholar-role
  namespace: library
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - pods/log
  verbs:
  - list
  - get

```

Let's apply the configuration to create the role:

```
kubectl apply -f ./scholar-role.yaml
```

Now, for that role to be used, you have to create a rolebinding that binds the role to the relevant account. But wait, we're doing this for a pod. Unlike users, pods don't have their own accounts that they use to login to the system. Now that's where `ServiceAccounts` come in. 

Let's create a ServiceAccount named `scholar-account` in the same namespace `library`

```
# scholar-account.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: scholar-account
  namespace: library

```

Let's apply the configuration to create the ServiceAccount:
```
kubectl apply -f ./scholar-account.yaml
```

Now, we can assign this ServiceAccount to any pod we want. Let's create a couple of pods -- `scholar-pod` and `dumb-pod` in the same namespace `library` . We will assign `scholar-account` ServiceAccount to the `scholar-pod` pod, and no ServiceAccount to the `dumb-pod`. They'll both use the `nginx` image, but that's irrelevant here, the image used doesn't make any difference.
```
# pods.yaml
apiVersion: v1
kind: Pod 
metadata:
  name: scholar-pod
  namespace: library
spec:
  serviceAccountName: scholar-account
  containers:
  - image: nginx
    name: scholar-pod

---
apiVersion: v1
kind: Pod 
metadata:
  name: dumb-pod
  namespace: library
spec:
  containers:
  - image: nginx
    name: dumb-pod

```

Let's apply the configuration to create the pods:
```
kubectl apply -f ./pods.yaml
```

At this point, we are ready to `bind` our ServiceAccount to the role we created. Let's create a `RoleBinding` that binds the role `scholar-role` to ServiceAccount `scholar-account`:

```
# scholar-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: scholar-rolebinding
  namespace: library
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: scholar-role
subjects:
- kind: ServiceAccount
  name: scholar-account
  namespace: library

```

Let's apply the configuration to create the RoleBinding:
```
kubectl apply -f ./scholar-binding.yaml
```

We are done! But how do we make sure things work as they should. Precisely, how do we make sure that the `scholar-pod` can read the pods and pods' logs, and that `dumb-pod` cannot do that.

One way we could verify the permissions is by calling the Kubernetes API to perform the read operations from each of the pods. But that's too much work just to verify that our ServiceAccounts work as expected. Instead, we can use the `can-i` subcommand provided by `kubectl auth` to test the permissions. 

Let's see if our `scholar-account` ServiceAccount can read the pod logs in the `library` namespsce.
```
kubectl auth can-i get pods --subresource=log --as=system:serviceaccount:library:scholar-account -n library
yes
```

Well, `yes`, as expected. 

While we're at it, let's also see if the ServiceAccount can list the pods themselves:
```
kubectl auth can-i list pods --as=system:serviceaccount:library:scholar-account -n library
yes
```

As expected. But, can the ServiceAccount do what it's not supposed to, say deleting pods. Let's check that:
```
kubectl auth can-i delete pods --as=system:serviceaccount:library:scholar-account -n library
no
```

`no` is the exact result we were looking for. Perfect!
