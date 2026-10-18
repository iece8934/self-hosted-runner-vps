# self hosted github runner: How to Set One Up on a VPS, What It Costs, and the Security Rules Most Guides Skip

A self-hosted GitHub runner is a machine you control — a VPS, a box under your desk, a Kubernetes pod — that runs GitHub Actions jobs instead of GitHub's shared hosted runners. You get unlimited build minutes on private repos, more CPU and RAM than the standard 2-core hosted machines, and a direct line to things hosted runners can never reach: internal databases, test hardware, private registries. The tradeoff is that you're now running CI infrastructure yourself. You pick the machine, patch the OS, and deal with GitHub's security warnings.

This guide covers the whole loop: how runner registration actually works, a step-by-step setup on a cheap VPS, the security rules that genuinely matter, and the point at which a $24/month server beats paying for hosted minutes.

> **Short version:** most people self-hosting a runner want a small always-on Linux VM. A Vultr instance with 2 vCPUs and 4 GB RAM at $24/month handles the majority of CI workloads, new accounts get up to $300 in credit for the first 30 days, and billing is hourly — so ephemeral runner fleets are cheap to experiment with. One hard rule first: only do this with **private repositories**. GitHub's own documentation warns that a fork of a public repo can open a pull request whose workflow executes arbitrary code on your machine. 👉 [check Vultr's current plan pricing](https://www.vultr.com/pricing/?ref=7268283)

## What a self-hosted GitHub runner actually does

It's an agent process — GitHub calls it the runner application — that connects to GitHub, listens for jobs tagged with its labels, and executes those jobs on your machine. Nothing about your workflows changes except one line: `runs-on`. Every self-hosted runner gets `self-hosted` plus its OS and architecture (`Linux`, `X64`) as automatic labels, so `[self-hosted, linux]` is enough to route jobs to it.

Registration happens through a config script. GitHub generates a one-time token in the UI, the script sends it along with your repo or org URL, and the runner shows up under Settings → Actions → Runners. Two permission details worth knowing: repo-level runners require admin access to that repository, and organization-level runners require org owner rights. The registration token itself expires **after one hour** — remember this, because it bites everyone who scripts their setup and comes back the next day.

Once running, the healthy state looks like this on the machine:


√ Connected to GitHub
Listening for Jobs


If you see those two lines, the runner is online and accepting work. The runner needs to stay online to receive jobs, which is why most people run it as a systemd service rather than a terminal window.

## Where to host it: Vultr plans and pricing

Any Linux VM with outbound internet access works as a runner host. Vultr is a good fit for a few structural reasons: flat monthly pricing across 30-plus data center regions, hourly billing capped at the monthly rate, and no per-minute CI metering at all. Here's the current lineup relevant to CI:

| Plan line | vCPU / RAM / Storage | Monthly | Hourly (approx.) | Deploy |
| --- | --- | --- | --- | --- |
| Regular Cloud Compute | 1 vCPU / 1 GB / 25 GB SSD | $5 | ~$0.007 | [ Deploy](https://www.vultr.com/pricing/?ref=7268283) |
| Regular Cloud Compute | 2 vCPU / 2 GB / 60 GB SSD | $12 | ~$0.018 | [ Deploy](https://www.vultr.com/pricing/?ref=7268283) |
| Regular Cloud Compute | 2 vCPU / 4 GB / 80 GB SSD | $20 | ~$0.030 | [ Deploy](https://www.vultr.com/pricing/?ref=7268283) |
| High Performance (AMD/Intel) | 1 vCPU / 1 GB / 25 GB NVMe | $6 | ~$0.009 | [ Deploy](https://www.vultr.com/pricing/?ref=7268283) |
| High Performance (AMD/Intel) | 2 vCPU / 4 GB / 75 GB NVMe | $24 | ~$0.036 | [ Deploy](https://www.vultr.com/pricing/?ref=7268283) |
| High Frequency | 2 vCPU / 4 GB / 80 GB NVMe | $24 | ~$0.036 | [ Deploy](https://www.vultr.com/pricing/?ref=7268283) |
| Optimized Cloud Compute (dedicated vCPU) | 1 vCPU / 4 GB / 25 GB NVMe | from $28 | ~$0.042 | [ Deploy](https://www.vultr.com/pricing/?ref=7268283) |
| Bare Metal | dedicated physical server | from $120 | ~$0.18 | [ Deploy](https://www.vultr.com/pricing/?ref=7268283) |

A few notes the table can't hold:

- There's also a $2.50/mo entry tier (1 vCPU, 512 MB RAM, 10 GB SSD), but it's only offered in some regions and 512 MB is genuinely tight for build workloads.
- Every family above scales up — 8, 16, 32 GB and beyond — with the same hourly billing model. The 4 GB tier is the practical floor for Docker-based CI; below that, builds fail in confusing ways.
- Outbound bandwidth is included at every tier (1 TB monthly at the 1 GB plans, scaling up from there), and each VM ships with an IPv4 address.
- New accounts get **up to $300 in credit valid for the first 30 days**, which comfortably covers a month of even the dedicated-CPU tiers while you figure out your real workload size.
- Windows VMs are available but carry an added license fee; macOS is not offered at all (more on why below).

## Setting up a self-hosted GitHub runner on a VPS, step by step

There are two routes: the official installer on a plain VM, or Docker images that register themselves. Start with the first — it takes about 15 minutes and teaches you what the container images do under the hood. If you don't have a machine yet, 👉 [create a Vultr account and deploy your first instance](https://bit.ly/VulTR) (Ubuntu 24.04, 2 vCPU / 4 GB is the sweet spot), then follow along.

1. **Open the runner wizard.** In your repository: Settings → Actions → Runners → New self-hosted runner. Pick your OS and architecture, and GitHub shows the exact commands for that machine.
2. **Download and extract the runner package** on your VPS using those commands.
3. **Configure it:**

   bash
   ./config.sh --url https://github.com/your-org/your-repo --token <TOKEN>
   

   The token is pre-generated in the wizard and valid for one hour. If it lapses mid-setup, just click "New self-hosted runner" again — a fresh token takes two seconds.
4. **Install it as a service** so it survives reboots:

   bash
   ./svc.sh install
   ./svc.sh start
   

   On Windows the config script offers this during setup instead, and GitHub recommends running the shell as administrator and keeping the application in `C:\actions-runner`.
5. **Verify.** You should see "Connected to GitHub" and "Listening for Jobs" in the logs, and the runner should appear with an Idle status in the Runners page.
6. **Point a workflow at it:**

   yaml
   jobs:
     build:
       runs-on: [self-hosted, linux]
       steps:
         - uses: actions/checkout@v4
         - run: echo "hello from my own machine"
   

### The Docker route

Community-maintained Docker images wrap the official runner application and register themselves on container start, which makes parallel runners trivial. The typical pattern: a `.env` file with the repo URL, a registration token, and a display name, then `docker compose up` with as many replicas as you want cores to spare. Two details make this setup production-grade:

- Setting an ephemeral flag makes each container register, run exactly one job, then deregister and die — a fresh, clean environment per job, much like GitHub-hosted runners.
- The shutdown scripts trap SIGINT/SIGTERM, so `docker compose down` deregisters runners cleanly instead of leaving zombie entries in your Runners list.

For larger fleets, Kubernetes-oriented images exist that register via a GitHub App and scale with plain Deployments — no controller or CRDs required. Whichever route you pick, the one-hour token expiry still applies, so automated deploys should mint a fresh token each run.

## The security rules most setup guides skip

GitHub's documentation is blunt about this: **self-hosted runners are recommended for private repositories only.** The reason is structural. Anyone can fork your public repo and open a pull request, and depending on your workflow triggers, that PR can execute workflow steps on your runner machine. On GitHub's infrastructure, that's a disposable sandbox. On your VPS, it's a machine with your credentials, network access, and Docker socket.

Practical mitigations, in order of impact:

- **Keep self-hosted runners on private repos**, full stop. Public-repo jobs belong on GitHub-hosted runners, where they're free anyway.
- **Run ephemeral runners.** One job per registration means a compromised job can't persist and harvest tokens from future jobs. This is the single biggest hardening win for the least effort.
- **If you automate registration with a GitHub App, grant the minimum.** Actions: Read & Write plus Metadata covers token minting. Don't grant Administration or workflow-write scopes — they're unnecessary for registration and dramatically widen the blast radius if the key leaks. Keep the App's private key off the runner itself; a sidecar that mints short-lived tokens keeps the runner container credential-free.
- **Network hygiene:** a runner needs only outbound HTTPS to GitHub. Don't open inbound ports for it. If your builds need Docker, isolate the Docker daemon (Docker-in-Docker in a locked-down container) so jobs can't reach the host's Docker socket.
- **Keep the runner application updated.** An outdated runner binary eventually gets deprecated by GitHub and the runner starts crash-looping — a failure mode that looks mysterious and is fixed by updating.

## Self-hosted vs GitHub-hosted minutes: the actual cost math

Self-hosted jobs don't consume Actions minutes at all — you pay for the machine and nothing per job. GitHub-hosted runners on private repos bill per minute once you exhaust your plan's included allowance: 2,000 minutes/month on the Free plan, 3,000 on Pro and Team, 50,000 on Enterprise. Public repos get hosted minutes free regardless.

At GitHub's published overage rate of $0.008/minute for standard Linux runners (about $0.48/hour), a $24/month Vultr instance crosses even at roughly **50 hours of Linux CI per month**. Windows hosted minutes run at $0.016/min, which drops that break-even to around 25 hours. Past those points, the VPS is strictly cheaper — and it usually has more RAM and disk than the standard hosted machines too.

Two honest caveats. First, hosted runners are zero-maintenance; your VPS is not, so factor an occasional hour of patching into the comparison. Second, macOS changes everything: Apple's licensing means no VPS provider can legally sell you a macOS VM, so macOS CI on your own infrastructure requires real Apple hardware. If your workload is macOS-heavy and small, hosted minutes at $0.08/min may simply be the rational choice.

If you want to see where the break-even lands for your actual usage, 👉 [compare Vultr's cloud compute plans side by side](https://www.vultr.com/pricing/?ref=7268283) and match your monthly CI hours against the hourly rate column.

## Self-hosted GitHub runner FAQ

**Does the runner software cost anything?**
No. The runner application is free. Your only cost is the machine it runs on, whether that's a $6 VPS or hardware you already own.

**Why does my registration token keep failing?**
Registration tokens expire one hour after generation. Generate a new one from the Runners page right before each setup or automated deploy.

**Can I run multiple runners on one VPS?**
Yes. Containerized runners with compose replicas are the standard way — GitHub automatically distributes queued jobs across every registered runner. Give each container its own resource limits so one build can't starve the others.

**Can I self-host a macOS runner on a VPS?**
No. Apple's licensing terms rule out macOS VMs at cloud providers, so self-hosted macOS CI requires Apple hardware. This is the one workload where staying on hosted runners is usually the only sane option.

**Does the VPS need to run 24/7?**
An always-on runner does, since it has to be connected to receive jobs. The upside of hourly billing with a monthly cap is that you pay the same whether the machine is busy or idle — and for bursty teams, ephemeral runner containers on hourly VMs let you scale capacity up during the day and tear it down after.

## Which setup makes sense for you

If your repo is public, stay on GitHub-hosted runners — free, sandboxed, zero maintenance. If your private repo fits inside your plan's included minutes, the same logic applies; there's nothing to optimize yet.

A self-hosted runner earns its keep once you hit any of these: more than ~50 hours of Linux CI monthly, Docker builds that need more than 7 GB of disk or 2 cores, or jobs that must touch your internal network. For that, one Vultr instance at $12–24/month with an ephemeral runner setup covers most teams, and the signup credit means your first month of figuring out the right size costs nothing.

Before you deploy, confirm two things: the repo is private, and you've scheduled the runner as a service so it survives reboots. Then 👉 [set up your runner on Vultr](https://bit.ly/VulTR) and point your first workflow at it.
