---
title: "Dockerfile - Entrypoint vs CMD"
date: 2024-06-14T14:15:20+05:30
description: "Let's understand what's the diffence between Entrypoint and CMD in Dockerfile."
tags: ["dockerfile", "docker image"]
cover:
  image: images/dockerfile-entrypoint-vs-cmd/cover.png
---

Recently, I was working on <a href="https://opentelemetry.io/" target="_blank">opentelemetry</a> and needed to customize its Docker image to meet our application requirements. Specifically, I needed to support a writable directory to mount rotating certificates for mTLS connections. The official OpenTelemetry Docker image uses `FROM scratch`, which results in a non-writable directory. It had been a while since I last worked with Dockerfiles, and I recently faced a stupid issue that costed me a couple of hours. The root cause was that I was using `CMD` instead of `ENTRYPOINT`.

I found it valuable to share this via my blog to clarify the distinction between `CMD` and `ENTRYPOINT` in a `Dockerfile`, and to explain when and how each should be used to prevent any confusion.

## Difference between Entrypoint and CMD

In Dockerfile, `CMD` and `ENTRYPOINT` are both instructions that define what command should be run inside the container when it starts. Here's a clear distinction between the two:

### **CMD**

- Specifies the default command to be executed when the container starts.
- If `CMD` is specified multiple times in a Dockerfile, only the last one takes effect.
- It can be overridden using `args` in the K8s Kustomization spec or by passing arguments to `docker run` after the image name.

Example:

```Dockerfile
CMD ["otelcol-contrib", "--config=config.yaml"]
```

### **ENTRYPOINT**

- Defines the executable that will run when the container starts.
- Unlike `CMD`, <mark>**ENTRYPOINT** parameters are not overridden</mark> when K8s Kustomization spec or `docker run` specifies a command args.
- If you need to pass arguments to the `ENTRYPOINT`, you can do so using `CMD`.

Example:

```Dockerfile
ENTRYPOINT ["otelcol-contrib"]
CMD ["--config=config.yaml"]
```

## When to use `ENTRYPOINT` vs `CMD`?

When deciding between `ENTRYPOINT` and `CMD` in your Dockerfile, it’s crucial to follow these guidelines:

- **ENTRYPOINT** should specify the path to the main executable binary or process that runs inside the container. It defines the primary command that K8/Docker executes when the container starts.

- **CMD** should provide default arguments for the `ENTRYPOINT` command. These arguments are optional and can be overridden by users as needed when running the container.

## My Dockerfile Tale: From Error to Clarity

<details>
  <summary><mark>Click here</mark> to uncover the mistake, the debugging journey, and how I fixed the issue!</summary>

**What I Did Wrong Initially**

I started by writing the `Dockerfile` and used `CMD` to define the command that runs when the container starts. My intention was to specify the binary and pass the default path for the config file, like this:

```Dockerfile
From alpine:3.18
...
...

CMD ["otelcol-contrib"]
CMD ["--config=config.yaml"]
```

Next, I created a simple Kustomization to deploy this image, thinking it would only override the CMD config path. However, when I deployed the pod, it failed!

```yaml {linenos=false,hl_lines=["16-17"]}
# Kustomization deployment file
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otelcollector
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app.kubernetes.io/name: otelcollector
    spec:
      containers:
        - name: otelcol
          image: docker.io/otel-metrics-collector
          args:
            - --config=/etc/otel/config.yaml
          volumeMounts:
            - name: collector-config
              mountPath: /etc/otel
      volumes:
        - name: collector-config
          configMap:
            items:
              - key: config.yaml
                path: config.yaml
```

The pod failed to start with the following error:

```bash {linenos=false,hl_lines=[5]}
Error response from daemon: failed to create task for container:
failed to create shim task: OCI runtime create failed:
runc create failed: unable to start container process:
exec: "--config=/etc/otel/config.yaml": stat --config=/etc/otel/config.yaml:
no such file or directory: unknown.
```

**My Debugging Journey:**

1. Initially, I suspected the issue was with mounting the config file via configMap due to the `no such file or directory error`, so I spent time troubleshooting there without success.
2. I then modified the Dockerfile to replace the existing `CMD` with `CMD ["sh", "-c", "sleep infinity"]` to keep the container running for debugging.
3. To my surprise, the config file was in the correct location, and the binary worked as expected with the config file.
4. Confused, I revisited the `args: --config=/etc/otel/config.yaml` in the Kustomization file. While Googling, I discovered that args might be overridden, which reminded me that I had mistakenly used `CMD` instead of `ENTRYPOINT`.
5. Further research clarified that `ENTRYPOINT` should define the executable, which cannot be overridden, while `CMD` should only provide the default arguments that can be overridden.

**How the Issue Was Fixed!**

Using `ENTRYPOINT` as shown below resolved the issue and taught me a valuable lesson worth sharing with my colleagues and curious minds on the internet.

```Dockerfile
From alpine:3.18
...
...

ENTRYPOINT ["otelcol-contrib"]
CMD ["--config=config.yaml"]
```

> *Thanks for reading! I hope my mistake provided some useful insights for you.*
</details>
