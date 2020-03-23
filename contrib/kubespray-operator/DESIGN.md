# Why 

Operating new cluster with kubespray can be difficult for adopters because of some pain points : 
- bootstraping a good ansible sandbox for running kubespray is not so easy : choosing bad combination on ansible version vs python version vs target OS version can lead two non trivial issue with bad ansible feedback (like erros on dict, missing requirements, libselinux not on Centos8, ...)
- ansible is not a workflow : unless you use external workflow tool (Jenkins, Awx, CDS, ..), dealing with necessary operations like scaling, upgrading, certifcates renewing has to be ran by hand, 

This leads to high learning curve and too many issues on first running for newly adopters. 

## High level API 

Anisble is anyway a very good (probably the best) tool to deal with operating kubernetes clusters. Some related projects, using ansible, trying to sove this : 
- https://github.com/wise2c-devops/breeze : with a GUI (for input control) + docker runner image as sandbox, 
- https://github.com/kubernauts/tk8 : with hosts.ini + some ConfigMaps (?)

Kubespray Operator aims to use Operator pattern with Custom Resource Definition as an alternative. Declaring somes Clusters and or Nodes objects, with a robbust and well desinged operator will offer a good way for people to use the full power of kubespray with ease. 

## Goals and non goals

TBD

## Workflows 

Creating an operator for kubespray means we can run different some operations splitted in small steps and make some decisions between steps.  

One dummy example could be on deployement operation. Instead of running one big playbook, Kubespray Operator could run some steps, including : 
- pinging nodes and make decisions of what to depending on operator configuration (ex: fail if > 20% not responding),
- test ssh connexion (with also conditions that can be defined for next steps), 
- bootsrap
- install etcds and masters in parralel if masters are not etcd, install sequencially else

Each of those steps would report steps in status to ease users unerstand whats going on ("pingin", "testing connexion") plus events ("WARN: 1 node overs 12 was node pingable"). 

Probably, kubespray operator should track also status of each nodes by creating dependent objects. 

# Sample CRDs

Here are sample CRDs, mainly inspired by what is done by the rook.io project. 

We coudl have two Kinds : 
- ClusterNode for inventory and hosts specific parameters, 
- Cluster for docker images, global paremeters and behaviour tuning 

```yaml
apiVersion: kubespray.k8s.io/v1
kind: ClusterNode
  name: master-1
  namespace: my-demo-cluster # One cluster per namespace to ease events handling
spec:
  type: master
  inventory: #ansible inventory style 
  certificates: 
    renew: 1y #supercharge vars defined 

status:
  node:
    health: HEALTH_OK
  state: Waiting # i.e. waiting operator to do something with it
```

Cluster should set docker images and global paremeters.  

```yaml
apiVersion: kubespray.k8s.io/v1
kind: Cluster
  name: kubespray-cluster
  namespace: my-demo-cluster # One cluster per namespace to ease event handling
spec:
  image: kubespray-operator:v0.0.1 # We should be allowed to supercharge operator image 

  runner:
    image: k8s.io/kubespray-operator-runner:v2.14.0 # we should embed able to override kubespray runner image for cherry-picks/customs etc
    maxForks: 30 # so of ansible cfg (most usefulls) may be supercharged from image 
    hostNetwok: true # should be usefull for ssh 
    placement: # nodeAffinity + tolerations for controlling which nodes can run runners 
    resources: # request & limits 
    volumeClaimTemplate: # default empty but should be overridable (could mount some hosts paths for ssh for example)

  nodes: 
    all:
      certificates: 
        renew: 60d 
      upgrade: 
        minor: 
          auto: true
        major: 
          auto: false
      
      deploy: 
        extra_vars: #json format of extra_vars that shoudl be provided 

  operations:
    creation: 
      # Need some conditions for starting operations => else we need to inform we're good to go
      # This kinda specific to kubespray : if one create a kind: ClusterNode of type: etcd 
      # then we still dont know if we're ready to create cluster because we may should wait two more etcds for starting etcd deployements
      # If we had ability to scale etcds and master easily, we could begin with what we have and scale but still this seems suboptimal (?)
      startConditions: 
        etcd: 
          minimum: 3
        lastNodeCreation: 0m 
      maxRetries: # per node max retries before setting status to Failed ? 

... 

status:
  cluster:
    health: HEALTH_OK
    lastChanged: "2020-03-17T13:01:14Z"
    lastChecked: "2020-03-20T22:15:01Z"
    previousHealth: HEALTH_ERR
  state: Created
```

# Some questions 

One object vs cluster/node ? 

How could someone add its own additional playbooks => add some hooks or let user code its own additional operator or replace playbooks/roles in image ? 

# Workflows

## Deployment

and others TBD 

# Toolbox

kubespray-toobox for applying simple kubeadms TBD 

# Kubespray changes

TBD


