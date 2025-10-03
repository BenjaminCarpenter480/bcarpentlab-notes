---
title: Building Container Images in Kubernetes with GitLab CI/CD
date: 2025-08-11 08:00:00 +0000
#categories: [TOP_CATEGORY, SUB_CATEGORY]
tags: [kubernetes, ci/cd, containers, registry]     # TAG names should always be lowercase
---

Container images have become the standard way to package and distribute applications in cloud environments. When setting up a CI/CD pipeline for containerized applications, one critical decision is choosing the right tool for building these images. This article explores different container image building tools, compares their features, and provides a step-by-step guide to set up a secure image building environment using Kubernetes and GitLab CI/CD.

Whether you're a DevOps engineer, a platform developer, or just curious about container technologies, this guide will help you understand the tradeoffs between different image building approaches and demonstrate how to implement a secure and efficient solution.

## BuildKit vs BuildAh vs Docker Build - A Comparative Analysis

After some research and testin, I found BuildKit to be the preferred choice for most CI/CD environments. It offers better feature support (caching, parallelism), can be run securely in rootless mode, and has a simpler setup that leads to better adoption.

### Docker Build

I recommend against using standard Docker build for CI/CD pipelines due to its requirement for privileged access, which creates security concerns for public-facing services. As noted in the [GitLab documentation](https://docs.gitlab.com/ci/docker/using_docker_build/#use-docker-in-docker), the Docker-in-Docker approach requires privileged mode, which exposes your system to potential security risks.

### Security Considerations - Rootless, Unprivileged Mode

While BuildAh was designed with security in mind (no root access required), BuildKit now provides an equivalent solution through its rootless build mode without any privileged requirements ([StackOverflow discussion](https://stackoverflow.com/a/68395788/7052741)). This makes BuildKit a secure option that doesn't compromise on features.

### Performance Features - Caching and Parallelism

BuildKit offers several performance advantages that can significantly reduce build times:

- Dedicated run cache allows faster performance as it eliminates the need to download dependencies at each step ([Docker docs](https://docs.docker.com/build/cache/#use-the-dedicated-run-cache))
- Ability to output to multiple registries in a single command ([LabLabs blog](https://lablabs.io/blog/building-container-images-in-cloud-native-ci-pipelines))
- Early support for S3 caching for distributed builds
- Cache exporters/importers for reuse across CI jobs
- Automatic parallelization of build steps: "BuildKit can figure that out and run the build steps in parallel until that dependency becomes a blocker" ([Python Speed](https://pythonspeed.com/articles/docker-buildkit/))

Multiple benchmark tests have shown reduced build times with BuildKit compared to Buildah.

### Developer Experience - Simplicity and Familiarity

BuildKit offers a more user-friendly experience compared to Buildah, which uses a more flexible but involved scripting configuration. This is important when considering adoption across teams with varying technical expertise.

Since BuildKit is closely related to Docker, many developers will already be familiar with its concepts and syntax. It also offers simple configuration options, like changing Docker versions with a single frontend argument ([Python Speed](https://pythonspeed.com/articles/docker-buildkit/)).

The image for `buildkit:rootless` is a lightweight AlpineLinux variation (see the [Dockerfile](https://github.com/moby/buildkit/blob/7bf23607055200aeb2f582c1a235af34524c8fe3/Dockerfile#L462)), making it an excellent choice for CI environments.

## Setting Up a Test Environment with Kubernetes and GitLab Runner

Now that we understand why BuildKit is a good choice, let's set up a local test environment to experiment with container image building in Kubernetes using GitLab CI/CD.

### Prerequisites

You'll need the following tools installed:

- [Docker](https://docs.docker.com/get-docker/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm](https://helm.sh/docs/intro/install/)
- [KinD](https://kind.sigs.k8s.io/docs/user/quick-start/) (or Minikube)

### Creating a Local Kubernetes Cluster

We'll use KinD (Kubernetes in Docker) to create a local cluster for testing:

1. Create a new cluster:

```bash
kind create cluster --name gitlab-learn
```

2. Verify it's running:

```bash
kubectl get nodes
```

When you're done testing, you can clean up with:

```bash
kind cluster delete gitlab-learn
```

> [!WARNING]
> If you're using an Apple M-chip laptop or a Raspberry pi, you'll need additional configuration as there's currently no Arm architecture GitLab runner helper code available. I just used a different laptop cause I'm lazy and lucky to have another about!

### Deploying a Secure GitLab Runner with BuildKit

We'll use the official Helm chart to deploy a GitLab Runner with Kubernetes executor and rootless BuildKit:

1. Create a namespace for the runner:

```bash
kubectl create namespace gitlab-buildkit-runner
```

2. Add the GitLab Helm repository:

```bash
helm repo add gitlab https://charts.gitlab.io && helm repo update
```

3. Create a configuration file for the runner (save as `gitlab-buildkit-runner-values.yml`):

```yaml
gitlabUrl: "https://gitlab.example.com"  # Replace with your GitLab instance URL
runnerRegistrationToken: "<RUNNER_TOKEN>"  # Replace with your actual runner token

rbac:
  create: true

runners:
  config: |
    [[runners]]
      name = "k8s-buildkit-runner"
      executor = "kubernetes"
      [runners.kubernetes]
        namespace = "gitlab-buildkit-runner"
        image = "moby/buildkit:rootless"
        privileged = false
```

4. Deploy the runner:

```bash
helm install --namespace gitlab-buildkit-runner gitlab-buildkit-runner -f gitlab-buildkit-runner-values.yml gitlab/gitlab-runner
```

### Example Implementation

I've successfully implemented this setup in a test environment as a proof of concept. A typical CI pipeline using this runner can build and publish container images without any privileged access, maintaining security while offering excellent performance.

The key advantage of this approach is that it allows teams to build container images in a CI/CD pipeline without compromising on security by avoiding privileged containers or the Docker-in-Docker pattern.

## Conclusion

Choosing the right container image building strategy for your CI/CD pipeline is critical for both security and performance. BuildKit in rootless mode provides an excellent balance of features, security, and ease of use, making it a strong choice for most environments.

By following the steps in this guide, you can create a secure, efficient container image building pipeline using Kubernetes and GitLab CI/CD that works well for teams of various skill levels.

### References and Further Reading

- [GitLab Documentation on Docker Build](https://docs.gitlab.com/ci/docker/using_docker_build/#use-docker-in-docker)
- [StackOverflow Discussion on BuildKit Security](https://stackoverflow.com/a/68395788/7052741)
- [Docker Documentation on Build Cache](https://docs.docker.com/build/cache/#use-the-dedicated-run-cache)
- [Python Speed Article on BuildKit](https://pythonspeed.com/articles/docker-buildkit/)
- [LabLabs Blog on Container Images in CI Pipelines](https://lablabs.io/blog/building-container-images-in-cloud-native-ci-pipelines)
- [GitLab Runner Kubernetes Executor Documentation](https://docs.gitlab.com/runner/install/kubernetes/)
