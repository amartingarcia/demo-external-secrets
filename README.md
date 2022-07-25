# External Secrets + Stakater/Reloader

[Kubernetes External Secrets](https://github.com/external-secrets/kubernetes-external-secrets) allows you to use external secret management systems, like AWS Secrets Manager or HashiCorp Vault, to securely add secrets in Kubernetes.

## External Secrets
1. The controller will need permissions to access AWS Secrets Manager. [How to grant permissions](https://github.com/external-secrets/kubernetes-external-secrets#aws-based-backends).

2. Create secrets in AWS Secret Manager:
```bash
$ aws secretsmanager create-secret --name credentials-test --secret-string '{"username":"user","password":"pass"}' >/dev/null
$ aws secretsmanager create-secret --name kv/extsec/secret1 --secret-string '{"intKey": 11,"objKey": {"strKey": "hello world"}}' >/dev/null
```

3. Install External Secrets:
```bash
$ helm repo add external-secrets https://external-secrets.github.io/kubernetes-external-secrets/
$ helm install external-secrets -n external-secrets --create-namespace --debug --wait external-secrets/kubernetes-external-secrets
```

4. Show resource controller:
```bash
$ kubectl -n external-secrets get po

NAME                                                           READY   STATUS    RESTARTS   AGE
external-secrets-kubernetes-external-secrets-d79446c65-2dlfw   1/1     Running   0          2m
```

5. Create resource:
```bash
$ kubectl apply -f manifests/
```

6. Get resources:
```bash
# Get externalsecrets resources
$ kubectl -n external-secrets get externalsecrets.kubernetes-client.io

NAME               LAST SYNC   STATUS    AGE
credentials-test   3s          SUCCESS   1m
tmpl-ext-sec       4s          SUCCESS   1m

# Get secrets resources
$ kubectl -n external-secrets get secrets

NAME                 TYPE          DATA   AGE
credentials-test     Opaque        1      2m
tmpl-ext-sec         Opaque        2      2m
```

7. Decode secret:
```bash
$ kubectl -n external-secrets get secret credentials-test --template {{.data.password}} | base64 -d

pass
```

8. Modify the AWS Secret Manager resource and decode the Kubernetes secret again:
```bash
# Modify secret
$ aws secretsmanager put-secret-value --secret-id credentials-test --secret-string '{"username":"user","password":"pass1"}' >/dev/null

# Get external secret resource
$ kubectl -n external-secrets get externalsecrets.kubernetes-client.io

NAME               LAST SYNC   STATUS    AGE
credentials-test   3s          SUCCESS   10m
tmpl-ext-sec       3s          SUCCESS   10m

# Decode secret
$ kubectl -n external-secrets get secret credentials-test --template {{.data.password}} | base64 -d

pass1
```

## Stakater/Reloader

### Problem:
We would like to watch if some change happens in ConfigMap and/or Secret; then perform a rolling upgrade on relevant DeploymentConfig, Deployment, Daemonset, Statefulset and Rollout.

### Solution:
[Reloader](https://github.com/stakater/Reloader) can watch changes in ConfigMap and Secret and do rolling upgrades on Pods with their associated DeploymentConfigs, Deployments, Daemonsets Statefulsets and Rollouts.

1. Deploy Stakater/Reloader:
```bash
$ helm repo add stakater https://stakater.github.io/stakater-charts
$ helm repo update
$ helm install reloader -n reloader --create-namespace --debug --wait stakater/reloader
```

2. Uncomment the annotation in the deployment file and deploy:
```yaml
...
metadata:
  annotations:
    reloader.stakater.com/auto: "true"
  name: nginx
...
```

3. Change your secret in AWS Secret manager and decode again the secret:
```bash
# Change secret
$ aws secretsmanager put-secret-value --secret-id credentials-test --secret-string '{"username":"user","password":"password1234"}' >/dev/null

# Get external secret resource
$ kubectl -n external-secrets get externalsecrets.kubernetes-client.io

NAME               LAST SYNC   STATUS    AGE
credentials-test   3s          SUCCESS   10m
tmpl-ext-sec       3s          SUCCESS   10m

# Decode secret
$ kubectl -n external-secrets get secret credentials-test --template {{.data.password}} | base64 -d

password1234

# Show reloader logs
$ kubectl -n tools logs reloader-reloader-65888d9cd9-p8g8x

time="2021-10-21T06:08:52Z" level=info msg="Environment: Kubernetes"
time="2021-10-21T06:08:52Z" level=info msg="Starting Reloader"
time="2021-10-21T06:08:52Z" level=warning msg="KUBERNETES_NAMESPACE is unset, will detect changes in all namespaces."
time="2021-10-21T06:08:52Z" level=info msg="Starting Controller to watch resource type: configMaps"
time="2021-10-21T06:08:52Z" level=info msg="Starting Controller to watch resource type: secrets"
time="2021-10-21T07:46:24Z" level=info msg="Changes detected in 'credentials-test' of type 'SECRET' in namespace 'external-secrets'"
time="2021-10-21T07:46:24Z" level=info msg="Updated 'nginx' of type 'Deployment' in namespace 'external-secrets'"

# Check de Pod status
$ kubectl -n external-secrets get po

NAME                                                           READY   STATUS        RESTARTS   AGE
external-secrets-kubernetes-external-secrets-d79446c65-2dlfw   1/1     Running       0          3h24m
nginx-asd518qwe-512fa                                          0/1     Terminating   0          1h
nginx-76899c979-625fl                                          1/1     Running       0          26s
```
