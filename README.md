[![GitHub release](https://img.shields.io/github/release/linki/cloudformation-operator.svg)](https://github.com/linki/cloudformation-operator/releases)
[![Docker Repository on Quay](https://quay.io/repository/linki/cloudformation-operator/status "Docker Repository on Quay")](https://quay.io/repository/linki/cloudformation-operator)

# cloudformation-operator

A Kubernetes operator for managing CloudFormation stacks via `kubectl` and a custom resource definition.

**Warning: this project is in alpha state. It should only be used to try out the demo and get the general idea.**

**This version uses the new operator-sdk. It's untested and may not work correctly**

# Deploy to a cluster

You need API access to a cluster running at least Kubernetes v1.7.

Start the CloudFormation operator in your cluster by using the provided manifests:

```console
$ kubectl create -f deploy/rbac.yaml
$ kubectl create -f deploy/crd.yaml
$ kubectl create -f deploy/operator.yaml
```

Modify the `region` flag to match your cluster's.

Additionally you need to make sure that the operator Pod has enough AWS IAM permissions to create, update and delete CloudFormation stacks as well as permission to modify any resources that are part of the CloudFormation stacks you intend to deploy. In order to follow the example below it needs access to CloudFormation as well as S3.

Use the following Policy document as a guideline in order to follow the tutorial:

```yaml
MyIAMRole:
  Properties:
    ...
    Policies:
    - PolicyDocument:
        Statement:
        - {Action: 'cloudformation:*', Effect: Allow, Resource: '*'}
        - {Action: 's3:*', Effect: Allow, Resource: '*'}
        Version: '2012-10-17'
    ...
```

The operator will usually use the IAM role of the EC2 instance it's running on, so you have to add those permissions to that role. If you're using [Kube2IAM](https://github.com/jtblin/kube2iam) or similar and give your Pod a dedicated IAM role then you have to add the permissions to that role.

Once running the operator should print some output but shouldn't actually do anything at this point. Leave it running, keep watching its logs and continue with the steps below.

# Demo

## Create stack

Currently you don't have any stacks.

```console
$ kubectl get stacks
No resources found.
```

Let's create a simple one that manages an S3 bucket:

```yaml
apiVersion: cloudformation.linki.space/v1alpha1
kind: Stack
metadata:
  name: my-bucket
spec:
  template: |
    ---
    AWSTemplateFormatVersion: '2010-09-09'

    Resources:
      S3Bucket:
        Type: AWS::S3::Bucket
        Properties:
          VersioningConfiguration:
            Status: Suspended
```

The Stack resource's definition looks a lot like any other Kubernetes resource manifest.
The `spec` section describes an attribute called `template` which contains a regular CloudFormation template.

Go ahead and submit the stack definition to your cluster:

```console
$ kubectl apply -f examples/cfs-my-bucket-v1.yaml
stack "my-bucket" created
$ kubectl get stacks
NAME        AGE
my-bucket   21s
```

Open your AWS CloudFormation console and find your new stack.

![Create stack](docs/img/stack-create.png)

Once the CloudFormation stack is created check that your S3 bucket was created as well.

The operator will write back additional information about the CloudFormation Stack to your Kubernetes resource's `status` section, e.g. the `stackID`:

```console
$ kubectl get stacks my-bucket -o yaml
spec:
  template:
  ...
status:
  stackID: arn:aws:cloudformation:eu-central-1:123456789012:stack/my-bucket/327b7d3c-f27b-4b94-8d17-92a1d9da85ab
```

Voilà, you just created a CloudFormation stack by only talking to Kubernetes.

## Update stack

You can also update your stack: Let's change the `VersioningConfiguration` from `Suspended` to `Enabled`:

```yaml
apiVersion: cloudformation.linki.space/v1alpha1
kind: Stack
metadata:
  name: my-bucket
spec:
  template: |
    ---
    AWSTemplateFormatVersion: '2010-09-09'

    Resources:
      S3Bucket:
        Type: AWS::S3::Bucket
        Properties:
          VersioningConfiguration:
            Status: Enabled
```

As with most Kubernetes resources you can update your `Stack` resource by applying a changed manifest to your Kubernetes cluster or by using `kubectl edit stack my-stack`.

```console
$ kubectl apply -f examples/cfs-my-bucket-v2.yaml
stack "my-bucket" configured
```

Wait until the operator discovered and executed the change, then look at your AWS CloudFormation console again and find your stack being updated, yay.

![Update stack](docs/img/stack-update.png)

## Tags

You may want to assign tags to your CloudFormation stacks. The tags added to a CloudFormation stack will be propagated to the managed resources. This feature may be useful in multiple cases, for example, to distinguish resources at billing report. Current operator provides two ways to assign tags:
- `--tag` command line argument or `AWS_TAGS` environment variable which allows setting default tags for all resources managed by the operator. The format is `--tag=foo=bar --tag=wambo=baz` on the command line or with a line break when specifying as an env var. (e.g. in zsh: `AWS_TAGS="foo=bar"$'\n'"wambo=baz"`)
- `tags` parameter at kubernetes resource spec:
```yaml
apiVersion: cloudformation.linki.space/v1alpha1
kind: Stack
metadata:
  name: my-bucket
spec:
  tags:
    foo: dataFromStack
  template: |
    ---
    AWSTemplateFormatVersion: '2010-09-09'

    Resources:
      S3Bucket:
        Type: AWS::S3::Bucket
        Properties:
          VersioningConfiguration:
            Status: Enabled
```

Resource-specific tags have precedence over the default tags. Thus if a tag is defined at command-line arguments and for a `Stack` resource, the value from the `Stack` resource will be used.

If we run the operation and a `Stack` resource with the described above examples, we'll see such picture:

![Stack tags](docs/img/stack-tags.png)

## Parameters

However, often you'll want to extract dynamic values out of your CloudFormation stack template into so called `Parameters` so that your template itself doesn't change that often and, well, is really a *template*.

Let's extract the `VersioningConfiguration` into a parameter:

```yaml
apiVersion: cloudformation.linki.space/v1alpha1
kind: Stack
metadata:
  name: my-bucket
spec:
  parameters:
    VersioningConfiguration: Enabled
  template: |
    ---
    AWSTemplateFormatVersion: '2010-09-09'

    Parameters:
      VersioningConfiguration:
        Type: String

    Resources:
      S3Bucket:
        Type: AWS::S3::Bucket
        Properties:
          VersioningConfiguration:
            Status:
              Ref: VersioningConfiguration
```

and apply it to your cluster:

```console
$ kubectl apply -f examples/cfs-my-bucket-v3.yaml
stack "my-bucket" configured
```

Since we changed the template a little this will update your CloudFormation stack. However, since we didn't actually change anything because we injected the same `VersioningConfiguration` value as before, your S3 bucket shouldn't change.

Any CloudFormation parameters defined in the CloudFormation template can be specified in the `Stack` resource's `spec.parameters` section. It's a simple key/value map.

## Outputs

Furthermore, CloudFormation supports so called `Outputs`. These can be used for dynamic values that are only known after a stack has been created.
In our example, we don't define a particular S3 bucket name but instead let AWS generate one for us.

Let's change our CloudFormation template to expose the generated bucket name via an `Output`:

```yaml
apiVersion: cloudformation.linki.space/v1alpha1
kind: Stack
metadata:
  name: my-bucket
spec:
  parameters:
    VersioningConfiguration: Enabled
  template: |
    ---
    AWSTemplateFormatVersion: '2010-09-09'

    Parameters:
      VersioningConfiguration:
        Type: String

    Resources:
      S3Bucket:
        Type: AWS::S3::Bucket
        Properties:
          VersioningConfiguration:
            Status:
              Ref: VersioningConfiguration

    Outputs:
      BucketName:
        Value: !Ref 'S3Bucket'
```

Apply the change to our cluster and wait until the operator has successfully updated the CloudFormation stack.

```console
$ kubectl apply -f examples/cfs-my-bucket-v4.yaml
stack "my-bucket" configured
```

Every `Output` you define will be available in your Kubernetes resource's `status` section under the `outputs` field as a key/value map.

Let's check the name of our S3 bucket:

```console
$ kubectl get stacks my-bucket -o yaml
spec:
  template:
  ...
status:
  stackID: ...
  outputs:
    BucketName: my-bucket-s3bucket-tarusnslfnsj
```

In the template we defined an `Output` called `BucketName` that should contain the name of our bucket after stack creation. Looking up the corresponding value under `.status.outputs[BucketName]` reveals that our bucket was named `my-bucket-s3bucket-tarusnslfnsj`.

## Delete stack

The operator captures the whole lifecycle of a CloudFormation stack. So if you delete the resource from Kubernetes, the operator will teardown the CloudFormation stack as well. Let's do that now:

```console
$ kubectl delete stack my-bucket
stack "my-bucket" deleted
```

Check your CloudFormation console once more and validate that your stack as well as your S3 bucket were deleted.

![Delete stack](docs/img/stack-delete.png)

# Command-line arguments

Argument | Environment variable | Default value | Description
---------|----------------------|---------------|------------
assume-role | | | Assume AWS role when defined. Useful for stacks in another AWS account. Specify the full ARN, e.g. `arn:aws:iam::123456789:role/cloudformation-operator`
capability | | | Enable specified capabilities for all stacks managed by the operator instance. Current parameter can be used multiple times. For example: `--capability CAPABILITY_NAMED_IAM --capability CAPABILITY_IAM`. Or with a line break when specifying as an environment variable: `AWS_CAPABILITIES=CAPABILITY_IAM$'\n'CAPABILITY_NAMED_IAM`
dry-run | | | If true, don't actually do anything.
tag ... | | | Default tags which should be applied for all stacks. The format is `--tag=foo=bar --tag=wambo=baz` on the command line or with a line break when specifying as an env var. (e.g. in zsh: `AWS_TAGS="foo=bar"$'\n'"wambo=baz"`)
namespace | WATCH_NAMESPACE | default | The Kubernetes namespace to watch
region | | | The AWS region to use

# Cleanup

Clean up the resources:

```console
$ kubectl delete -f examples/
$ kubectl delete -f deploy/rbac.yaml
$ kubectl delete -f deploy/crd.yaml
$ kubectl delete -f deploy/operator.yaml
```

# Build and run locally

This project uses the [operator sdk](https://github.com/operator-framework/operator-sdk).

```console
$ go build -o ./tmp/_output/bin/cloudformation-operator ./cmd/manager
  $ WATCH_NAMESPACE=default KUBERNETES_CONFIG=~/.kube/config ./tmp/_output/bin/cloudformation-operator --region eu-central-1
$ # if you're using the operator-sdk helper use `operator-flags` to configure the flags.
$ operator-sdk up local --operator-flags="--region=eu-central-1"
```

## Build the docker image

```console
$ operator-sdk build quay.io/linki/cloudformation-operator:v0.6.0
$ docker push quay.io/linki/cloudformation-operator:v0.6.0
$ # or use the previously used Dockerfile (not the one from operator-sdk)
$ docker build -t quay.io/linki/cloudformation-operator:v0.6.0 .
```

## Test it locally

You can use `--operator-flags` to pass in flags using the operator-sdk.

Assuming you are using minikube:

```console
$ minikube start # you will be have a kubeconfig read to use by cloudformation operator
$ export AWS_PROFILE=my_profile # setup your aws config
$ cd $GOPATH/src/github.com/linki/cloudformation-operator
$ # run cloudformation operator based on previous settings and env vars
$ WATCH_NAMESPACE=staging operator-sdk up local --operator-flags="--dry-run=true --region=eu-central-1"
INFO[0000] Go Version: go1.13.0
INFO[0000] Go OS/Arch: darwin/amd64
INFO[0000] operator-sdk Version: v0.10.0
INFO[0000] cloudformation-operator Version: 0.6.0+git
INFO[0000] starting stacks controller
```
