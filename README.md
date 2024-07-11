# EC2 Mono

This GitHub action will deploy your application to an AWS EC2 instance and terminate old instances running the
application.

The action assumes you will only ever be running a single instance of the application. This means that only one instance
will be created and all other instances running the application will be shut down.

Before associating the elastic IP and shutting down any old instances, the action will wait for a successful HTTPS
response from the newly created server. This ensures zero downtime for your application.

## Usage

This workflow depends heavily on the use of tags to uniquely identify your application in AWS: `app` and `env`. Before
running this workflow, make sure you have an Elastic IP Address tagged with your unique `app` and `env` values.

This workflow assumes you are running an HTTP server. If your application does not run an HTTP server, then the workflow
will not work for you.

## Options

| name                     | required | default                 | description                                                                                                         |
|--------------------------|----------|-------------------------|---------------------------------------------------------------------------------------------------------------------|
| `aws-access-key-id`      | yes      |                         | Your AWS Access Key ID                                                                                              |
| `aws-secret-access-key`  | yes      |                         | Your AWS Secret Access Key                                                                                          |
| `aws-region`             | no       | `us-east-1`             | The AWS region being deploy to                                                                                      |
| `aws-image-id`           | no       | `ami-04a81a99f5ec58529` | The image to build the instance from (Default is Ubuntu 24.04)                                                      |
| `aws-instance-type`      | no       | `t2.micro`              | The type of instance you'd like to use                                                                              |
| `aws-security-group-ids` | yes      |                         | Any security groups you'd like to include on the instance. One of these will need to have SSH access.               |
| `aws-key-name`           | yes      |                         | The name of the SSH key created in AWS                                                                              |
| `ssh-private-key`        | yes      |                         | The private key associated with `aws-key-name`                                                                      |
| `ssh-user`               | no       | `ubuntu`                | The user that will be installing the application                                                                    |
| `app-name`               | yes      |                         | A unique name for your application. Pair with `env-name` to make unique among all of your deployed services.        |
| `env-name`               | yes      |                         | A unique environment for your application. Pair with `app-name` to make unique among all of your deployed services. |
| `install`                | no       |                         | The commands required to install and run your application                                                           |

### Sample Workflow YML

```yml
name: Build & Deploy
on:
  push:
    branches:
      - master
      - stage
      - production

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Run Tests
        run: ./your-test-command

  deploy:
    runs-on: ubuntu-latest
    # Don't deploy unless tests pass
    needs: test

    # Only deploy on stage and production branches
    if: contains('stage production', github.ref_name)

    # Environments in this example are the same as the branch name
    environment: ${{ github.ref_name }}

    # Cancels this deployment if a new one begins
    concurrency:
      group: deploy-${{ github.ref }}
      cancel-in-progress: true

    steps:
      - uses: brandoncorrea/ec2-mono@master
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
          aws-image-id: ami-04a81a99f5ec58529
          aws-instance-type: t2.small
          aws-security-group-ids: 'sg-1234 sg-5678'
          aws-key-name: YourKeyName
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}
          ssh-user: ubuntu
          app-name: acme
          env-name: ${{ github.ref_name }}
          install: |
            # install and start your service
```
