# My Homelab Cluster 🏍

## 📖 Overview

What pushed me to invest my time and money into creating a homelab
- Smart Home
  - I already had Home Assistant running on a Raspberry Pi 4
  - but I was aware that a single hardware failure would stop my lights from working
- Personal projects
  -
- Bootstrapping new projects
-

### Hardware

| Role                | Hardware               | RAM  | System Disk |
|---------------------|------------------------|------|-------------|
| Router              | Mikrotik RB5009UG+S+IN | 1GB  | 1GB         |
| Virtualization host | Eglobal F9 i7-1260P    | 64GB | 2Tb Nvme    |

## 👋 Introduction

The goal of this project is to make it easy for people interested in learning Kubernetes to deploy a basic cluster at home and become familiar with the GitOps tool Flux.

This template implements Flux in a way that promotes legibility and ease of use for those who are new (or relatively new) to the technology and GitOps in general. It assumes a typical homelab setup: namely, a single "home prod" cluster running mostly third-party apps.

## ✨ Features

- Automated, reproducible, customizable setup through Ansible templates and playbooks
- Opinionated implementation of Flux with [strong community support](https://github.com/onedr0p/flux-cluster-template#-support)
- Encrypted secrets thanks to [SOPS](https://github.com/getsops/sops) and [Age](https://github.com/FiloSottile/age)
- Web application firewall thanks to [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps)
- SSL certificates thanks to [Cloudflare](https://cloudflare.com) and [cert-manager](https://cert-manager.io)
- HA control plane capability thanks to [kube-vip](https://kube-vip.io)
- Next-gen networking thanks to [Cilium](https://cilium.io/)
- A [Renovate](https://www.mend.io/renovate)-ready repository
- Integrated [GitHub Actions](https://github.com/features/actions)

... and more!

## 📝 Pre-start checklist

Before we get started everything below must be taken into consideration, you must...

- [ ] bring a **positive attitude** and be ready to learn and fail a lot. _The more you fail, the more you can learn from._
- [ ] run the cluster on bare metal machines or VMs within your home network &mdash; **this is NOT designed for cloud environments**.
- [ ] have Debian 12 freshly installed on 1 or more AMD64/ARM64 bare metal machines or VMs. Each machine will be either a **control node** or a **worker node** in your cluster.
- [ ] give your nodes unrestricted internet access &mdash; **air-gapped environments won't work**.
- [ ] have a domain you can manage on Cloudflare.
- [ ] be willing to commit encrypted secrets to a public GitHub repository.
- [ ] have a DNS server that supports split DNS (e.g. Pi-Hole) deployed somewhere outside your cluster **ON** your home network.


### 🚀 Proxmox


## 📣 Post installation

#### 🌐 Public DNS

The `external-dns` application created in the `networking` namespace will handle creating public DNS records. By default, `echo-server` and the `flux-webhook` are the only subdomains reachable from the public internet. In order to make additional applications public you must set the correct ingress class name and ingress annotations like in the HelmRelease for `echo-server`.

#### 🏠 Home DNS

`k8s_gateway` will provide DNS resolution to external Kubernetes resources (i.e. points of entry to the cluster) from any device that uses your home DNS server. For this to work, your home DNS server must be configured to forward DNS queries for `${bootstrap_cloudflare_domain}` to `${bootstrap_k8s_gateway_addr}` instead of the upstream DNS server(s) it normally uses. This is a form of **split DNS** (aka split-horizon DNS / conditional forwarding).

📍 _Below is how to configure a Pi-hole for split DNS. Other platforms should be similar._

1. Apply this file on the server

   ```sh
   # /etc/dnsmasq.d/99-k8s-gateway-forward.conf
   server=/${bootstrap_cloudflare_domain}/${bootstrap_k8s_gateway_addr}
   ```

2. Restart dnsmasq on the server.

3. Query an internal-only subdomain from your workstation: `dig @${home-dns-server-ip} echo-server.${bootstrap_cloudflare_domain}`. It should resolve to `${bootstrap_internal_ingress_addr}`.

If you're having trouble with DNS be sure to check out these two GitHub discussions: [Internal DNS](https://github.com/onedr0p/flux-cluster-template/discussions/719) and [Pod DNS resolution broken](https://github.com/onedr0p/flux-cluster-template/discussions/635).

... Nothing working? That is expected, this is DNS after all!

#### 📜 Certificates

By default this template will deploy a wildcard certificate using the Let's Encrypt **staging environment**, which prevents you from getting rate-limited by the Let's Encrypt production servers if your cluster doesn't deploy properly (for example due to a misconfiguration). Once you are sure you will keep the cluster up for more than a few hours be sure to switch to the production servers as outlined in `config.yaml`.

📍 _You will need a production certificate to reach internet-exposed applications through `cloudflared`._

#### 🪝 Github Webhook

By default Flux will periodically check your git repository for changes. In order to have Flux reconcile on `git push` you must configure Github to send `push` events.

1. Obtain the webhook path

    📍 _Hook id and path should look like `/hook/12ebd1e363c641dc3c2e430ecf3cee2b3c7a5ac9e1234506f6f5f3ce1230e123`_

    ```sh
    kubectl -n flux-system get receiver github-receiver -o jsonpath='{.status.webhookPath}'
    ```

2. Piece together the full URL with the webhook path appended

    ```text
    https://flux-webhook.${bootstrap_cloudflare_domain}/hook/12ebd1e363c641dc3c2e430ecf3cee2b3c7a5ac9e1234506f6f5f3ce1230e123
    ```

3. Navigate to the settings of your repository on Github, under "Settings/Webhooks" press the "Add webhook" button. Fill in the webhook url and your `bootstrap_flux_github_webhook_token` secret and save.

### 🤖 Renovate

[Renovate](https://www.mend.io/renovate) is a tool that automates dependency management. It is designed to scan your repository around the clock and open PRs for out-of-date dependencies it finds. Common dependencies it can discover are Helm charts, container images, GitHub Actions, Ansible roles... even Flux itself! Merging a PR will cause Flux to apply the update to your cluster.

To enable Renovate, click the 'Configure' button over at their [Github app page](https://github.com/apps/renovate) and select your repository. Renovate creates a "Dependency Dashboard" as an issue in your repository, giving an overview of the status of all updates. The dashboard has interactive checkboxes that let you do things like advance scheduling or reattempt update PRs you closed without merging.

The base Renovate configuration in your repository can be viewed at [.github/renovate.json5](https://github.com/onedr0p/flux-cluster-template/blob/main/.github/renovate.json5). By default it is scheduled to be active with PRs every weekend, but you can [change the schedule to anything you want](https://docs.renovatebot.com/presets-schedule), or remove it if you want Renovate to open PRs right away. It is also configured to [automerge some updates](https://github.com/onedr0p/flux-cluster-template/blob/main/.github/renovate/autoMerge.json5).

## 🐛 Debugging

Below is a general guide on trying to debug an issue with an resource or application. For example, if a workload/resource is not showing up or a pod has started but in a `CrashLoopBackOff` or `Pending` state.

1. Start by checking all Flux Kustomizations & Git Repository & OCI Repository and verify they are healthy.

    ```sh
    flux get sources oci -A
    flux get sources git -A
    flux get ks -A
    ```

2. Then check all the Flux Helm Releases and verify they are healthy.

    ```sh
    flux get hr -A
    ```

3. Then check the if the pod is present.

    ```sh
    kubectl -n <namespace> get pods -o wide
    ```

4. Then check the logs of the pod if its there.

    ```sh
    kubectl -n <namespace> logs <pod-name> -f
    # or
    stern -n <namespace> <fuzzy-name>
    ```

5. If a resource exists try to describe it to see what problems it might have.

    ```sh
    kubectl -n <namespace> describe <resource> <name>
    ```

6. Check the namespace events

    ```sh
    kubectl -n <namespace> get events --sort-by='.metadata.creationTimestamp'
    ```

Resolving problems that you have could take some tweaking of your YAML manifests in order to get things working, other times it could be a external factor like permissions on NFS. If you are unable to figure out your problem see the help section below.

## 👉 Help

- Make a post in this repository's Github [Discussions](https://github.com/onedr0p/flux-cluster-template/discussions).
- Start a thread in the `support` or `flux-cluster-template` channel in the [k8s@home](https://discord.gg/k8s-at-home) Discord server.

## ❔ What's next

The cluster is your oyster (or something like that). Below are some optional considerations you might want to review.

#### Ship it

To browse or get ideas on applications people are running, community member [@whazor](https://github.com/whazor) created [this website](https://nanne.dev/k8s-at-home-search/) as a creative way to search Flux HelmReleases across Github.

#### Storage

The included CSI (`local-path-provisioner`) is a great start for storage but soon you might find you need more features like replicated block storage, or to connect to a NFS/SMB/iSCSI server. If you need any of those features be sure to check out the projects like [rook-ceph](https://github.com/rook/rook), [longhorn](https://github.com/longhorn/longhorn), [openebs](https://github.com/openebs/openebs), [democratic-csi](https://github.com/democratic-csi/democratic-csi), [csi-driver-nfs](https://github.com/kubernetes-csi/csi-driver-nfs),
and [synology-csi](https://github.com/SynologyOpenSource/synology-csi).

#### Authenticate Flux over SSH

Authenticating Flux to your git repository has a couple benefits like using a private git repository and/or using the Flux [Image Automation Controllers](https://fluxcd.io/docs/components/image/).

By default this template only works on a public Github repository, it is advised to keep your repository public.

The benefits of a public repository include:

- Debugging or asking for help, you can provide a link to a resource you are having issues with.
- Adding a topic to your repository of `k8s-at-home` to be included in the [k8s-at-home-search](https://nanne.dev/k8s-at-home-search/). This search helps people discover different configurations of Helm charts across others Flux based repositories.

<details>
  <summary>Expand to read guide on adding Flux SSH authentication</summary>

1. Generate new SSH key:

    ```sh
    ssh-keygen -t ecdsa -b 521 -C "github-deploy-key" -f ./kubernetes/bootstrap/github-deploy.key -q -P ""
    ```

2. Paste public key in the deploy keys section of your repository settings
3. Create sops secret in `./kubernetes/bootstrap/github-deploy-key.sops.yaml` with the contents of:

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: github-deploy-key
     namespace: flux-system
   stringData:
     # 3a. Contents of github-deploy-key
     identity: |
       -----BEGIN OPENSSH PRIVATE KEY-----
           ...
       -----END OPENSSH PRIVATE KEY-----
     # 3b. Output of curl --silent https://api.github.com/meta | jq --raw-output '"github.com "+.ssh_keys[]'
     known_hosts: |
       github.com ssh-ed25519 ...
       github.com ecdsa-sha2-nistp256 ...
       github.com ssh-rsa ...
   ```

4. Encrypt secret:

    ```sh
    sops --encrypt --in-place ./kubernetes/bootstrap/github-deploy-key.sops.yaml
    ```

5. Apply secret to cluster:

    ```sh
    sops --decrypt ./kubernetes/bootstrap/github-deploy-key.sops.yaml | kubectl apply -f -
    ```

6. Update `./kubernetes/flux/config/cluster.yaml`:

    ```yaml
    apiVersion: source.toolkit.fluxcd.io/v1beta2
    kind: GitRepository
    metadata:
      name: home-kubernetes
      namespace: flux-system
    spec:
      interval: 10m
      # 6a: Change this to your user and repo names
      url: ssh://git@github.com/$user/$repo
      ref:
        branch: main
      secretRef:
        name: github-deploy-key
    ```

7. Commit and push changes
8. Force flux to reconcile your changes

    ```sh
    flux reconcile -n flux-system kustomization cluster --with-source
    ```

9. Verify git repository is now using SSH:

    ```sh
    flux get sources git -A
    ```

10. Optionally set your repository to Private in your repository settings.

</details>

## 🤝 Thanks

Big shout out to all the contributors, sponsors and everyone else who has helped on this project.
