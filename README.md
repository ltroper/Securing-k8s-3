# Securing LKE Cluster Part 3: Monitoring Syscalls with Tracee

## Introduction
System calls (syscalls) provide a vital interaction point between user-space programs and the kernel, and monitoring them can reveal suspicious or malicious activity in your Kubernetes (K8s) cluster. **Tracee**, an open-source runtime security tool powered by eBPF, helps you observe and trace these syscalls in real-time.

This guide will show you how to:
- Deploy Tracee as a **DaemonSet** on a Linode Kubernetes Engine (LKE) cluster.
- Capture syscall activity using Tracee.
- Understand how monitoring syscalls can enhance your cluster’s security by detecting activities like privilege escalation, anti-debugging, and unauthorized file access.

By the end of this walkthrough, you'll be able to monitor your cluster’s syscalls and detect potential security issues in real-time.

---

## Step 1: Deploy an LKE Cluster and Connect to It
First, ensure you have a functioning LKE cluster.

1. **Deploy an LKE cluster** using the Linode Cloud Manager or the `linode-cli`.
2. **Download the kubeconfig file** to connect to your cluster. Set up the environment variable:
    ```bash
    export KUBECONFIG=~/path-to-your-kubeconfig-file
    ```
3. **Verify your connection**:
    ```bash
    kubectl get nodes
    ```

---

## Step 2: Deploy Tracee as a DaemonSet
We'll deploy **Tracee** as a DaemonSet so that it runs on every node in your cluster. Tracee will monitor syscalls from all pods on the respective nodes.

1. Create a YAML file called `tracee-daemonset.yaml`:

    ```yaml
    apiVersion: apps/v1
    kind: DaemonSet
    metadata:
      name: tracee
    spec:
      selector:
        matchLabels:
          app: tracee
      template:
        metadata:
          labels:
            app: tracee
        spec:
          containers:
          - name: tracee
            image: docker.io/aquasec/tracee:v0.7.0
            args:
              - --webhook http://postee-svc:8082 --webhook-template ./templates/rawjson.tmpl --webhook-content-type application/json
            securityContext:
              privileged: true
            volumeMounts:
            - name: tmp-tracee
              mountPath: /tmp/tracee
            - name: etc-os-release
              mountPath: /etc/os-release-host
              readOnly: true
          tolerations:
            - effect: NoSchedule
              operator: Exists
          volumes:
          - name: tmp-tracee
            hostPath:
              path: /tmp/tracee
          - name: etc-os-release
            hostPath:
              path: /etc/os-release
    ```

2. Deploy the DaemonSet:
    ```bash
    kubectl apply -f tracee-daemonset.yaml
    ```

This configuration sets up Tracee with webhook alerts sent to `postee-svc:8082` when it detects suspicious activity.

---

## Step 3: Monitor Syscall Activity in Your Cluster
Once Tracee is running, you can start monitoring syscalls made by your containers. We’ll simulate suspicious activity using the `strace` tool.

1. Deploy an Nginx pod:
    ```bash
    kubectl create deployment nginx --image=nginx
    ```

2. Install `strace` inside the Nginx pod to trace syscalls:
    ```bash
    kubectl exec -ti deployment/nginx -- apt-get update && apt-get install strace
    ```

3. Run `strace` on a simple command (e.g., `ls`):
    ```bash
    kubectl exec -ti deployment/nginx -- strace ls
    ```

4. Check the logs from one of the Tracee pods for activity:
    ```bash
    kubectl logs <tracee-pod-name>
    ```

You'll see a message in the Tracee logs, such as:

```
Time: 2024-09-19T17:48:32Z
Signature ID: TRC-2
Signature: Anti-Debugging
Command: strace
Hostname: nginx-7854ff887
```

This indicates that Tracee detected `strace` usage, which can be considered an anti-debugging action.

---

## Step 4: Understanding Real-World Applications

Monitoring syscalls can be extremely useful for detecting real-time security threats in your Kubernetes clusters. Some real-world applications include:

- **Anti-Debugging**: Detecting attempts to trace or debug running processes (as seen with `strace`). This can prevent attackers from learning about internal system behavior.
- **Privilege Escalation**: Monitoring system calls related to privilege escalation, such as `setuid` or `execve` with high privileges.
- **File Tampering**: Detecting attempts to access or modify sensitive files, such as `/etc/passwd` or `/etc/shadow`.
- **Process Injection**: Tracing syscalls like `ptrace` that are used in process injection attacks, where one process attempts to manipulate another.

By using Tracee, you can enhance your cluster’s runtime security and gain visibility into low-level behaviors that are otherwise difficult to observe.

---

## Step 5: Expanding Your Monitoring with Webhooks and Alerts
In this example, Tracee is configured to send alerts to a webhook at `postee-svc:8082`. You can replace this webhook with your own alerting service, such as:
- **Slack**: Send alerts to a Slack channel.
- **PagerDuty**: Notify your incident response team.
- **SIEMs**: Forward events to security information and event management (SIEM) systems like Splunk or Elasticsearch for centralized monitoring.

---

## Conclusion
By deploying Tracee as a DaemonSet in your LKE cluster, you’ve taken a significant step toward enhancing security observability. Monitoring syscalls can help you detect potential attacks in real time, providing critical insights into the behavior of running containers. In combination with webhook alerts and incident response systems, Tracee offers a powerful solution for runtime security in Kubernetes.

Make sure to continue securing your LKE clusters by implementing network policies, RBAC configurations, and further security tools. This guide serves as Part 3 in your journey toward a fully secure Kubernetes environment.

