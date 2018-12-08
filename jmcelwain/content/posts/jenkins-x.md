+++
tags = ["devops", "ci/cd", "Kubernetes", "Openshift", "Jenkins X"]
date = "2018-08-10"
title = "Jenkins X and Managed Kubernetes"
+++

## Kubernetes vs Serverless

We've been using Openshift, Red Hat's "distribution" of Kubernetes, for over a year now. Although there are definitely pain points with learning an API with such a large surface area, in general my experience as an application developer has been really positive. Although our biggest gains were effects of moving off an old school Spring/Tomcat WAR deployment model to Docker and Spring Boot, Openshift itself is packaged really nicely and seems to have a lot of commitment from Red Hat. Because Kubernetes releases outside of a LTS model, documentation takes time to catch up, but hopefully this churn will settle as the Kubernetes ecosystem becomes more settled.

Although I've never done a project with a serverless architecture, I've been reading a lot of debate about whether Kubernetes is just a stepping stone to serverless or the future itself. The argument for serverless is pretty simple: abstracting infrastructure entirely is the internal teleology to modern devops and kubernetes has a lot of unmanaged complexity. If Kubernetes is intented to provide a consistent API for scheduling heterogenous kinds of work, serverless achieves the same goal through a nicer interface and promotes an architecture that is scalable at a more fine-grained per-method level.

Although I'm probably bisaed, given I have much more experience with Kubernetes than serverless and the public cloud, the arguments in favor of Kubernetes seem obvious: Kubernetes still does a great job of abstracting infrastructure, and Docker is a strictly more powerful way to package applications than a bundle of code. Serverless is still running a container, you're just provided much more limited access to how that container is built. While stateless services are nice, sometimes having specific knowledge of your storage rather than "it's some kind of network attached volume" is necessary. Being able to schedule work on particular nodes may break the abstraction in some ways, but it seems strictly preferable to have a system that encourages light stateless services but still gives you a way out when things need to get messy.

Furthermore, it's still possible to build single-function style services on Kubernetes, but in a language like Java that specifically benefits from the JIT "getting hot" over time, a one-off container/function based architecture might not always make sense. I'd really love to do a project with a serverless architecture, because I'm sure there are projects where it works really well. At the same time, I really buy the argument that Kubernetes is low level, and the tools that are built on top of Kubernetes will be strictly more powerful than using a vendor specific managed solution. More specifically, managed Kubernetes provides the opportunity to build your infrastructure in a non-vendor specific portable manner by programming to Kubernetes API, while having the actual administration of Kubernetes abstracted and handled by your cloud provider.

Openshift isn't totally there yet, but it's easy to see the value add it has over "vanilla" Kubernetes. Another product I started playing with this past-weekend was "Jenkins X", a cli interface that is meant to handle much of the difficulty of setting up Kubernetes in the public cloud, including provisioning Jenkins on that cluster and providing an opinionated project structure. [This sales pitch provides a great overview of Kubernetes in the modern cloud and what Jenkins X is intended to provide.](https://www.youtube.com/watch?v=BF3MhFjvBTU)

## Jenkins X

Jenkins X is provided as a binary `jx` that wasn't available yet in the AUR for Arch, a sign that it's a *very* new tool. Once installed, I had to go through the process of figuring out how to set up a GKE account. As of writing, AWS's managed Kubernetes solution isn't fully out, and it makes sense that Google has excellent in-house support for managing Kubernetes as a service. 

What follows is an automated set-up that is heavy on magic. To create your first `jx` managed project, all that is required is the following command:

```
jx create cluster gke
```

With a few prompts to select the size and number of nodes in your cluster, `jx` will provision a Kubernetes cluster, install Jenkins, and set up a deployment project. While this process is ideally smooth, in practice, I had a little bit of trouble getting it to run to completion. Twice, the provisioning process timed out, which left the cluster in an undefined state, and required me to go into the GKE control panel and delete nodes to clean up.

When it finally worked, I experimented with the process of creating a new Spring Boot app:
```
jx create spring
```
`jx` provides project archetypes for a few different languages, and one of the benefits of using Spring Boot with Maven is that it provides an incredibly consistent project structure that can be automated by tools like this. For other languages that require more custom configuration, the process is probably less good. Although the entire goal of a tool like `jx` is to provide an opinionated CI/CD system, for someone who does have a mental model of building Spring Boot apps with Maven and Docker, the magic is definitely opaque. `jx` uses Helm, which I am less familiar with, and it's unclear how annoying it would be to have to extend the build system outside of its opinated core.

I've only used Jenkins itself very briefly, and my first impression was,... mixed. It definitely seems powerful, but it also seems like there's a lot of cruft, and it's unclear to me why Blue Ocean feels like a seperate product. Because `jx` provisions everything for you, when something goes wrong, it's kind of hard to diagnose. Although I'm sure if I had more experience with specifically Helm and Jenkins, my mental model of the deployment pipeline would be more clear, it still seems like a very leaky abstraction.

What was most frustrating to me was figuring out how to run a proxy to GKE's managed SQL as a sidecar. Running multiple containers in a pod is kind of unusual, but if I was working with raw Kubernetes API objects, it would be pretty intuitive. The problem with opinionated frameworks, however, is that when they lack documentation, it can be difficult to figure out how to accomplish something in the presence of magic. While I'm sure there's a way to accomplish the sidecar pattern in `jx`, it wasn't obvious to me.

## Openshift vs Jenkins X

Openshift and Jenkins X seems to provide two different kinds of paradigms for what building on top of Kubernetes looks like. Openshift focuses mainly on building extensions to Kubernetes, e.g. projects and security, and providing a nicer interface to Kubernetes. However, Openshift does not fundamentally attempt to hide Kubernetes from the user, it just improves working with the APIs. I can imagine that there's a lot of usability gains that could be had here, including different ways to edit API objects that doesn't involve editing large blocks of YAML.

Jenkins X on the otherhand is opinated and wants to provide large doses of magic in its tooling. Jenkins X also provides a single unified way to do CI/CD. Openshift has a CI/CD product "S2I" that provides similar magical capacities to abstract the process of building Docker containers by simply building from source code, but it's by no means reqiured. Because Openshift is focused on building extensions directly on top of Kubernetes, it provides its own set of API objects that feel very familiar and can be managed in a single unified configuration language. By using Helm and Jenkin's own configuration, Jenkins X ends up proliferating configuration in a way that is a little annoying.
