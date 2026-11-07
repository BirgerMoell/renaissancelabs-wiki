# Running RL rollouts with internet access on a shared cluster

**Status:** design notes
**Authors:** Renaissance Labs

Reinforcement learning for coding and agent tasks means running a lot of model-written code, and often giving it internet access. On a shared cluster that is a real risk. These notes describe the design we would use. They are general principles and not tied to any one cluster.

## Why a cluster is different from a laptop

A cluster is a login node, a scheduler such as Slurm, compute nodes, a shared filesystem and a fast interconnect. Jobs run as the user who submitted them and can reach whatever that user can reach: their files, their credentials, and other nodes. The operators hold the user responsible for what the job does. Code that a model writes during training therefore inherits the full authority of the person who started it.

## Layers of the design

1. **Sandbox.** Run each rollout in an isolated environment. Plain containers share the host kernel. gVisor intercepts system calls in user space, and Firecracker microVMs run their own kernel. Apptainer mounts your home directory and shares the host network by default, so use options such as `--containall`, `--no-home` and a separate network namespace. Where you cannot get root, rootless options (user namespaces, bubblewrap, seccomp, Landlock) may be available; that depends on the cluster's policy.
2. **No direct network.** Give the sandbox a network namespace with no interfaces, and one socket to a proxy outside it.
3. **Proxy.** All outbound traffic goes through it. It enforces an allowlist of hosts, blocks internal addresses, caps request rate and size, and logs every request.
4. **Monitor.** Log commands, files touched and requests made, with a rollout ID on each line. Alert on anything outside the allowlist, on large transfers and on attempts to read credentials. Review a sample by hand regularly.
5. **Staged rollout.** Start with no internet and a handful of rollouts. Add internet to a small read-only allowlist. Scale up only after reading the logs. Widen access one step at a time, and keep the ability to stop everything.

## Risks to keep in mind

- **The lethal trifecta** (Simon Willison, 2025): private data, untrusted content, and a way to communicate externally. Remove at least one leg.
- **Reward hacking.** Coding agents edit tests, read hidden files and use git history. Learning to reward-hack has been reported to generalise into broader misalignment.
- **Training against a monitor.** If the training signal penalises what the monitor sees in the model's reasoning, the model can learn to hide it (Baker et al., 2025).
- **Escapes.** Kernel bugs and bad mounts have led to container escapes, so patching and least privilege matter.
- **Do not train on security or red-teaming tasks** until the setup has a track record.

## Working rules

Go gradually, reduce the blast radius, assume the sandbox can fail, and don't be a cowboy. Even code-execution RL with no internet is high risk with frontier-class models.

## Caveats

This was written from general knowledge and not from a new literature search. What is allowed on your cluster varies: whether compute nodes can reach the internet, and whether user namespaces are enabled, are decided by its operators. Ask them.
