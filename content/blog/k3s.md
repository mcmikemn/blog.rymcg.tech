---
title: "How to create a single node K3s (kubernetes) cluster for development or small apps"
url: "blog/k3s"
date: 2020-10-30T11:45:03-07:00
tags: ['k3s', 'kubernetes']
---
## Abstract

 * You will create a single node [k3s](https://k3s.io) droplet on
   [DigitalOcean](https://m.do.co/c/d827a13964d7) (this link includes my
   referral code, which will help support this blog if you sign up for a new
   account.)
 * This can be used for small self-hosted apps and development purposes.
 * You will create an attached volume for pod storage.
 * You will set up a DigitalOcean firewall.
 * You will install [Traefik](https://traefik.io) (v2) as an ingress controller
   for your cluster. This allows you to expose your pods to the internet and
   automatically generate ACME (Let's Encrypt) certificates for TLS/SSL
   encrypted HTTP(s).
 * You will install the [whoami](https://github.com/traefik/whoami) service
   which is a simple debug HTTP server that prints some information. It will
   help you test that networking and TLS is working, and form the simplest model
   for how to deploy other kinds of apps.
 * These same instructions can easily be adapted to Raspberry Pi or other raw
   metal as supported by k3s.
 
Self-hosting k3s as a droplet is considerably less expensive than
managed/enterprise kubernetes solutions, like the one DigitalOcean and other
providers offer, as this will not incur the additonal cost of a Load Balancer
node for ingress. (You can run all of this on a single $5 droplet.) This makes
it a good fit for development, or for small deployments, where you don't care
about high availability (multi-node redundancy.)

## Create Droplet

 * Create a Debian (`10 x64`) droplet on DigitalOcean
   * $10/mo 2GB RAM (tested configuration)
   * Add a block storage volume for pod data
     * You choose how much space you need for all of your pods.
     * Choose `Manually Format & Mount` (we want to customize the mount point,
       so the following script will take care of formatting and mounting
       `/dev/sda` which is the device name for the volume this creates.)
   * Enter the following script into the `User data` section of the droplet creation screen:
   
   ```bash
   #!/bin/bash
   VOLUME=/dev/sda
   umount $VOLUME
   if (! blkid $VOLUME); then 
     mkfs.ext4 $VOLUME
   fi
   mkdir -p /opt/local-path-provisioner
   echo "$VOLUME /opt/local-path-provisioner " \
        "ext4 defaults,nofail,discard 0 0" | sudo tee -a /etc/fstab
   mount $VOLUME
   apt-get update -y
   apt-get install -y curl
   ```
 * Assign your ssh key
 * Confirm details and create the droplet.
 * From the DigitalOcean platform, [create a
   firewall](https://cloud.digitalocean.com/droplets/new):
 
   * Give it a name, maybe `k3s`
 
   * Open TCP port 22 for SSH
   
   * Open TCP port 80 for unencrypted HTTP
   
   * Open TCP port 443 for TLS encrypted HTTPS
   
   * Open TCP port 6443 for Kubernetes (kubectl) access
   
   * **Apply the firewall to your droplet or tag.**
   
   * From the droplet page, under `Networking` you can confirm that the firewall
     rules are applied to the droplet.
   
 * Assign a [floating IP
   address](https://cloud.digitalocean.com/networking/floating_ips)
   
 * [Create wildcard DNS](https://cloud.digitalocean.com/networking/domains)
   names pointing to floating IP address (`subdomain.example.com` and `*.subdomain.example.com`)
   
## Prepare your workstation

When working with kubernetes, you should eschew directly logging into the
droplet via ssh, unless you have to. Instead, we will create all files and do
all of the setup, indirectly, from your local laptop, which will be referred to
as your workstation. `kubectl` is our local tool to access the cluster.

 * Install kubectl on your workstation:
 
   * Prefer your os packages:
   
     * Arch: `sudo pacman -S kubectl`
     
     * Ubuntu and Other OS: [See docs](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-using-native-package-management)
     
     * You can take a small detour now and setup bash shell completion for
       kubectl, this is quite useful. Run `kubectl completion -h` and follow the
       directions for setting up your shell. However, you can skip this, it's
       not required.
       
 * Install [k3sup](https://github.com/alexellis/k3sup) (a remote k3s installer tool) on your workstation:
 ```bash
 curl -sLS https://get.k3sup.dev | sh
 sudo install k3sup /usr/local/bin/
 ```
 
 * `k3sup` will install k3s on your droplet using the SSH key you defined during
   droplet creation. `k3sup` will also take care of installing another key file
   required for `kubectl` to access your cluster. That key is created locally in
   your `$HOME/.kube` directory, which you must keep safe, as it provides full
   access to your cluster.

## Create the cluster

 * Create the cluster using k3sup, it will ask you to type/paste in your
   Droplet's Floating IP address:
 
 ```bash
 mkdir -p ${HOME}/.kube && \
   read -p "Enter the droplet Floating IP address: " IP_ADDRESS && \
   k3sup install --ip ${IP_ADDRESS} --k3s-extra-args '--no-deploy traefik' \
       --local-path ${HOME}/.kube/config
 ```
 * Test kubectl :
 
 ```bash
 kubectl get node -o wide
 ```
(It should print the node status as `Ready` before you proceed,
   be patient.)

 * Install the
   [local-path-provisioner](https://github.com/rancher/local-path-provisioner)
   which manages the volumes created for Persistent Volume Claims, and stores
   them on our droplet's external volume:
   
 ```bash
 kubectl apply -f https://git.io/JvdvR
 ```
 
 * Check the `local-path-provisioner` pod is installed with status `Running`:
 
 ```bash
 kubectl -n local-path-storage get pod
 ```
 
## Deploy Traefik on k3s

On your workstation, create a new directory (anywhere) to store the YAML files
for this deployment.

```bash
DIR=$HOME/git/k3s
mkdir -p $DIR
cd $DIR
```

Download the [environment file
template](https://raw.githubusercontent.com/EnigmaCurry/blog.rymcg.tech/master/src/k3s/env.sh):

```bash
curl -Lo env.sh https://git.io/JTQGD
```

Edit the file `env.sh` and change the variables according to your own
environment. Required parameters to change: `ACME_EMAIL`, `ACME_SERVER`, `WHOAMI_DOMAIN`

 * `ACME_EMAIL` is required, it is your personal/work email address that you
   will be sending to Let's Encrypt. You will get emails every 3 months when
   your certificate automatically renews, as well as other reminders if
   something goes wrong.
 * `ACME_SERVER` is the API URL of the Let's Encrypt service to generate
   certificates. The default is to use the staging server, which will generate
   certificates that are not valid in your web browser, but are useful for
   testing. **You should leave it as the staging URL when you first test this.**
   Once you have tested it is working, you can change it to the production URL
   and redeploy. (This is important because Lets Encrypt [rate
   limits](https://letsencrypt.org/docs/rate-limits/) the production
   certificates generation and you want to make sure you get it right the first
   time.) The production Let's Encrypt `ACME_SERVER` URL is :
   
   * `https://acme-v02.api.letsencrypt.org/directory`

 * `WHOAMI_DOMAIN` this is the domain name for the included `whoami` service.
   The `whoami` service is a simple HTTP server that you deploy inside a pod.
   This service only displays a single debug page. The purpose of which, is to
   test that Traefik is functional, and able to expose internet web sites.
   Suppose you had a domain name for your cluster as `k3s.example.com` you would
   create `WHOAMI_DOMAIN` as a subdomain of this, for example
   `whoami.k3s.example.com`

You can find all of the [prepared YAML templates in the git repostory for this
website](https://github.com/EnigmaCurry/blog.rymcg.tech/tree/master/src/k3s).
Use this bash script to automate the download and rendering of these templates:

```bash
curl -Lo render.sh https://git.io/JTQG5
chmod 0755 render.sh
```

Execute the `render.sh` script in order to: download the YAML templates, replace
variables in the templates from your `env.sh`, and produce the final kubernetes
manifest YAML files. Run:

```bash
./render.sh
```

You should now have a number of new YAML files rendered and ready to apply to
your cluster. Examine them, and when you're satisfied they are correct, apply
them with `kubectl`:

```bash
kubectl apply -f traefik.pvc.yaml
kubectl apply -f traefik.yaml
kubectl apply -f whoami.yaml
```

Keep these files safe, a private git repository is recommended! You can use them
again in the future, to deploy to another cluster.

## Test that it works

Open your web browser to the domain you chose for `WHOAMI_DOMAIN`. If it is
working correctly, you should see some text like this:

```
Hostname: whoami-678c86b5c7-ghb5r
IP: 127.0.0.1
IP: ::1
IP: 10.42.0.21
IP: fe80::8c0a:4aff:feba:ad8c
RemoteAddr: 10.42.0.22:36564
GET / HTTP/1.1
Host: whoami.collab.rymcg.tech
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/86.0.4240.111 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Accept-Encoding: gzip, deflate, br
Accept-Language: en-US,en;q=0.9,fr;q=0.8
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: none
Sec-Fetch-User: ?1
Upgrade-Insecure-Requests: 1
X-Forwarded-For: xxxxxxxxxxxx
X-Forwarded-Host: whoami.collab.rymcg.tech
X-Forwarded-Port: 443
X-Forwarded-Proto: https
X-Forwarded-Server: traefik-nrx8w
X-Real-Ip: xxxxxx
```

If you chose the production `ACME_SERVER`, the TLS/SSL certificate should be
valid, and you see a Lock icon in your browser URL bar, if you do, you're good
to go!

If you left the `ACME_SERVER` as the default staging server, you will instead
get an error about the certificate being invalid, which is to be expected until
you switch to the production `ACME_SERVER`. Your browser should still allow you
to confirm an exception for the warning, and view the `whoami` output anyway. If
you check the certificate details in your browser URL bar, you should find that
the Common Name (CN) is listed as `Fake LE Intermediate X1` which is the name
that the Let's Encrypt service (`LE`) assigns it. It could also say ```TRAEFIK
DEFAULT CERT```. If it says this, then wait a few minutes, and it might still
change, but if it persists, then something has gone wrong with the ACME process.
Double check your `env.sh` and check the traefik logs (see next section). As
long as it says `Fake LE Intermediate X1` you can assume that everything would
work correctly if you were to move to the production `ACME_SERVER`.

## Checking the logs

You can monitor the Traefik logs to view any errors regarding certificate
generation:

```bash
kubectl -n kube-system logs -f daemonset/traefik
```
(Press Ctrl-C anytime to quit monitoring the logs)


## Changing parameters and redeploying

To change settings, you go through the same process again. For example, if you
wish to update `ACME_SERVER` from the staging to the production URL:
 * Edit `env.sh` and update `ACME_SERVER` to:
   * `https://acme-v02.api.letsencrypt.org/directory`
 * Delete the existing `traefik.yaml`
 * Re-run `./render.sh` to make the new `traefik.yaml`
 * Re-deploy with `kubectl apply -f traefik.yaml` 
 * The magic of Kubernetes will automatically restart traefik and apply the new
   setting.

## Future

Check back for future blog posts that will add more functionality to this
cluster. You can use the `whoami.yaml` as a template for your other services.
Try it yourself by making a `whoami2` service with a different domain. 