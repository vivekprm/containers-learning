# containers-learning
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


