# Automatically (idempotently) creating the ArgoCD application with Tekton

## How to Use

```shell
kubectl apply -f https://raw.githubusercontent.com/codecentric/tekton-catalog/main/task/argocd-task-create-sync-wait/0.4/argocd-task-create-sync-wait.yml
```

Use it in your Tekton Pipeline like this:

```yaml
    - name: argo-create-app-sync-wait
      taskRef:
        name: argocd-task-create-sync-wait
      runAfter:
        - commit-and-push-to-config-repo
      params:
        - name: application-name
          value: "$(params.PROJECT_NAME)-$(params.SOURCE_BRANCH)"
        - name: config-repository
          value: "$(params.CONFIG_URL)"
        - name: config-path
          value: deployment
        - name: config-revision
          value: "$(params.SOURCE_BRANCH)"
        - name: destination-namespace
          value: default
        - name: argo-appproject
          value: apps2deploy
        - name: wait-timeout-in-seconds
          description: "Time out after this many seconds."
```





#### Why create Argo App in Tekton?

The Hub task https://hub.tekton.dev/tekton/task/argocd-task-sync-and-wait HAZ NO APP CREATE!

So we created our own simple Task here derived from https://hub.tekton.dev/tekton/task/argocd-task-sync-and-wait and https://github.com/tektoncd/catalog/pull/903/files (since the `v0.1` uses the old ArgoCD version `1.x`):



#### Create ConfigMap

https://hub.tekton.dev/tekton/task/argocd-task-sync-and-wait

```shell
# Get ArgoCD server hostname
kubectl get service argocd-server -n argocd --output=jsonpath='{.status.loadBalancer.ingress[0].hostname}'

kubectl create configmap argocd-env-configmap --from-literal="ARGOCD_SERVER=$(kubectl get service argocd-server -n argocd --output=jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
```




#### Omit error Failed to establish connection to xyz.com:443: x509: certificate is valid for localhost, argocd-server, not xyz.com

```
time="2022-02-04T08:02:13Z" level=fatal msg="Failed to establish connection to a5f715808162c48c1af54069ba37db0e-1371850981.eu-central-1.elb.amazonaws.com:443: x509: certificate is valid for localhost, argocd-server, argocd-server.argocd, argocd-server.argocd.svc, argocd-server.argocd.svc.cluster.local, not a5f715808162c48c1af54069ba37db0e-1371850981.eu-central-1.elb.amazonaws.com"
```

Can't we simply access the ArgoCD server from within our cluster - since it's all deployed on the same K8s cluster?!

