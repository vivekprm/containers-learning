# containers-learning
If you want to know what containers are, you could read [You Could Have Invented Container Runtimes: An Explanatory Fantasy](https://medium.com/@gtrevorjay/you-could-have-invented-container-runtimes-an-explanatory-fantasy-764c5b389bd3)

# reasons to use containers
- **packaging**. Let’s imagine you want to run your application on a computer. Your application depends on some weird JSON library being installed. Installing stuff on computers so you can run your program on them really sucks. It’s easy to get wrong! It’s scary when you make changes! Even if you use Puppet or Chef or something to install the stuff on the computers, it sucks. Containers are nice, because you can install all the stuff your program needs to run in the container inside the container. packaging is a huge deal and it is probably the thing that is most interesting to me about containers right now

- **scheduling**. $$$. Computers are expensive. If you have custom magical computers that are all set up differently, it’s hard to move your programs from running on Computer 1 to running on Computer 2. If you use containers, you can make your computers all the same! Then you can more easily pack your programs onto computers in a more reasonable way. Systems like Kubernetes do this automagically but we are not going to talk about Kubernetes.

- **better developer environment**. If you can make your application run in a container, then maybe you can also develop it on your laptop in the same container? Maybe? I don’t really know much about this.
- **security**. You can use **seccomp-bpf** or something to restrict which system calls your program runs? [sandstorm](https://sandstorm.io/) does stuff like this. I don’t really know.

There are probably more reasons that I’ve forgotten right now.

Usually I run programs on my computers. With Docker, you have to run a magical “docker daemon” that handles all my containers. Why? I don’t know! I don’t understand what the Docker daemon is for. With **rkt, you just run a process**.

People have been telling me stories that sometimes if you do the Wrong Thing and the Docker daemon has a bug, then the daemon can get deadlocked and then you have to kill it and all your containers die. I assume they work pretty hard on fixing these bugs, but I don’t want to have to trust that the Docker daemon has no bugs that will affect me. All software has bugs!

If you treat your container more like a process, then you can just run it, supervise it with supervisord or upstart or whatever your favorite way to supervise a process is. I know what a process is! I understand processes, kind of. Also I already use supervisord so I believe in that.

So that kind of makes me want to run rkt, even though it is a Newer Thing.

# PID 1
My coworker told me a very surprising thing about containers. If you run just one process in a container, then it apparently gets PID 1? PID 1 is a pretty exciting process. Usually on your computer, init gets PID 1. In particular, if another container process gets orphaned, then it suddenly magically becomes a child of PID 1.

So this is kind of weird, right? You don’t want your process to get random zombie processes as children. Like that might be fine, but it violates a lot of normal Unix assumptions about what normally happens to normal processes.

I think “violates a lot of normal Unix assumptions about what normally happens to normal processes” is basically the whole story about containers.

Yelp made a solution to this called [dumb-init](https://engineeringblog.yelp.com/2016/01/dumb-init-an-init-for-docker.html). It’s kind of interesting.

# networking
It can be really complicated! There are these “network namespaces” that I don’t really understand, and you need to do port forwarding, and why? Do you really need to use this complicated container networking stuff? I kind of just want to run container processes and run them on normal ports in my normal network namespace and leave it at that. Is that wrong?

# secrets
Often applications need to have passwords and things! Like to databases! How do you get the passwords into your container? This is a pretty big problem that I’m not going to go into now but I wanted to ask the question. (I know there are things like vault by hashicorp. Is using that a good idea? I don’t know!)

creating container images
To run containers, you need to create container images! There seem to be two main formats: Docker’s format and rkt’s format. I know nothing about either of them.

One nice thing is that rkt can run Docker containers images. So you could use Docker’s tools to make a container but then run it without using Docker.

There are at least 2 ways to make a Docker image: using a Dockerfile and [packer](https://www.packer.io/). Packer lets you provision a container with Puppet or Chef! That’s kind of useful.

# so many questions
From the few (maybe 5?) people I’ve talked to about containers so far, the overall consensus seems to be that they’re a pretty useful thing, despite all the hype, but that there are a lot of sharp edges and unexpected things that you’ll have to make your way through.

Good thing we can all learn on the internet together.

Docker’s current architecture. (tl;dr: @jpetazzo wrote a really [awesome gist](https://gist.github.com/jpetazzo/f1beba1dfd4c38e8daf2ebf2dcf3cdeb))

# Docker is too complicated! I just want to run a container
![docker-rkt](https://github.com/vivekprm/containers-learning/assets/2403660/a52c00bd-261f-4dc1-a3c7-594b8c5df094)

https://medium.com/@adriaandejonge/moving-from-docker-to-rkt-310dc9aec938#.mmmi6m9ql

# What is wrong with Docker?
If you are anything like me, you may still be so enthusiastic about Docker that you’re blindsided to some of its flaws. Don’t get me wrong, I am not saying Docker is bad. But it is not perfect either. So let’s take a look at some of these flaws.

Docker starts behaving like The Old Microsoft
Once companies realize they have a monopoly, they start behaving like monopolists. Just like Microsoft once practically killed Netscape by including Internet Explorer with Windows, now Docker is trying to defeat Kubernetes, Mesos/Marathon and Nomad by including Swarm into the Docker Core.

Docker deserves credit for simplifying Swarm and making it easily accessible to the average Docker user. Just as much as Microsoft deserves credit for the improvements they introduced in Internet Explorer 6.0 (True story!). The biggest issue in MSIE 6 is that it made Microsoft so dominant that they stopped innovating for five years.

It is good Docker created an easy to use scheduler for Docker but in my opinion, they should have kept it separate from the Docker core.

# "Docker’s architecture is fundamentally flawed"
At the heart of Docker is a daemon process that is the starting point of everything Docker does. The docker executable is merely a REST client that requests the Docker daemon to do its work. Critics of Docker say this is not very Linux-like.

Where it starts hurting is if you use an init system like systemd. Since systemd was not designed for Docker specifically, when you try to start a Docker process, you actually start a Docker client process that in turn requests the Docker daemon to start the actual Docker container. There is a risk that the Docker client fails while the actual Docker container keeps running. In such a situation, systemd concludes that the process has stopped and restarts the Docker client — in turn possibly creating a second container (yes, you can work around this but that is besides the point).

Now, I learned that perhaps Docker is the issue rather than systemd. As mentioned, Docker is not very Linux-like and because of this generic Linux tools do not play nice with Docker. Who is at fault, relative newcomer Docker or the Linux tools that have been around for years? Some say that “Docker’s architecture is fundamentally flawed” — this is a statement of Alex Polvi, CEO of CoreOS.

# The Docker build process is stuck in second gear
One of the nice features of Docker was the introduction of Dockerfiles that you can maintain in version control to reproduce Docker images. The only issue here is that the Dockerfile syntax has been frozen in Docker’s roadmap for a long time now. This means that the Dockerfile format has not evolved with the insights in the Docker community for at least a year and a half.

Of course, at some point we need a stable format. But only when the format has fully matured. Just to give you an example where Dockerfiles are lacking, consider Kelsey Hightower’s statement in his blog [12 Fractured Apps](https://medium.com/@kelseyhightower/12-fractured-apps-1080c73d481c#.smga9216i):

“Remember, ship artifacts not build environments.”

Fact is that the Dockerfile format does not support this separation very well. Consider for example building a Golang executable: the build environment is several hundreds of megabytes while the resulting executable may be as small as a couple of megabytes only. Sure, you can work around this by building the image on your local system. But you cannot elegantly build a small Golang image from a single Dockerfile using an Automated Build in Docker Hub. [Issues proposing to extend the Dockerfile format](https://github.com/docker/docker/issues/7115) to better support this principle have been on hold for years because of the freeze.

Update (11 May 2017): The above is FIXED with Multi-Stage Builds. See my new blog “Simplify the Smallest Possible Docker Image”

# How does rkt improve the situation?
The short answer is that rkt now provides a viable alternative to Docker. It has a more Linux-like architecture. And a strong competitor will keep the monopolist sharp.
The long answer:

Late 2014, CoreOS announced Rocket — later abbreviated and renamed as rkt — as a competing container platform. While rkt got a lot of attention in the first weeks after the announcement, it became silent for some time. CoreOS continued developing rkt into a viable alternative to Docker and only came back in the spotlight with the release of rkt 1.0 in February 2016.

Now that Docker announced its inclusion of Swarm into Docker Engine 1.12, it is time to start looking seriously at rkt as an alternative to Docker. Can it replace Docker now or in future? Is it difficult to switch between Docker and rkt?

Let’s take a look at rkt and find out…

## rkt can run Docker images
Consider you want to replace your staging and production systems with rkt while keeping all your development systems as is… In this case you can replace Docker with rkt on your runtime systems only. It is not precisely a drop-in replacement. After all, the architecture is different — but we’ll get to that. On the other hand, it is not too difficult to learn the new command line instructions for running a Docker container on rkt.

Open a CoreOS instance on your favorite cloud provider and type:
```sh
rkt run --insecure-options=image --port=80-tcp:80 docker://nginx
```
or replace nginx with your favorite Docker image. Under the hood, rkt converts the Docker image to Application Container (appc) format.

This means that you don’t need Docker to run Docker images! And it means that you can reuse anything you created with Docker without the least bit of migration pain — other than learning rkt’s command line syntax.

Let’s take a look at the individual parts of this command:

--insecure-options=image
If you leave this part out, rkt will refuse to start your image as it cannot find a signature (.asc file) to check the integrity of the Docker image. rkt is built secure by default. In this case, it means that you can save yourself some typing on the command line by providing the proper signatures.

--port=80-tcp:80
Just like in Docker, you need to explicitly expose a port to the outside world. Unlike Docker, rkt ports are named rather than numbered. If this were a native rkt image (or appc image to be precise) it would have probably read:

--port=http:80
However, since Docker does not have a concept of named ports, the exposed ports are automatically named as <number>-<protocol>. This also means that if you can only expose ports that have been explicitly specified using the EXPOSE keyword in the Dockerfile.

docker://nginx
The image is expected in the default Docker repository (Docker Hub). Instead of this example, there can be longer URLs pointing to either alternative Docker repositories (docker://…) or to generic websites containing appc images (https://…) but then you are not running Docker images anymore.

# rkt has a simpler architecture
In rkt, there is no daemon process. The rkt command line tool does all the work. And I thought “that rkt diagram looks way easier to operate in production! That’s what I want!”

Okay, sure! No problem. I can use runC! Go to runc.io, follow the direction, make a config.json file, extract my container into a tarball, and now I can run my container with a single command. Awesome.

# Actually I want to run 50 containers on the same machine.
Oh, okay, that’s pretty different. So – let’s say all my 50 containers share a bunch of files (shared libraries like libc, Ruby gems, a base operating system, etc.). It would be nice if I could load all those files into memory just once, instead of 3 times.

If I did this I could save disk space on my machine (by just storing the files once), but more importantly, I could save memory!

If I’m running 50 containers I don’t want to have 50 copies of all my shared libraries in memory. That’s why we invented dynamic linking!

If you’re running just 2-3 containers, maybe you don’t care about a little bit of copying. That’s for you to decide!

It turns out that the way Docker solves this is with **“overlay filesystems”** or **“graphdrivers”**. (why are they called graphdrivers? Maybe because different layers depend on each other like in a directed graph?) These let you stack filesystems – you start with a base filesystem (like Ubuntu 14.04) and then you can start adding more files on top of it one step at a time.

Filesystem overlays need some Linux kernel support to work – you need to use a filesystem that supports them. [The Brutally Honest Guide to Docker Graphdrivers](https://blog.jessfraz.com/post/the-brutally-honest-guide-to-docker-graphdrivers/) by the fantastic Jessie Frazelle has a quick overview. overlayfs seems to be the most normal option.
At this point, I was running Ubuntu 14.04. 14.04 runs a 3.13 Linux kernel! But to use overlayfs, you need a 3.18 kernel! So you need to upgrade your kernel. That’s fine.

Back to runC. runC [does not support overlay filesystems](https://github.com/opencontainers/runc/issues/1040). This is an intentional design choice – it lets runC run on older kernels, and lets you separate out the concerns. But it’s not super obvious right now how to use runC with overlay filesystems. So what do I do?

# I’m going to use rkt to get overlay filesystem support
So! I’ve decided I want overlay filesystem support, and gotten a Linux kernel newer than 3.18. Awesome. Let’s try rkt, like in that diagram! It lives at coreos.com/rkt/

If you download rkt and run ```rkt run docker://my-docker-registry/container```, This totally works. Two small things I learned:

```--net=host``` will let you run in the host network namespace

Network namespaces are one of the most important things in container land. But if you want to run containers using as few new things as possible, you can start out by just running your containers as normal programs that run on normal ports, like any other program on your computer. Cool

```--exec=/my/cool/program``` lets you set which command you want rkt to execute inside the image

**systemd**: rkt will run a program called ```systemd-nspawn``` as the init (PID 1) process inside your container. This is because it can be bad to run an arbitrary process as PID 1 – your process isn’t expecting it and will might react badly. It also run some systemd-journal process? I don’t know what that’s for yet.

The systemd journal process might act as a syslog for your container, so that programs sending logs through syslog end up actually sending them somewhere.

There is quite a lot more to know about rkt but I don’t know most of it yet.

# I’d like to trust that the code I’m running is actually my code
So, security is important. Let’s say I have a container registry. I’d like to make sure that the code I’m running from that registry is actually trusted code that I built.

Docker lets you sign images to verify where they came from. rkt lets you run Docker images. **rkt does not let you check signatures from Docker images though!** This is bad.

You can fix this by setting up your own rkt registry. Or maybe other things! I’m going to leave that here. At this point you probably have to stop using Docker containers though and convert them to a different format.

# Supervising my containers (and let’s talk about Docker again)
So, I have this Cool Container System, and I can run containers with overlayfs and I can trust the code I’m running. What now?

Let’s go back to Docker for a bit. So far I’ve been a bit dismissive about Docker, and I’d like to look at its current direction a little more seriously. Jérôme Petazzoni wrote an extremely informative and helpful discussion about how Docker got to its architecture today in [this gist](https://gist.github.com/jpetazzo/f1beba1dfd4c38e8daf2ebf2dcf3cdeb). He says (which I think is super true) that Docker’s approach to date has done a huge amount to drive container adoption and let us try out different approaches today.

The end of that gist is a really good starting point for talking about how “start new containers” should work.

Jérôme very correctly says that if you’re going to run containers, you need a way to tell boxes which containers to run, and supervise and restart containers when they die. You could supervise them with daemontools, supervisord, upstart, or systemd, or something else!

“Tell boxes which containers to run” is another nontrivial problem and I’m not going to talk about it at all here. So, back to supervision.

Let’s say you use systemd. Then that’ll look like (from the diagram I posted at the top):
```
- systemd -+- rkt -+- process of container X
           |       \- other process of container X
           +- rkt --- process of container Y
           \- rkt --- process of container Z
```

My understanding of the problem with Docker in production historically is that – the process that is responsible for this core functionality of process supervision was the Docker engine, but it also had a lot of other features that you don’t necessarily want running in production.

The way Docker seems to be going in the future is something like: (this diagram is from jpetazzo’s gist above)

```
- init - containerd -+- shim for container X -+- process of container X
         |                        \- other process of container X
                     +- shim for container Y --- process of container Y
                     \- shim for container Z --- process of container Z
```

where [containerd](https://containerd.tools/) is a separate tool, and the Docker engine talks to containerd but isn’t as heavily coupled to it. Right now containerd’s website says it’s alpha software, but they also say on their website that it’s used in current versions of Docker, so it’s not totally obvious what the state is right now.

# the OCI standard
We talked about how runC can run containers just fine, but cannot do overlay filesystems or fetch + validate containers from a registry. I would be remiss if I didn’t mention the OCID project that @grepory told me about last week, which aims to do those as separate components instead of in an integrated system like Docker.

Here’s the article: [Red Hat, Google Engineers Work on a Way for Kubernetes to Run Containers Without Docker](http://thenewstack.io/oci-building-way-kubernetes-run-containers-without-docker/) .

Today there’s skopeo which lets you fetch and validate images from Docker registries

# what we learned
here’s the tl;dr:
- you can run Docker containers without Docker
- runC can run containers… but it doesn’t have overlayfs
- but overlay filesystems are important!
- rkt has overlay filesystem support.
- you need to start & supervise the containers! You can use any regular process supervisor to do that.
- also you need to tell your computers which containers to run
- software around the OCI standard is evolving but it’s not there yet

As far as I can tell running containers without using Docker or Kubernetes or anything is totally possible today, but no matter what tools you use it’s definitely not as simple as “just run a container”. Either way going through all these steps helps me understand what the actual components of running a container are and what all these different pieces of software are trying to do.

This landscape is pretty confusing but I think it’s not impossible to understand! There are only a finite number of different pieces of software to figure out the role of :)

If you want to see more about running containers from scratch, see [Cgroups, namespaces, and beyond: what are containers made from?](https://www.youtube.com/watch?v=sK5i-N34im8&feature=youtu.be) by jpetazzo. There’s a live demo of how to run a container with 0 tools (no docker, no rkt, no runC) [at this point in the video](https://www.youtube.com/watch?v=sK5i-N34im8&feature=youtu.be&t=41m11s) which is super super interesting.


The major organizations writing open source software to help people run containers on Linux seem to be (alphabetically): Canonical, CoreOS, Docker, Google, HashiCorp, Mesosphere, Red Hat, and OCI (cross-company foundation).

I’ve tried to summarize each one in 3 words or less which is hard because a lot of this software has a lot of different jobs.
- docker stuff
  - [docker](https://www.docker.com/)
  - [containerd (process supervisor)](https://www.containerd.tools/)
  - [docker swarm](https://docs.docker.com/swarm/) (orchestration)
- Kubernetes stuff
  - [kubernetes](http://kubernetes.io/) (orchestration, has many components)
- Mesosphere stuff
  - [Mesos](http://mesos.apache.org/) (orchestration)
- CoreOS stuff
  - [CoreOS](https://coreos.com/why/) (linux distribution)
  - [rkt](https://coreos.com/rkt) (runs containers)
  - [flannel](https://coreos.com/flannel/docs/latest/) (network overlay)
  - [etcd](https://coreos.com/etcd/) (key-value store)
- HashiCorp stuff
  - [consul](https://www.consul.io/) (key-value store, service discovery)
  - [packer](https://www.packer.io/intro/) (creates containers)
  - [vault](https://www.vaultproject.io/) (secrets management)
  - [nomad](https://www.nomadproject.io/) (orchestration)
- OCI (open container initiative) stuff
  - [runC](http://runc.io/) (runs containers)
  - [libcontainer](https://github.com/opencontainers/runc/tree/master/libcontainer) (donated by Docker, powers runC)
- systemd-nspawn ([man page](https://www.freedesktop.org/software/systemd/man/systemd-nspawn.html)) (starts containers)
- [dumb-init](https://github.com/Yelp/dumb-init) (init process)
- [LXC](https://linuxcontainers.org/) (runs containers, from Canonical)

There are also a bunch of container registries you can pay for, like [quay (from CoreOS)](https://quay.io/), [google’s one](https://cloud.google.com/container-registry/), [docker trusted registry](https://docs.docker.com/docker-trusted-registry/), etc.

# Docker Networking
## what even is container networking?
When you run a program in a container, you have two main options:

- run the program in the host network namespace. This is normal networking – if you run a program on port 8282, it will run on port 8282 on the computer. No surprises.
- run the program in its own network namespace
  - If you have a program running in its own network namespace (let’s say on port 9382), other programs on other computers need to be able to make network connections to that program.

At first I thought “how complicated can that be? connecting programs together is simple, right?” Like, there’s probably only one way to do it? It turns out that this problem of how to connect two programs in containers together has a ton of different solutions. Let’s learn what those solutions are!

## “every container gets an IP”
If you are a container nerd these days, you have probably heard of Kubernetes. Kubernetes is a system that will take a container and automatically decide which computer your container should run on. (among other things)

One of Kubernetes’ core requirements (for you to even start using it) is that every container has to have an IP address, and that any other program inside you cluster can talk to your container just using that IP address. So this might mean that on one computer you might have containers with hundreds or thousands of IP addresses (instead of just one IP address and many ports).

When I first heard of this “every container gets an IP” concept I was really confused and kind of concerned. How would this even work?! My computer only has one IP address! This sounds like weird confusing magic! Luckily it turns out that, as with most computer things, this is actually totally possible to understand.

This “every container gets an IP” problem is what I’m going to explain in this blog post. There are other ways to network containers, but it’s going to take long enough already to just explain this one :)

I’m also going to restrict myself to mostly talking about how to make this work on AWS. If you have your own physical datacenter there are more options.

## Our goal
You have a computer (AWS instance). That computer has an IP address (like 172.9.9.9).

You want your container to also have an IP address (like 10.4.4.4).

We’re going to learn how to get a packet sent to 10.4.4.4 on the computer 172.9.9.9.

On AWS this can actually be super easy – there are these things called “VPC Route Tables”, and you can just say “send packets for 10.4.4.* to 172.9.9.9 please” and AWS will make it work for you. The catch is you can only have 50 of these rules, so if you want to have a cluster of more than 50 instances, you need to go back to being confused about networking.

## some networking basics: IP addresses, MAC addresses, local networks
In order to understand how you can have hundreds of IP addresses on one single machine, we need to understand a few basic things about networking.

I’m going to take for granted that you know:

- In computer networking, programs send packets to each other
- Every packet (for the most part) has an IP address on it
- On Linux, the kernel is responsible for implementing most networking protocols
- **a little bit about subnets**: the subnet 10.4.4.0/24 means “every IP from 10.4.4.0 to 10.4.4.255”. I’ll sometimes write 10.4.4.* to mean this.

I’ll do my best to explain the rest.

### Thing 0: parts of a network packet
A network packet has a bunch of different parts (often called “layers”). There are a lot of different kinds of network packets, but let’s just talk about a normal HTTP request (like GET /). The parts are:
- the MAC address this packet should go to (“Layer 2”)
- the source IP and destination IP (“Layer 3”)
- the port and other TCP/UDP information (“Layer 4”)
- the contents of your HTTP packet like GET / (“Layer 7”)

### Thing 1: local networking vs far-away networking
When you send a packet directly to a computer (on the same local network), here’s how it works.

Packets are addressed by MAC address. My MAC address is 3c:97:ae:44:b3:7f; I found it by running ifconfig.
```sh
-> ifconfig
wlxf0a73129cd4a   ether f0:a7:31:29:cd:4a 
```
So to send a packet to me, any computer on my local network can write ether f0:a7:31:29:cd:4a on it, and it gets to my computer. **In AWS, “local network” basically means “availability zone”**. If two instances are in the same AWS availability zone, they can just put the MAC address of the target computer on it, and then the packet will get to the right place. It doesn’t matter what IP address is on the packet!

Okay, what if my computer isn’t in the same local network / availability zone as the target computer? What then? **Then routers in the middle need to look at the IP address on the packet** and get it to the right place.

There is a lot to know about how routers work, and we do not have time to learn it all right now. Luckily, in AWS you have basically no way to configure the routers, so it doesn’t matter if we don’t know how they work! **To send a packet to an instance outside your availability zone, you need to put that instance’s IP address on it**. Full stop. Otherwise it ain’t gonna get there.

If you manage your own datacenter, you can do clever stuff to set up your routers.

So! Here’s what we’ve learned, for AWS:
- if you’re in the same AZ as your target, you can just send a packet with any random IP address on it, and as long as the MAC address is right it’ll get there.
- if you are in a different AZ, to send a packet to a computer, it has to have the IP address of that instance on it.

# The route table
You may be wondering “But how can I control the MAC address my packet gets sent to! I have never done that ever! That is very confusing!”

When you send a packet to 172.23.2.1 on your local network, your operating system (Linux, for our purposes) **looks up the MAC address for that IP address in a table it maintains (called the ARP table)**. Then it puts that MAC address on the packet and sends it off.

So! **What if I had a packet for the container 10.4.4.4 but I actually wanted it to go to the computer 172.23.1.1?** It turns out this actually easy peasy! You just add an entry to another table. It’s all tables.

Here’s command you could run to do this manually:
```sh
sudo ip route add 10.4.4.0/24 via 172.23.1.1 dev eth0
```

ip route add adds an entry to the route table on your computer. This route table entry says “Linux, whenever you see a packet for 10.4.4.*, just send it to the MAC address for 172.23.2.1, would ya darling?”

# we can give containers IPs!
It is time celebrate our first victory! We now know all the basic tools for one main way to route container IP addresses!

The steps are:
- pick a different subnet for every computer on your network (like 10.4.4.0/24 – that’s 10.4.4.*). That subnet will let you have 256 containers per machine.
- On every computer, add routes for every other computer. So you’d add a route for 10.4.1.0/24, 10.4.2.0/24, 10.4.3.0/24, etc.
- You’re done! Packets to 10.4.4.4 will now get routed to the right computer. There’s still the question of what they will do when they get to that computer, but we’ll get there in a bit.

So our first tool for doing container networking is the **route table**.

# what if the two computers are in different availability zones?
We said earlier that this route table trick will only work if the computers are connected directly. If the two computers are far apart (in different local networks) we’ll need to do something more complicated.

We want to send a packet to the container IP 10.4.4.4, and it is on the computer 172.9.9.9. But because the computer is far away, we have to address the packet to the IP address 172.9.9.9. Woe is us! All is lost! Where are we going to put the IP address 10.4.4.4?

# Encapsulation
All is not lost. We can do a thing called “encapsulation”. **This is where you take a network packet and put it inside ANOTHER network packet.**

So instead of sending
```
IP: 10.4.4.4
TCP stuff
HTTP stuff
```

we will send
```
IP: 172.9.9.9
(extra wrapper stuff)
IP: 10.4.4.4
TCP stuff
HTTP stuff
```

There are at least 2 different ways of doing encapsulation: there’s **“ip-in-ip”** and **“vxlan”** encapsulation.

**vxlan** encapsulation takes your whole packet (including the MAC address) and wraps it inside a UDP packet. That looks like this:
```
MAC address: 11:11:11:11:11:11
IP: 172.9.9.9
UDP port 8472 (the "vxlan port")
MAC address: ab:cd:ef:12:34:56
IP: 10.4.4.4
TCP port 80
HTTP stuff
```

**ip-in-ip** encapsulation just slaps on an extra IP header on top of your old IP header. This means you don’t get to keep the MAC address you wanted to send it to but I’m not sure why you would care about that anyway.
```
MAC:  11:11:11:11:11:11
IP: 172.9.9.9
IP: 10.4.4.4
TCP stuff
HTTP stuff
```

# How to set up encapsulation
Like before, you might be thinking “how can I get my kernel to do this weird encapsulation thing to my packets”? This turns out to be not all that hard. **Basically all you do is set up a new network interface with encapsulation configured**.

On my laptop, I can do this using: (taken from [these instructions](http://www.linux-admins.net/2010/09/tunneling-ipip-and-gre-encapsulation.html))
```sh
sudo ip tunnel add mytun mode ipip remote 172.9.9.9 local 10.4.4.4 ttl 255
sudo ifconfig mytun 10.42.1.1
```

Then you set up a route table, but you tell Linux to route the packet with your new magical encapsulation network interface. Here’s what that looks like:
```sh
sudo route add -net 10.42.2.0/24 dev mytun
sudo route list
```

I’m mostly giving you these commands to get an idea of the kinds of commands you can use to create / inspect these tunnels (```ip route list``` , ```ip tunnel```, ```ifconfig```) – I’ve almost certainly gotten a couple of the specifics wrong, but this is about how it works.

# How do routes get distributed?
We’ve talked a lot about adding routes to your route table (“10.4.4.4 should go via 172.9.9.9”), but I haven’t explained at all how those routes should actually get in your route table. Ideally you’d like them to configured automatically.

Every container networking thing to runs some kind of daemon program on every box which is in charge of adding routes to the route table.

There are two main ways they do it:
- the routes are in an etcd cluster, and the program talks to the etcd cluster to figure out which routes to set
- use the **BGP** protocol to gossip to each other about routes, and a **daemon (BIRD)** listens for BGP messages on every box

# What happens when packets get to your box?
So, you’re running Docker, and a packet comes in on the IP address 10.4.4.4. How does that packet actually end up getting to your program?

I’m going to try to explain **bridge networking** here. I’m a bit fuzzy on this so some of this is probably wrong.

## My understanding right now is:
- every packet on your computer goes out through a real interface (like eth0)
- Docker will create **fake** (virtual) network interfaces for every single one of your containers. These have IP addresses like 10.4.4.4
- Those virtual network interfaces are **bridged** to your real network interface. This means that the packets get copied (?) to the network interface corresponding to the real network card, and then sent out to the internet
This seems important but I don’t totally get it yet.

# finale: how all these container networking things work
Okay! Now we we have all the fundamental concepts you need to know to navigate this container networking landscape.

## Flannel
Flannel supports a few different ways of doing networking:
- **vxlan** (encapsulate all packets)
- **host-gw** (just set route table entries, no encapsulation)

The daemon that sets the routes gets them from an **etcd cluster**.

## Calico
Calico supports 2 different ways of doing networking:

- **ip-in-ip** encapsulation
- “regular” mode, (just set route table entries, no encapsulation)

The daemon that sets the routes gets them using BGP messages from other hosts. **There’s still an etcd cluster with Calico but it’s not used for distributing routes**.

The most exciting thing about Calico is that it has the option to not use encapsulation. If you look carefully though you’ll notice that **Flannel also has an option to not use encapsulation!** If you’re on AWS, I can’t actually tell which of these is better. They have the same limitations: they’ll both only work between instances in the same availability zone.

Most of these container networking things will set up all these routes and tunnels and stuff for you, but I think it’s important to understand what’s going on behind the scenes, so that if something goes wrong I can debug it and fix it.

# is this software defined networking?
I don’t know what software defined networking. All of this helps you do networking differently, and it’s all software, so maybe it’s software defined networking?

# that’s all
That’s all I have for now! Hopefully this was helpful. It turns out this stuff isn’t so bad, and spending some time with the ```ip``` command, ```ifconfig``` and ```tcpdump``` can help you understand the basics of what’s going on in your Kubernetes installation. You don’t need to be an expert network engineer!

