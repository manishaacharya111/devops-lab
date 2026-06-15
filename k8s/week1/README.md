# DevOps Lab — Production-grade infrastructure project
## Debugging notes — OOMKill and CrashLoopBackOff

While testing resource limits on the `hello-app` deployment (nginxdemos/hello),
I created a second test deployment with `requests` and `limits` both set to `2Mi`
memory to observe behavior under memory pressure.

### What happened

The pod never reached `Running`. `kubectl describe pod` showed:

\`\`\`
Warning  Failed   spec.containers{hello-app}: Error: failed to create containerd task:
failed to create shim task: OCI runtime create failed: runc create failed:
unable to start container process: container init was OOM-killed (memory limit too low?)

Warning  BackOff  Back-off restarting failed container hello-app in pod ...
\`\`\`

### What this means

- At `2Mi`, nginx couldn't even complete its **initialization** — the master
  process plus worker processes need more memory than that just to start up.
  The container was killed (OOM, exit code 137) before it could begin serving requests.
- Kubernetes then entered **CrashLoopBackOff**: it kept retrying with exponential
  backoff (10s → 20s → 40s → 80s...), visible as a climbing `RESTARTS` count in
  `kubectl get pods`.

### Comparison

| Memory limit | Result |
|---|---|
| `2Mi`  | OOMKilled during init — pod never starts, CrashLoopBackOff |
| `10Mi` | Pod starts and runs normally |

### Takeaway

Resource limits that are too aggressive don't just "throttle" an app — they can
prevent it from starting at all, and the resulting failure mode (CrashLoopBackOff)
looks identical to an application bug at first glance. Always test limits against
the app's actual minimum memory footprint before setting them in production manifests.