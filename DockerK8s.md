# Discovering Docker & Kubernetes: Why I'm Going to Build in Them

**Date:** October 13, 2026
**Type:** Learning reflection / postmortem entry

## Summary

I recently started working with Docker and Kubernetes (K8s) and found them genuinely fun. This entry explains what these tools are, why they clicked for me, where I struggled, and why I now want to build real projects inside them instead of only reading about them.

## Background: what these tools are

**Docker** is a tool for packaging an application together with everything it needs to run: the code, the runtime, the libraries, and the configuration. This package is called an **image**, and a running copy of it is called a **container**. Because the container carries its own environment, the app behaves the same on my laptop, on a teammate's machine, or on a server.

**Kubernetes** is a system for running many containers reliably. Instead of starting and stopping containers by hand, I describe the state I want, for example "run three copies of this app", and Kubernetes works to keep reality matching that description. If a container crashes, it starts a new one. If traffic grows, it can run more copies.

A simple way to think about it: Docker is the shipping container, and Kubernetes is the port that decides where containers go, how many are needed, and what happens when one is damaged.

## What happened

I spent time experimenting with both tools. At first they looked like a wall of unfamiliar terms, but once I ran a few things myself, the ideas started to make sense, and I realised I was enjoying it. It felt less like configuring software and more like solving a puzzle where each piece fits a clear purpose.

## What clicked

- **Docker removes "it works on my machine."** Before, setting up a project meant installing the right versions of languages and tools and hoping nothing conflicted. With Docker, a single `Dockerfile` records every step, so anyone can reproduce the same environment with one command.
- **Containers are lightweight and disposable.** I can start one, break it, delete it, and start fresh in seconds. That makes experimenting safe, which is a big deal when learning.
- **Kubernetes works from declared intent.** I write down what I want in a config file, and Kubernetes keeps making it true. This is very different from manually fixing things when they break.
- **Self-healing made it real.** Watching a container get killed and then automatically restarted was the moment orchestration stopped being an abstract idea. The system noticed the problem and fixed it without me.

## What went wrong (the honest part)

- **Too much vocabulary at once.** Pods, deployments, services, ingress, volumes, namespaces. I tried to understand all of them before doing anything, which made the whole subject feel bigger than it is.
- **Reading more than building.** I spent a lot of time on tutorials and documentation, but the concepts only stuck once I ran commands and saw real output and real errors.
- **Not starting small.** I looked at large, production-style setups too early. A tiny app in a single container would have taught me the fundamentals faster.

**Lesson:** learn one layer at a time. Get comfortable with a single container first, then multiple containers, then orchestration.

## The ambition: building something in Docker

I don't want to only use Docker and Kubernetes as tools other people set up. I want to build a real project that is **containerised from the very first commit**.

The reasons are practical:

- **Reproducibility.** Anyone, including future me on a new machine, can run the project with the same result.
- **Real-world skills.** Containers and orchestration are widely used for deploying software, so building in them teaches how modern applications are actually shipped.
- **Learning through failure.** A project gives me real problems to solve: networking between containers, storing data, handling configuration and secrets. Those are the things tutorials gloss over.
- **A portfolio piece.** A containerised project with clear documentation shows what I can do.

### My plan

1. **Pick a small project.** Something simple, such as a small web app or API with a database, so the focus stays on learning the tooling.
2. **Write a clean Dockerfile.** Understand every line: the base image, how dependencies are installed, which port is exposed, and how the app starts.
3. **Use Docker Compose.** Run the app and its database together with one command, and learn how containers talk to each other.
4. **Move to Kubernetes.** Run it on a local cluster using minikube or kind, writing the deployment and service files myself.
5. **Document everything.** Record what I built, what broke, and how I fixed it, right here in this repository, as a postmortem for each problem.

## Next steps

- [ ] Choose the project idea
- [ ] Containerise it with a Dockerfile
- [ ] Run it locally with Docker Compose
- [ ] Deploy it to a local Kubernetes cluster
- [ ] Write a short postmortem for each failure along the way

## Takeaway

The fastest way for me to learn this stack is to build in it and let things fail safely. Docker and Kubernetes turned out to be more approachable and more enjoyable than I expected, and I'm starting to build now.
