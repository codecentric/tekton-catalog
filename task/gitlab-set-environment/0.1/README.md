# GitLab Set Environment

This task creates or updates an environment within a gitlab project.

### Install the Task

```
kubectl apply -f https://raw.githubusercontent.com/codecentric/tekton-catalog/task/gitlab-set-environment/0.1/gitlab-set-environment.yaml
```

### Parameters
* **GITLAB_TOKEN_SECRET_NAME**: The name of the kubernetes secret that contains the GitLab access token. _default:_ `gitlab-api-secret`
* **GITLAB_TOKEN_SECRET_KEY**: The key within the kubernetes secret that contains the GitLab token, _default:_ `token`
* **ENVIRONMENT_NAME**: The symbolic name of the environment. _e.g:_ `main`
* **ENVIRONMENT_URL**: The target URL to associate with this environment. 
* **GITLAB_HOST_URL**: The GitLab host domain _default:_ `https://gitlab.com`
* **REPO_FULL_NAME**: The GitLab repository full name, _default:_ `codecentric/tekton-catalog`

## Platforms

The Task can be run on `linux/amd64` platform.
