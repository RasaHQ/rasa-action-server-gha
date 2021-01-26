# Rasa Build Action Server GitHub Action

You can find more information about Rasa actions in [the Rasa Open Source docs](https://rasa.com/docs/rasa/core/actions/).
You can find more information about the action server in the [action server docs](https://rasa.com/docs/action-server).

_You don't need to have a Dockerfile for your action server to build a Docker image, the GH action helps you to build a Docker image in the easiest way possible._

## Input arguments

In order to pass the input parameters to the GH action, you have to use the [`with`](https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions#jobsjob_idstepswith) argument in a step that uses the GH action, e.g.

```yaml
jobs:
  my_first_job:
    steps:
      - name: My first step
        uses: RasaHQ/rasa-action-server-gha@main
        with:
          actions_directory: my_directory
          requirements_file: my_file
          docker_registry: my_registry
```

Here are all the parameters you can change via the inputs available through [`with`](https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions#jobsjob_idstepswith):

|           Input            |                                                           Description                                                           |        Default         |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------- | ---------------------- |
| `actions_directory`        | Path to the directory with actions                                                                                              | `actions`              |
| `requirements_file`        | Path to the requirements.txt file                                                                                               | `none`                 |
| `docker_registry`          | Name of the Docker registry that the Docker image is published to                                                               | `docker.io`            |
| `docker_registry_login`    | Login name for the Docker registry                                                                                              | `none`                 |
| `docker_registry_password` | Password for the Docker registry                                                                                                | `none`                 |
| `docker_image_name`        | Docker image name                                                                                                               | `action_server`        |
| `docker_image_tag`         | Docker image tag                                                                                                                | `${{ github.run_id }}` |
| `docker_registry_push`     | Push a Docker image to the registry. If `false` the user can add manual extra steps in their workflow which use the built image | `true`                 |
| `dockerfile`               | Path to a custom Dockerfile                                                                                                     | `none`                 |
| `rasa_sdk_version`         | Version of the Rasa SDK which should be used to build the image                                                                 | `latest`               |
| `docker_build_args`        | List of build-time variables                                                                                                    | `none`                 |

## Outputs

The list of available output variables:

|          Output          |                                                          Description                                                          |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------- |
| `docker_image_name`      | Docker image name, the name contains the registry address and the image name, e.g., `docker.io/my_account/my_image_name`      |
| `docker_image_tag`       | Tag of the image, e.g., `v1.0`                                                                                                |
| `docker_image_full_name` | Docker image name (contains an address to the registry, image name, and tag), e.g., `docker.io/my_account/my_image_name:v1.0` |

_GitHub Actions that run later in a workflow can use the output parameters returned by the Rasa GitHub Action, see [the example](examples/upgrade-deploy-rasa-x.yml) of output parameters usage._

## Example Usage

Build a Docker image and push it into the Docker Hub registry.

```yaml
jobs:
    build:
        # ...
        steps:
            # ...
            - name: Build an action server
              uses: RasaHQ/rasa-action-server-gha@main
              with:
                docker_image_name: 'rasahq/action-server-example'
                # More details on how to use GitHub secrets:
                # https://docs.github.com/en/actions/configuring-and-managing-workflows/creating-and-storing-encrypted-secrets
                docker_registry_login: ${{ secrets.DOCKER_HUB_LOGIN }}
                docker_registry_password: ${{ secrets.DOCKER_HUB_PASSWORD }}
                # More details about github context:
                # https://docs.github.com/en/actions/reference/context-and-expression-syntax-for-github-actions#github-context
                docker_image_tag: ${{ github.sha }}
            # ...
```

The next examples shows how to build a Docker image with additional requirements.
You have to use the `requirements_file` input argument in order to pass Python packages that have to be install.

```yaml
jobs:
    build:
        # ...
        steps:
            # ...
            - name: Build an action server
              uses: RasaHQ/rasa-action-server-gha@main
              with:
                docker_image_name: 'rasahq/action-server-example'
                # More details on how to use GitHub secrets:
                # https://docs.github.com/en/actions/configuring-and-managing-workflows/creating-and-storing-encrypted-secrets
                docker_registry_login: ${{ secrets.DOCKER_HUB_LOGIN }}
                docker_registry_password: ${{ secrets.DOCKER_HUB_PASSWORD }}
                # More details about github context:
                # https://docs.github.com/en/actions/reference/context-and-expression-syntax-for-github-actions#github-context
                docker_image_tag: ${{ github.sha }}
                requirements_file: 'examples/requirements.txt'
            # ...
```

### Automatically update a Rasa X deployment if a new Docker image is available

This example shows how to use the GitHub action and output variable to upgrade a Rasa X deployment.
In the example was used [the Rasa X helm chart](https://github.com/RasaHQ/rasa-x-helm). More information on how to use the helm chart can be found in [the docs.](https://rasa.com/docs/rasa-x/installation-and-setup/install/helm-chart/#helm-chart-installation)

```yaml
on:
  # Deploy a new image whenever new changes were pushed to the main branch
  push:
    branches:
      - main

jobs:
  build_and_deploy:
    runs-on: ubuntu-latest
    name: Build Rasa Action Server image and upgrade Rasa X deployment
    steps:
    - name: Checkout repository
      uses: actions/checkout@v2

    - id: action_server
      name: Build an action server with custom actions
      uses: RasaHQ/rasa-action-server-gha@main
      with:
        docker_image_name: 'rasahq/action-server-example'
        docker_registry_login: ${{ secrets.DOCKER_HUB_LOGIN }}
        docker_registry_password: ${{ secrets.DOCKER_HUB_PASSWORD }}
        docker_image_tag: ${{ github.sha }}

    - name: Upgrade Rasa X deployment
      run: |
        # More information: https://rasa.com/docs/rasa-x/installation-and-setup/install/helm-chart/

        # Upgrade the helm release using output parameters from the `action_server` step
        helm upgrade --install --reuse-values \
          --set app.name=${{ steps.action_server.outputs.docker_image_name }} \
          --set app.tag=${{ steps.action_server.outputs.docker_image_tag }} rasa rasa-x/rasa-x
```

## Configuration

This section describes how to customize a Docker image build by using the GitHub action.

### Dockerfile

It's possible to use a custom Dockerfile for an image build. An example below shows how to do this.

_Notice: the Rasa SDK uses Python 3.7_

```yaml
jobs:
    build:
        # ...
        steps:
            # ...
            - name: Build an action server
              uses: RasaHQ/rasa-action-server-gha
              with:
                actions_directory: 'examples/actions'
                # Push a Docker image into GitHub Container Registry
                docker_registry: 'docker.pkg.github.com'
                docker_image_name: 'github-account/repository-name/image-name'
                docker_registry_login: ${{ github.actor }}
                docker_registry_password: ${{ secrets.GITHUB_TOKEN }}
                docker_image_tag: ${{ github.sha }}
                # A value of the Dockerfile parameter is a path to a Dockerfile
                # in your repository
                dockerfile: 'examples/Dockerfile'
                requirements_file: 'examples/requirements.txt'
            # ...
```

#### Dockerfile arguments

Below you can find a list of build arguments that you can use for your Dockerfile.

- `GITHUB_SHA` - The commit SHA that triggered the workflow. For example, `ffac537e6cbbf934b08745a378932722df287a53`.
- `GITHUB_REF` - The branch or tag ref that triggered the workflow. For example, `refs/heads/feature-branch-1`.
- `DOCKER_IMAGE_NAME` - A Docker image name, the name contains a registry address and the image name. Fox example, `docker.io/myaccount/myimage`.
- `DOCKER_IMAGE_TAG` - A tag of the image. For example, `v1.0.0`.
- `RASA_SDK_VERSION` - Version of the Rasa SDK, For example, `latest`. The full list of available versions can be found [here](https://github.com/RasaHQ/rasa-sdk/releases).

In addition to default build arguments, it's possible to add custom arguments by using the `docker_build_args` input argument. For example, `docker_build_args: "--build-arg MY_ARG_1=1 --build-arg MY_ARG_2=2"`.

Notice! The requirements file and an action directory are mapped to the following paths.

- `requirements.txt`:`./tmp/requirements.txt`
- `actions_directory`:`./tmp/actions`

An example of the Dockerfile that is used as a default you can find [here](examples/Dockerfile).

## Next steps

- [Install Rasa X](https://rasa.com/docs/rasa-x/installation-and-setup/installation-guide/#)
- [Rasa Actions](https://rasa.com/docs/rasa/core/actions/)
