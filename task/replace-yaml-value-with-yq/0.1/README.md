# Replace Yaml values with yq

This task replaces multiple (or one) value(s) using yq - and works (in contrast to the Tekton Hubs yq task, see https://stackoverflow.com/questions/70944069/tekton-yq-task-gives-safelyrenamefile-erro-failed-copying-from-tmp-temp-e/70944070#70944070)

### Install the Task

```
kubectl apply -f https://raw.githubusercontent.com/codecentric/tekton-catalog/main/task/replace-yaml-value-with-yq/0.1/replace-yaml-value-with-yq.yaml
```

### Parameters
* **YQ_EXPRESSIONS** (type: array): The yq expression yaml selectors to choose a key(s) for replacement like .spec.template.spec.containers[0].image = \"$(params.IMAGE):$(params.SOURCE_REVISION)\""
* **FILE_PATH**: The file path relative to the workspace dir.
* **YQ_VERSION** (optional): Version of https://github.com/mikefarah/yq

## Platforms

The Task can be run on `linux/amd64` platform.


## Examples

### Add branch name in Deployment & Service (metadata.name, spec.selector.matchLabels.branch, spec.template.metadata.labels.branch, spec.selector.branch)

Now that we have our `branch name` extracted via Tekton Triggers CEL interceptor and available for the Pipeline as a parameter, we should add it to our application's `Deployment` and `Service` manifests.

```yaml
   - name: replace-deployment-name-branch-image
      taskRef:
        name: replace-yaml-values-with-yq
      runAfter:
        - fetch-config-repository
      workspaces:
        - name: source
          workspace: config-workspace
      params:
        - name: YQ_EXPRESSIONS
          value:
            - ".metadata.name = \"$(params.PROJECT_NAME)-$(params.SOURCE_BRANCH)\""
            - ".spec.template.spec.containers[0].image = \"$(params.IMAGE):$(params.SOURCE_REVISION)\""
            - ".spec.selector.matchLabels.branch = \"$(params.SOURCE_BRANCH)\""
            - ".spec.template.metadata.labels.branch = \"$(params.SOURCE_BRANCH)\""
        - name: FILE_PATH
          value: "./deployment/deployment.yml"
```

In our application configuration repository https://gitlab.com/jonashackt/microservice-api-spring-boot-config inside the `deployment/deployment.yml` we need to replace:

* `metadata.name` to contain our `PROJECT_NAME-SOURCE_BRANCH` - for example  `microservice-api-spring-boot-trigger-tekton-via-webhook`
* `spec.template.spec.containers[0].image` must contain the correct image name as already implemented
* `spec.selector.matchLabels.branch` should contain the `branch name`
* `spec.template.metadata.labels.branch` should also contain the `branch name`

And in the application configuration repository's `deployment/service.yml` we need to replace:

* `metadata.name` to contain `PROJECT_NAME-SOURCE_BRANCH` - just like in our Deployment
* `spec.selector.branch` should contain the `branch name` - also very similar to our Deployment

which inside our [pipeline.yml](tekton/pipelines/pipeline.yml) looks like:

```yaml
    - name: replace-service-name-branch
      taskRef:
        name: replace-yaml-values-with-yq
      runAfter:
        - replace-deployment-name-branch-image
      workspaces:
        - name: source
          workspace: config-workspace
      params:
        - name: YQ_EXPRESSIONS
          value:
            - ".metadata.name = \"$(params.PROJECT_NAME)-$(params.SOURCE_BRANCH)\""
            - ".spec.selector.branch = \"$(params.SOURCE_BRANCH)\""
        - name: FILE_PATH
          value: "./deployment/service.yml"
```


The `traefik-ingress-route.yml` will also be added to our application configuration repository https://gitlab.com/jonashackt/microservice-api-spring-boot-config in the `deployment` directory.

```yaml
    - name: replace-ingress-name-route
      taskRef:
        name: replace-yaml-values-with-yq
      runAfter:
        - replace-service-name-branch
      workspaces:
        - name: source
          workspace: config-workspace
      params:
        - name: YQ_EXPRESSIONS
          value:
            - ".metadata.name = \"$(params.PROJECT_NAME)-$(params.SOURCE_BRANCH)-ingressroute\""
            - ".spec.routes[0].match = \"Host(`$(params.PROJECT_NAME)-$(params.SOURCE_BRANCH).$(params.TRAEFIK_DOMAIN)`)\""
            - ".spec.routes[0].services[0].name = \"$(params.PROJECT_NAME)-$(params.SOURCE_BRANCH)\""
        - name: FILE_PATH
          value: "./deployment/traefik-ingress-route.yml"
```