Since `argocd-server`  is not enough and produces a `Failed to establish connection to argocd-server:443: dial tcp: lookup argocd-server on 10.100.0.10:53: no such host`, we should go with `argocd-server.argocd.svc.cluster.local` (see https://stackoverflow.com/a/44329470/4964553):

```shell
kubectl create configmap argocd-env-configmap --from-literal="ARGOCD_SERVER=argocd-server.argocd.svc.cluster.local"
```

As this also gives us a `Failed to establish connection to argocd-server.argocd.svc.cluster.local:443: x509: certificate signed by unknown authority` we should try the `--insecure` flag, wwhich is described as:

```
--insecure    Skip server certificate and domain verification
```

Now the argocd command should reach the ArgoCD server as expected.



#### Add ArgoCD AppProject with needed role and create, sync, wait permissions

See https://stackoverflow.com/questions/71052421/argocd-app-create-in-ci-pipeline-github-actions-tekton-throws-permissio/71052422#71052422

Tackling the error:

```
error rpc error: code = PermissionDenied desc = permission denied: applications, create, default/jonashackt/microservice-api-spring-boot, sub: tekton, iat: 2022-02-03T16:36:48Z
```

So maybe we have the following issue described in https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/#local-usersaccounts-v15

> When you create local users, each of those users will need additional RBAC rules set up, otherwise they will fall back to the default policy specified by policy.default field of the argocd-rbac-cm ConfigMap.

Here [ArgoCD Projects]() come into play:

> Projects provide a logical grouping of applications -
> [they] restrict what may be deployed (trusted Git source repositories)


ArgoCD projects have the ability to define [Project roles](https://argo-cd.readthedocs.io/en/stable/user-guide/projects/#project-roles):

> Projects include a feature called roles that enable automated access to a project's applications. These can be used to give a CI pipeline a restricted set of permissions. For example, a CI system may only be able to sync a single app (but not change its source or destination).

So let's get our hands dirty and create a ArgoCD project:

```shell
argocd proj create apps2deploy -d https://kubernetes.default.svc,default --src "*"
```

We create it with the `--src "*"` as a wildcard for any git repository ([as described here](https://github.com/argoproj/argo-cd/issues/5382#issue-799715045)).

Now we create a Project role called `create-sync` via:

```shell
argocd proj role create apps2deploy create-sync --description "project role to create and sync apps from a CI/CD pipeline"
```

You can check the new role has been created with `argocd proj role list apps2deploy`.

Now we need to create a token for the new Project role `create-sync`, which can be created via:

```shell
argocd proj role create-token apps2deploy create-sync
```

Directly update the `ARGOCD_AUTH_TOKEN` in the `argocd-env-secret` secret:

```yaml
kubectl create secret generic argocd-env-secret \
  --from-literal=ARGOCD_AUTH_TOKEN=INSERT_TOKEN_HERE \
  --namespace default \
  --save-config --dry-run=client -o yaml | kubectl apply -f -
```

Now we need to give permissions for Tekton to be able to create and sync our application in ArgoCD. Therefore use ([for more details see](https://argo-cd.readthedocs.io/en/stable/user-guide/projects/#project-roles)):

```shell
argocd proj role add-policy apps2deploy create-sync --action get --permission allow --object "*"
argocd proj role add-policy apps2deploy create-sync --action create --permission allow --object "*"
argocd proj role add-policy apps2deploy create-sync --action sync --permission allow --object "*"
argocd proj role add-policy apps2deploy create-sync --action update --permission allow --object "*"
argocd proj role add-policy apps2deploy create-sync --action delete --permission allow --object "*"
```

Have a look on the role policies with `argocd proj role get apps2deploy create-sync`, which should look somehow like this:

```shell
$ argocd proj role get apps2deploy create-sync
Role Name:     create-sync
Description:   project role to create and sync apps from a CI/CD pipeline
Policies:
p, proj:apps2deploy:create-sync, projects, get, apps2deploy, allow
p, proj:apps2deploy:create-sync, applications, get, apps2deploy/*, allow
p, proj:apps2deploy:create-sync, applications, create, apps2deploy/*, allow
p, proj:apps2deploy:create-sync, applications, update, apps2deploy/*, allow
p, proj:apps2deploy:create-sync, applications, delete, apps2deploy/*, allow
p, proj:apps2deploy:create-sync, applications, sync, apps2deploy/*, allow
JWT Tokens:
ID          ISSUED-AT                                EXPIRES-AT
1644166189  2022-02-06T17:49:49+01:00 (2 hours ago)  <none>
```



#### Introduce `PROJECT_NAME` parameter and create ArgoCD app from Tekton finally

Now we finally need to add our application to the `AppProject` we created.

We add it to our [argocd-task-app-create.yml](tasks/argocd-task-app-create.yml) `argocd app create` command as ` --project "$(params.argo-appproject)"` with a new parameter `argo-appproject`.

Finally we need to introduce a new parameter containing only the project name, since the `REPO_PATH_ONLY` parameter containing e.g. `jonashackt/microservice-api-spring-boot` produces an error like `rpc error: code = Unknown desc = invalid resource name \"jonashackt/microservice-api-spring-boot\": [may not contain '/']`.

So let's introduce `PROJECT_NAME` to our [pipeline.yml](pipelines/pipeline.yml), which we can also retrieve easily in our EventListener / Tekton Trigger solution implemented in [gitlab-push-listener.yml](triggers/gitlab-push-listener.yml).

There we can use `$(body.project.name)` inside the TriggerBinding to retrieve the project name from the payload (see [gitlab-push-test-event.json](triggers/gitlab-push-test-event.json)) and use it later in the parameter definition.

Mind the spec params definition of `project_name` also to not run into `'$(tt.params.project_name)' must be declared in spec.params` errors. Now the parameter can finally be used as:

```yaml
                  - name: PROJECT_NAME
                    value: $(tt.params.project_name)
```

In the end our pipeline should be able to create our app and sync/wait for it to be deployed:

![tekton-argocd-successful-deployment](screenshots/tekton-argocd-successful-deployment.png)

