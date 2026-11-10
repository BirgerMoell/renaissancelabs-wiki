# Running agentic RL environments on a cluster: a practical guide

**Status:** ongoing (first model rollouts done; full-dataset validation in progress)
**Authors:** Birger Moëll, drafted with AI assistance (Claude Code) from the session logs of 10 November 2026

This page is for people who want to run agentic reinforcement-learning environments (tasks where a model works in a shell and a script checks the result) on a shared GPU cluster. It explains what the pieces of software are, shows how we run them, and sets out the safety practice we follow. It records what we actually measured, including what went wrong.

It builds on two other pages: [Validating RL environments](../external/marianna13-rl-validation.md), which describes the datasets and the validation protocol, and [RL rollouts with internet access](rl-rollouts-with-internet.md), which gives the general design principles. This page is the hands-on companion.

## Summary

- We took the twelve [OT validated RL datasets](https://huggingface.co/collections/laion/ot-validated-rl-datasets) and got them running on an x86_64 Slurm cluster with rootless Apptainer.
- The released container images are built for aarch64, so they cannot be used on x86_64. Every image has to be rebuilt from the task's Dockerfile, and the "validated" claim has to be re-checked on the new machine.
- Out of the box, a task container on our cluster could read and write the whole shared project filesystem and reach the internet. We closed both with an opt-in sandbox: the shared filesystem is not mounted, and every command runs under Landlock and seccomp.
- With the sandbox on, the reference solutions passed 12 of 12 test tasks, and a first open model (Nemotron 3.5 Lightning) solved 10 of 12.
- The main lesson: validate with scripted agents first, add isolation before any model writes a command, and expect most of your time to go to infrastructure rather than models.

## How to learn this

If you are new, work through the stages in this order. Each one teaches something the next one needs, and each is cheap to get wrong.

1. **Read one task by hand.** Open a task folder and read `instruction.md`, the `Dockerfile`, `solution/solve.sh` and `tests/test.sh`. You should be able to say what the model is asked to do and how the script decides whether it succeeded.
2. **Run one task with the reference solution.** No model, no GPU. If this fails, the problem is your infrastructure.
3. **Run the same task with an agent that does nothing.** It must score zero. If it passes, the grader is broken and any reward from that task is meaningless.
4. **Scale the two scripted runs to a few tasks per dataset.** This is where image builds, rate limits and missing dependencies show up.
5. **Decide the sandbox, and test it with commands you wrote yourself.** Prove what a command in the container can and cannot reach before a model gets to try.
6. **Run a model on a dozen tasks.** Read the trajectories, not only the score.
7. **Only then think about training.**

The scripted stages (2 to 4) need only CPUs, so they are a good place to learn without spending GPU time.

## What the software is

| Term | What it is | Why you need it |
|---|---|---|
| **RL environment** | Something a model can act in and get a reward from. Here: a container with a shell, an instruction, and a grading script. | It is the thing you train or evaluate on. |
| **Task** | One problem in an environment: a folder with `instruction.md`, `environment/Dockerfile`, `tests/` and usually `solution/`. | The unit you run. A dataset is a few hundred to a few thousand tasks. |
| **Verifier (grader)** | The script in `tests/` that inspects the container afterwards and writes a reward, usually 0 or 1. | Without a trustworthy verifier there is no training signal. |
| **Oracle** | A scripted agent that just runs the task's reference solution. | Shows the task is solvable on your machine. |
| **No-op (negative control)** | A scripted agent that does nothing. | Shows the verifier does not hand out reward for free. |
| **[Harbor](https://github.com/harbor-framework/harbor)** | A framework that takes a folder of tasks, starts an environment for each, lets an agent act, runs the verifier and collects results. | It is the runner. The datasets here are in Harbor's task format. |
| **Terminus-2** | A Harbor agent that connects a language model to a terminal: it shows the model the screen and types the model's keystrokes into a tmux session. | It is how a model "uses" the shell. |
| **tmux** | A terminal multiplexer: a shell session that keeps running in the background and can be read and typed into by another program. | Terminus-2 needs it inside the container. |
| **Docker / Dockerfile** | Docker runs containers; a Dockerfile is the recipe for an image (base system plus install steps). | Tasks define their environment as a Dockerfile. Clusters rarely allow Docker itself, because its daemon runs as root. |
| **[Apptainer](https://apptainer.org/)** (formerly Singularity) | A container runtime made for HPC. It runs as the ordinary user and stores an image as a single `.sif` file. | It is usually the only container runtime a cluster offers. |
| **SIF** | Apptainer's image file. It is built for one CPU architecture. | You build one per task (or per shared environment) and cache it. |
| **Rootless** | Running and building containers without administrator rights. | It is what you get on a shared cluster. Some things a task may want (creating users, `chown`, `chroot`) do not work. |
| **Slurm** | The cluster's scheduler. You submit a job script with `sbatch`; it runs on a compute node when resources are free. | Everything heavy runs as a Slurm job, never on the login node. |
| **[vLLM](https://docs.vllm.ai/)** | A server that loads model weights onto a GPU and exposes an OpenAI-compatible HTTP API. | It serves the model that the agent calls. |
| **LiteLLM** | A client library that speaks to many model APIs through one interface. | Harbor's agents use it; a local vLLM server is addressed as `hosted_vllm/<name>`. |
| **Namespaces** | A Linux kernel feature that gives a process its own view of something: mounts, process IDs, the network. | Containers are built from them. A network namespace with no interfaces is the cleanest way to cut off the network, if your cluster allows it. |
| **[Landlock](https://landlock.io/)** | A Linux security feature that lets an unprivileged process restrict itself: which files it may touch and whether it may open TCP connections. The restriction is inherited by everything it starts and cannot be removed. | Sandboxing without root. |
| **seccomp** | A Linux feature that filters which system calls a process may make. | Blocks what Landlock does not cover, such as UDP sockets, `mount` and `ptrace`. |
| **[Monty](https://github.com/pydantic/monty)** | A small Python interpreter written in Rust that runs a piece of Python with no access to files, network or subprocesses. | Very fast and safe for scoring a single pure function. It cannot run shell tasks. |
| **Wheelhouse** | A folder of Python packages (`.whl` files) that `pip` can install from without the internet. | Lets tasks `pip install` when the network is blocked. |
| **uv** | A fast Python package and environment manager. | We use it to create the environment Harbor runs in. |

## How the pieces fit together

```text
                       one Slurm job on one node
 ┌──────────────────────────────────────────────────────────────────┐
 │  vLLM server  ◄──── HTTP ────  Harbor + agent (Terminus-2)       │
 │  (GPU, model)                        │                           │
 │                                      │ HTTP, token-protected     │
 │                                 bridge server                    │
 │                                      │                           │
 │                                   worker ── apptainer exec ──┐   │
 │                                                              ▼   │
 │        task container (SIF): shell, tmux, task files, verifier   │
 │        every command wrapped in Landlock + seccomp               │
 └──────────────────────────────────────────────────────────────────┘
```

The agent and the model server run **outside** the task container. Only the commands the model chooses, and the verifier, run inside it. This matters for safety: the container needs no network to talk to the model.

Harbor's Apptainer backend is a small bridge: Harbor sends "start environment", "run this command", "upload these files" to a bridge server, and a worker on the node carries them out with `apptainer`. We run all of it inside one job, bound to the loopback address and protected by a random token, because other users share the node.

## How we run things

Paths below are placeholders. Keep everything on project storage and point every cache there; home directories on clusters are small and fill up quickly.

### 1. One folder, one environment file

```bash
ROOT=/path/on/project/storage/rl-envs
mkdir -p $ROOT/{datasets,sif-cache,jobs,logs,scripts,.cache}
```

An `env.sh` that every script sources sets the cache locations (`UV_CACHE_DIR`, `HF_HOME`, `PIP_CACHE_DIR`, `APPTAINER_CACHEDIR`), the image cache (`SIF_CACHE`) and the Harbor checkout, and activates the Python environment.

### 2. Download the datasets, without the images

```bash
hf download laion/endless-terminals-rootless-apptainer --repo-type dataset \
    --local-dir $ROOT/datasets/endless-terminals --exclude '*.sif' --exclude 'tasks/*'
cd $ROOT/datasets/endless-terminals
sha256sum --quiet --ignore-missing -c SHA256SUMS
mkdir extracted && tar --use-compress-program=zstd -xf tasks.tar.zst -C extracted
```

We skip the `.sif` files because they are aarch64, and the loose `tasks/` tree because it duplicates the archive. All twelve datasets came to 2.2 GiB this way.

### 3. Install the runner

The datasets were validated with a fork of Harbor that adds the Apptainer backend ([marianna13/harbor](https://github.com/marianna13/harbor)). It needs Python 3.12 or later.

```bash
git clone https://github.com/marianna13/harbor.git $ROOT/harbor && cd $ROOT/harbor
uv venv --python 3.12 $ROOT/venv && source $ROOT/venv/bin/activate
uv sync --active --frozen
```

Two branches matter. `main` has the image builder that understands heredocs in Dockerfiles, which most of the "rootless" releases need. `jureca-eval-fixes` is the one named on the Terminal-Bench 2.1 card. The dataset cards do not state which revision validated the rootless releases, so our choice of `main` is an inference from what worked.

### 4. Build images in a job

```bash
python -m harbor.environments.apptainer.prebuild_sifs $TASKS --sif-cache $SIF_CACHE \
    --max-workers 4 --build-timeout-sec 1800
```

Add `--dry-run` first to see how many unique images a dataset needs. Run real builds in a CPU job on a node with internet access. Rootless builds worked for us without any special setup.

### 5. Validate with the two scripted agents

Inside the job, start the bridge server and worker, then:

```bash
umask 022
python -m harbor.cli.main run --path $TASKS --agent oracle --env apptainer \
    --environment-kwarg bridge_url=$BRIDGE_URL --environment-kwarg sif_cache=$SIF_CACHE \
    --jobs-dir $RUN/oracle --n-concurrent 4 --yes
# repeat with --agent nop
```

Expect reward 1 from `oracle` and 0 from `nop` on every task. Each trial leaves a folder with the agent log, the verifier output and `verifier/reward.txt`.

### 6. Run a model

Same job shape, on a GPU node, with a model server started first:

```bash
vllm serve $MODEL_DIR --served-model-name my-model --host 127.0.0.1 --port $PORT --max-model-len 65536 &

python -m harbor.cli.main run --path $TASKS --agent terminus-2 --model hosted_vllm/my-model \
    --ak api_base=http://127.0.0.1:$PORT/v1 --ak max_turns=30 \
    --ak 'model_info={"max_input_tokens":65536,"max_output_tokens":8192,"input_cost_per_token":0,"output_cost_per_token":0}' \
    --env apptainer --environment-kwarg bridge_url=$BRIDGE_URL --environment-kwarg sif_cache=$SIF_CACHE \
    --jobs-dir $RUN/terminus-2 --n-concurrent 8 --yes
```

`model_info` tells LiteLLM the context size of a model it has never heard of. Start with a low turn cap and a dozen tasks.

## Safety and sandboxing

### What a container does and does not give you

A container is packaging, not a security boundary. On our cluster, a task container started the way Harbor starts it could list the whole shared project filesystem and open a TCP connection to the public internet. Apptainer mounts whatever the site configuration says (for us, the project filesystem) and the current working directory, and it shares the host's network. Commands run as you, with your permissions.

That is acceptable for a reference solution you have read. It is not acceptable for commands written by a model.

### What we measured

We wrote a harmless probe as a task "solution" and ran it through Harbor with and without our sandbox, so the test goes through the same code path a model's commands take.

| Probe | Default | Sandbox on |
|---|---|---|
| Shared project filesystem | visible | not mounted |
| TCP connection to an outside host | open | denied |
| UDP socket (DNS) | allowed | denied |
| System-call filter | none | active |
| Writing inside the task's own directories | works | works |

### The sandbox we use

Three layers, all without root:

1. **Do not mount what the task does not need.** `apptainer ... --no-mount /shared/path,cwd` removes the site-wide bind and the working-directory bind. Removing only the first is not enough: if the runner's working directory is on the shared filesystem, Apptainer mounts it anyway.
2. **Landlock on every command.** A small launcher applies a Landlock ruleset to itself and then starts the real command. It denies all TCP bind and connect, and any path under the shared filesystem that is still visible.
3. **seccomp on every command.** The same launcher installs a filter that allows only Unix-domain sockets and denies calls such as `mount`, `ptrace`, `unshare` and `bpf`. It also sets `no_new_privs` and drops capabilities.

The launcher **fails closed**: if any protection cannot be applied, it exits without running the command. We hook it into the runner in one place, the function that executes a command in the container, and switch it on with an environment variable so the default behaviour is unchanged. The model-rollout job script forces it on.

### What the sandbox costs, and how to pay for it

- **`pip install` stops working**, because the network is gone. Two of our first twelve tasks failed for this reason. The fix is an offline wheelhouse: the dataset lists the exact wheels its tasks need, we downloaded the x86_64 builds of the same versions, mounted the folder read-only and pointed `pip` at it (`PIP_NO_INDEX=1`, `PIP_FIND_LINKS=...`). All twelve then passed.
- **Loopback is blocked too.** Landlock's TCP rules are per port, not per address, so "loopback only" cannot be expressed. Tasks that start a local web server will fail. A private network namespace would solve this properly, but our cluster does not allow unprivileged users to create one.
- **A later command cannot signal an earlier one.** Each command is its own Landlock domain, so `pkill` on a process started by a previous command is refused.
- **Harness steps are not wrapped.** File upload and download, and a dataset's own pre-agent setup script, still run unsandboxed. They are trusted code, but know that they are outside the fence.

### Where Monty fits

Monty runs one pure Python function in an interpreter that has no files, network or subprocesses. For environments where the model writes a single function and a script compares outputs, it is fast and strong. It cannot run a shell session, install packages or start a service, so it is not an option for terminal tasks. Use Monty for function-level tasks and an operating-system sandbox for everything else.

### Practice we follow

- **Strict first, loosen later.** Start with no network and no shared filesystem. Open one specific thing when a task set needs it, and write down why.
- **Scripted agents before models.** The oracle and no-op runs exercise the whole pipeline with commands you can read.
- **Test the sandbox on the real code path**, with a probe you wrote, and keep the before-and-after output.
- **Fail closed.** A sandbox that silently falls back to "no sandbox" is worse than none, because you stop looking.
- **No secrets in the job.** Clean environment, no tokens, no SSH agent, home directory not mounted.
- **Least access for the model server too.** It needs to read the weights; the task containers do not need to see it at all.
- **Small first.** A dozen tasks and a turn cap before hundreds.
- **Read trajectories.** A reward tells you the verifier was satisfied, not how.
- **Ask the operators.** Whether you may run model-written commands on compute nodes, and under what isolation, is the cluster operators' decision. Ask before you scale up, and tell them what isolation you have.
- **Know what the sandbox is not.** Landlock and seccomp share the host kernel with everything else on the node. They stop ordinary code; they are not a virtual machine. Do not run security or exploitation tasks on this setup.

## Results so far

### Scripted validation, four tasks per dataset

No sandbox, x86_64, images rebuilt from Dockerfiles. "n/a" means the dataset ships no reference solutions, so only the no-op run is possible.

| Dataset | Images built | Oracle passed | No-op rejected | Note |
|---|---|---|---|---|
| Endless Terminals | 4/4 | 4/4 | 4/4 | |
| Nemotron Terminal (easy) | 1/1 | 4/4 | 4/4 | |
| BugsInPy subset | 1/1 | 4/4 | 4/4 | |
| InferredBugs | 1/1 | n/a | 4/4 | |
| TaskTrove curriculum, easy | 1/1 | n/a | 4/4 | |
| TaskTrove curriculum, medium | 1/1 | n/a | 4/4 | |
| TMax legacy | 4/4 | n/a | 4/4 | Reference solutions are not public; builds took about 22 minutes |
| Recursive Task Synthesis | 2/4 | 2/2 | 2/2 | Two builds failed on an npm package that no longer resolves |
| SWE-Gym | 0/1 | not run | not run | Dockerfiles expect an offline wheel folder that is not in the release |
| R2E-Gym | 0/1 | not run | not run | The host's `fakeroot` cannot run inside the old `python:3.6-slim-buster` base |
| OpenThoughts-TBLite | 4/4 | 2/4 | 3/4 | Graders need a pre-downloaded Python 3.13; one task passed with no action, which the card says should not happen |
| Terminal-Bench 2.1 | 4/4 | 1/4 | 4/4 | Same grader dependency |

Four tasks per dataset is a smoke test, not a validation. It shows which datasets run as they are and which need work.

### With the sandbox on (Endless Terminals, 12 tasks)

| Configuration | Oracle passed | No-op rejected |
|---|---|---|
| Sandbox, no wheelhouse | 10/12 | 12/12 |
| Sandbox plus offline wheelhouse | 12/12 | 12/12 |

### First model rollout

Nemotron 3.5 Lightning (30B, served with vLLM on one GPU), Terminus-2, 30-turn cap, sandbox and wheelhouse on, twelve Endless Terminals tasks. The run took under seven minutes.

| Outcome | Tasks |
|---|---|
| Solved | 10 |
| Failed (verifier rejected the result) | 1 |
| Could not start (image had no `tmux`) | 1 |

The failure was the model installing a package together with its dependencies when the grader allowed only the listed packages. The task that could not start is an image problem, not a model problem. Twelve tasks from the easiest dataset say that the pipeline works; they say little about the model.

## Things that went wrong

Each of these cost time. They are listed so you can check for them first.

| Symptom | Cause | Fix |
|---|---|---|
| Released images will not run | Built for aarch64; our nodes are x86_64 | Rebuild from the Dockerfiles and re-validate |
| Image build fails with a Python `IndentationError` | The builder on one branch cannot convert `RUN ... <<EOF` heredocs | Use the branch whose builder handles heredocs |
| Reference solution fails a file-permission check | Cluster default `umask` is 0002; tasks assume Docker's 022 | `umask 022` before starting the worker |
| "0 tasks found" | The task scanner does not follow symbolic links | Copy task folders instead of linking them |
| Dataset download stops with HTTP 429 | Anonymous Hugging Face rate limit on datasets with thousands of small files | Retry (downloads resume), or authenticate |
| Hundreds of builds fail with `TOOMANYREQUESTS` | Every build asks Docker Hub for the base image manifest; the anonymous limit is about 100 pulls per six hours | Pull the base image through a public mirror, or build it once to a local file. In a four-task test the mirror route got past the limit; it is not yet tried at full scale |
| One build hangs for many minutes inside `apt-get install` | A stalled connection to the distribution's package mirror; every task image runs its own `apt-get` | Keep a build timeout, retry, and limit parallel builds. A shared base image with the common packages would avoid most of these downloads |
| A handful of builds fail at the very start with a JSON error | Many parallel builds race on Apptainer's shared download cache | Warm the cache with one build before starting the rest |
| `fakeroot` fails inside the image | The host's `fakeroot` binary needs a newer C library than an old base image has | Not solved yet |
| Graders fail with "No interpreter found for Python 3.13" | The grader expects a pre-downloaded Python and package mirror that is not redistributed | Build that mirror, as the dataset card describes |
| Agent cannot start: "tmux: command not found" | The image build installs tmux on a best-effort basis and it silently failed | Check for tmux after building; treat its absence as a build failure |
| Home directory over quota | Tools cache into `$HOME` by default | Point every cache at project storage |

## Checklist before a model writes its first command

- Reference solutions pass and the no-op agent is rejected on the tasks you will use.
- You have probe output showing what a command in the container can reach.
- Shared storage is not mounted in the task container.
- The network is blocked, or goes through a proxy you control.
- The sandbox fails closed, and you have seen it do so.
- The job has no credentials in its environment.
- There is a turn cap, a time limit and a small task count.
- The cluster operators know what you are running.
- Someone will read the trajectories afterwards.

## Limits of this write-up

- Everything here is from one day of work on one cluster. Other clusters differ in what they mount, whether compute nodes reach the internet, and which namespaces users may create.
- Only Endless Terminals has been run with the sandbox on. The other datasets have only the four-task smoke test without it.
- The full 1,000-task validation has not completed; the first attempt was stopped by the Docker Hub rate limit.
- The sandbox launcher and the hook into the runner are local changes. They are not published, and nobody outside the project has reviewed them.
- We have not read the model's trajectories in detail, so we cannot yet say how often the sandbox restrictions cost it turns.
- The datasets, the protocol and the Harbor fork are other people's work; see [the external resource page](../external/marianna13-rl-validation.md).

## Sources

- [OT validated RL datasets](https://huggingface.co/collections/laion/ot-validated-rl-datasets) and the [validation protocol](https://gist.github.com/marianna13/f9bc94d2ecb39c302c6c9b540ea6100a), by Marianna13.
- [Harbor](https://github.com/harbor-framework/harbor) and the [fork with the Apptainer backend](https://github.com/marianna13/harbor).
- [Apptainer documentation](https://apptainer.org/docs/user/latest/), [Landlock](https://landlock.io/), [vLLM](https://docs.vllm.ai/), [Monty](https://github.com/pydantic/monty).

## Related

- [Validating RL environments](../external/marianna13-rl-validation.md): the datasets and the protocol.
- [RL rollouts with internet access](rl-rollouts-with-internet.md): design principles for sandbox, proxy and monitoring.
- [Verifiable Swedish RL environments and baselines](../contributors/swedish-rl-environment-baselines.md): function-level environments, where Monty applies.
