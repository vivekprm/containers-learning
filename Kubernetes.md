# Kubernetes lets you run code in production without setting up new servers
The first pitch I got for Kubernetes was the following conversation with my partner Kamal:

Here’s an approximate transcript:
Kamal: With Kubernetes you can set up a new service with a single command
Julia: I don’t understand how that’s possible.
Kamal: Like, you just write 1 configuration file, apply it, and then you have a HTTP service running in production
Julia: But today I need to create new AWS instances, write a puppet manifest, set up service discovery, configure my load balancers, configure our deployment software, and make sure DNS is working, it takes at least 4 hours if nothing goes wrong.
Kamal: Yeah. With Kubernetes you don’t have to do any of that, you can set up a new HTTP service in 5 minutes and it’ll just automatically run. As long as you have spare capacity in your cluster it just works!
Julia: There must be a trap

There kind of is a trap, setting up a production Kubernetes cluster is (in my experience) is definitely not easy. (see [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) for what’s involved to get started). But we’re not going to go into that right now!

So the first cool thing about Kubernetes is that it has the potential to make life way easier for developers who want to deploy new software into production. That’s cool, and it’s actually true, once you have a working Kubernetes cluster you really can set up a production HTTP service (“run 5 of this application, set up a load balancer, give it this DNS name, done”) with just one configuration file. It’s really fun to see.

# Kubernetes gives you easy visibility & control of what code you have running in production
IMO you can’t understand Kubernetes without understanding etcd. So let’s talk about etcd!

Imagine that I asked you today “hey, tell me every application you have running in production, what host it’s running on, whether it’s healthy or not, and whether or not it has a DNS name attached to it”. I don’t know about you but I would need to go look in a bunch of different places to answer this question and it would take me quite a while to figure out. I definitely can’t query just one API.

In Kubernetes, all the state in your cluster – applications running (“pods”), nodes, DNS names, cron jobs, and more – is stored in a single database (etcd). Every Kubernetes component is stateless, and basically works by 
- Reading state from etcd (eg “the list of pods assigned to node 1”)
- Making changes (eg “actually start running pod A on node 1”)
- Updating the state in etcd (eg “set the state of pod A to ‘running’”)

This means that if you want to answer a question like “hey, how many nginx pods do I have running right now in that availabliity zone?” you can answer it by querying a single unified API (the Kubernetes API!). And you have exactly the same access to that API that every other Kubernetes component does.

This also means that you have easy control of everything running in Kubernetes. If you want to, say,
- Implement a complicated custom rollout strategy for deployments (deploy 1 thing, wait 2 minutes, deploy 5 more, wait 3.7 minutes, etc)
- Automatically [start a new webserver](https://github.com/kamalmarhubi/kubereview) every time a branch is pushed to github
- Monitor all your running applications to make sure all of them have a reasonable cgroups memory limit

all you need to do is to write a program that talks to the Kubernetes API. (a “controller”)

Another very exciting thing about the Kubernetes API is that you’re not limited to just functionality that Kubernetes provides! If you decide that you have your own opinions about how your software should be deployed / created / monitored, then you can write code that uses the Kubernetes API to do it! It lets you do everything you need.

# If every Kubernetes component dies, your code will still keep running
One thing I was originally promised (by various blog posts :)) about Kubernetes was “hey, if the Kubernetes apiserver and everything else dies, it’s ok, your code will just keep running”. I thought this sounded cool in theory but I wasn’t sure if it was actually true.

So far it seems to be actually true!

I’ve been through some etcd outages now, and what happens is
- All the code that was running keeps running
- Nothing new happens (you can’t deploy new code or make changes, cron jobs will stop working)
- When everything comes back, the cluster will catch up on whatever it missed

This does mean that if etcd goes down and one of your applications crashes or something, it can’t come back up until etcd returns.

# Kubernetes’ design is pretty resilient to bugs
Like any piece of software, Kubernetes has bugs. For example right now in our cluster the controller manager has a memory leak, and the scheduler crashes pretty regularly. Bugs obviously aren’t good but so far I’ve found that Kubernetes’ design helps mitigate a lot of the bugs in its core components really well.

If you restart any component, what happens is:

- It reads all its relevant state from etcd
- It starts doing the necessary things it’s supposed to be doing based on that state (scheduling pods, garbage collecting completed pods, scheduling cronjobs, deploying daemonsets, whatever)
Because all the components don’t keep any state in memory, you can just restart them at any time and that can help mitigate a variety of bugs.

For example! Let’s say you have a memory leak in your controller manager. Because the controller manager is stateless, you can just periodically restart it every hour or something and feel confident that you won’t cause any consistency issues. Or we ran into a bug in the scheduler where it would sometimes just forget about pods and never schedule them. You can sort of mitigate this just by restarting the scheduler every 10 minutes. (we didn’t do that, we fixed the bug instead, but you could :) )

So I feel like I can trust Kubernetes’ design to help make sure the state in the cluster is consistent even when there are bugs in its core components. And in general I think the software is generally improving over time. The only stateful thing you have to operate is etcd

Not to harp on this “state” thing too much but – I think it’s cool that in Kubernetes the only thing you have to come up with backup/restore plans for is etcd (unless you use persistent volumes for your pods). I think it makes kubernetes operations a lot easier to think about.

# Implementing new distributed systems on top of Kubernetes is relatively easy
Suppose you want to implement a distributed cron job scheduling system! Doing that from scratch is a ton of work. But implementing a distributed cron job scheduling system inside Kubernetes is much easier! (still not trivial, it’s still a distributed system)

The first time I read the code for the Kubernetes cronjob controller I was really delighted by how simple it was. Here, go read it! The main logic is like 400 lines of Go. Go ahead, read it! => [cronjob_controller.go](https://github.com/kubernetes/kubernetes/blob/e4551d50e57c089aab6f67333412d3ca64bc09ae/pkg/controller/cronjob/cronjob_controller.go) <=

Basically what the cronjob controller does is:

- Every 10 seconds:
  - Lists all the cronjobs that exist
  - Checks if any of them need to run right now
  - If so, creates a new Job object to be scheduled & actually run by other Kubernetes controllers
  - Clean up finished jobs
  - Repeat

The Kubernetes model is pretty constrained (it has this pattern of resources are defined in etcd, controllers read those resources and update etcd), and I think having this relatively opinionated/constrained model makes it easier to develop your own distributed systems inside the Kubernetes framework.

Kamal introduced me to this idea of “Kubernetes is a good platform for writing your own distributed systems” instead of just “Kubernetes is a distributed system you can use” and I think it’s really interesting. He has a prototype of a [system to run an HTTP service for every branch you push to github](https://github.com/kamalmarhubi/kubereview). It took him a weekend and is like 800 lines of Go, which I thought was impressive!

# Kubernetes lets you do some amazing things (but isn’t easy)
I started out by saying “kubernetes lets you do these magical things, you can just spin up so much infrastructure with a single configuration file, it’s amazing”. And that’s true!

What I mean by “Kubernetes isn’t easy” is that Kubernetes has a lot of moving parts learning how to successfully operate a highly available Kubernetes cluster is a lot of work. Like I find that with a lot of the abstractions it gives me, I need to understand what is underneath those abstractions in order to debug issues and configure things properly. I love learning new things so this doesn’t make me angry or anything, I just think it’s important to know :)

One specific example of “I can’t just rely on the abstractions” that I’ve struggled with is that I needed to learn a LOT [about how networking works on Linux](https://jvns.ca/blog/2016/12/22/container-networking/) to feel confident with setting up Kubernetes networking, way more than I’d ever had to learn about networking before. This was very fun but pretty time consuming. I might write more about what is hard/interesting about setting up Kubernetes networking at some point.

Or I wrote a [2000 word blog post](https://jvns.ca/blog/2017/08/05/how-kubernetes-certificates-work/) about everything I had to learn about Kubernetes’ different options for certificate authorities to be able to set up my Kubernetes CAs successfully.

I think some of these managed Kubernetes systems like GKE (google’s kubernetes product) may be simpler since they make a lot of decisions for you but I haven’t tried any of them.

# Operating a Kubernetes network
I’ve been working on Kubernetes networking a lot recently. One thing I’ve noticed is, while there’s a reasonable amount written about how to set up your Kubernetes network, I haven’t seen much about how to operate your network and be confident that it won’t create a lot of production incidents for you down the line.

In this post I’m going to try to convince you of three things: (all I think pretty reasonable :))
- Avoiding networking outages in production is important
- Operating networking software is hard
- It’s worth thinking critically about major changes to your networking infrastructure and the impact that will have on your reliability, even if very fancy Googlers say “this is what we do at Google”. (google engineers are doing great work on Kubernetes!! But I think it’s important to still look at the architecture and make sure it makes sense for your organization.)
- 
I’m definitely not a Kubernetes networking expert by any means, but I have run into a few issues while setting things up and definitely know a LOT more about Kubernetes networking than I used to.

# Operating networking software is hard
Here I’m not talking about operating physical networks (I don’t know anything about that), but instead about keeping software like DNS servers & load balancers & proxies working correctly.

I have been working on a team that’s responsible for a lot of networking infrastructure for a year, and I have learned a few things about operating networking infrastructure! (though I still have a lot to learn obviously). 3 overall thoughts before we start:

- Networking software often relies very heavily on the Linux kernel. So in addition to configuring the software correctly you also need to make sure that a bunch of different **sysctls** are set correctly, and **a misconfigured sysctl can easily be the difference between “everything is 100% fine” and “everything is on fire”**.
- Networking requirements change over time (for example maybe you’re doing 5x more DNS lookups than you were last year! Maybe your DNS server suddenly started returning TCP DNS responses instead of UDP which is a totally different kernel workload!). This means software that was working fine before can suddenly start having issues.
- To fix a production networking issues you often need a lot of expertise. (for example see this great post by Sophie Haskins on [debugging a kube-dns issue](http://blog.sophaskins.net/blog/misadventures-with-kube-dns/)) I’m a lot better at debugging networking issues than I was, but that’s only after spending a huge amount of time investing in my knowledge of Linux networking.

I am still far from an expert at networking operations but I think it seems important to:
- Very rarely make major changes to the production networking infrastructure (because it’s super disruptive)
- When you are making major changes, think really carefully about what the failure modes are for the new network architecture are
- Have multiple people who are able to understand your networking setup

Switching to Kubernetes is obviously a pretty major networking change! So let’s talk about what some of the things that can go wrong are!

# Kubernetes networking components
The Kubernetes networking components we’re going to talk about in this post are:
- Your overlay network backend (like flannel/calico/weave net/romana)
- ```kube-dns```
- ```kube-proxy```
- Ingress controllers / load balancers
- The ```kubelet``
  
If you’re going to set up HTTP services you probably need all of these. I’m not using most of these components yet but I’m trying to understand them, so that’s what this post is about.

## The simplest way: Use host networking for all your containers
Let’s start with the simplest possible thing you can do. This won’t let you run HTTP services in Kubernetes. I think it’s pretty safe because there are less moving parts.

If you use host networking for all your containers I think all you need to do is:
- Configure the kubelet to configure DNS correctly inside your containers
- That’s it

If you use host networking for literally every pod you don’t need ```kube-dns``` or ```kube-proxy```. **You don’t even need a working overlay network**.

In this setup your pods can connect to the outside world (the same way any process on your hosts would talk to the outside world) but **the outside world can’t connect to your pods**.

This isn’t super important (I think most people want to run HTTP services inside Kubernetes and actually communicate with those services) but I do think it’s interesting to realize that at some level all of this networking complexity isn’t strictly required and sometimes you can get away without using it. **Avoiding networking complexity seems like a good idea to me if you can**.

## Operating an overlay network
The first networking component we’re going to talk about is your overlay network. Kubernetes assumes that every pod has an IP address and that you can communicate with services inside that pod by using that IP address. **When I say “overlay network” this is what I mean (“the system that lets you refer to a pod by its IP address”)**.

All other Kubernetes networking stuff relies on the overlay networking working correctly. You can read more about the kubernetes networking model [here](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model).

The way Kelsey Hightower describes in [kubernetes the hard way](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/11-pod-network-routes.md) seems pretty good but it’s not really viable on AWS for clusters more than 50 nodes or so, so I’m not going to talk about that.

There are a lot of overlay network backends (**calico, flannel, weaveworks, romana**) and the landscape is pretty confusing. But as far as I’m concerned an overlay network has 2 responsibilities:
- Make sure your pods can send network requests outside your cluster
- Keep a stable mapping of nodes to subnets and keep every node in your cluster updated with that mapping. Do the right thing when nodes are added & removed.

Okay! So! What can go wrong with your overlay network?
- The overlay network is responsible for setting up iptables rules (basically ```iptables -A -t nat POSTROUTING -s $SUBNET -j MASQUERADE```) to ensure that containers can make network requests outside Kubernetes. If something goes wrong with this rule then your containers can’t connect to the external network. This isn’t that hard (it’s just a few iptables rules) but it is important. I made a [pull request](https://github.com/coreos/flannel/pull/808) because I wanted to make sure this was resilient
- Something can go wrong with adding or deleting nodes. We’re using the flannel hostgw backend and at the time we started using it, node deletion [did not work](https://github.com/coreos/flannel/pull/803).
- Your overlay network is probably dependent on a distributed database (etcd). If that database has an incident, this can cause issues. For example https://github.com/coreos/flannel/issues/610 says that if you have data loss in your flannel etcd cluster it can result in containers losing network connectivity. (this has now been fixed)
- You upgrade Docker and everything breaks
- Probably more things!

I’m mostly talking about past issues in Flannel here but I promise I’m not picking on Flannel – I actually really like Flannel because I feel like it’s relatively simple (for instance the [vxlan backend part](https://github.com/coreos/flannel/tree/master/backend/vxlan) of it is like 500 lines of code) and I feel like it’s possible for me to reason through any issues with it. And it’s obviously continuously improving. They’ve been great about reviewing pull requests.

My approach to operating an overlay network so far has been:
- Learn how it works in detail and how to debug it (for example the hostgw network backend for Flannel works by creating routes, so you mostly just need to do ```sudo ip route list``` to see whether it’s doing the correct thing)
- Maintain an internal build so it’s easy to patch it if needed
- When there are issues, contribute patches upstream

I think it’s actually really useful to go through the list of merged PRs and see bugs that have been fixed in the past – it’s a bit time consuming but is a great way to get a concrete list of kinds of issues other people have run into.

It’s possible that for other people their overlay networks just work but that hasn’t been my experience and I’ve heard other folks report similar issues. If you have an overlay network setup that is a) on AWS and b) works on a cluster more than 50-100 nodes where you feel more confident about operating it I would like to know.

# Operating kube-proxy and kube-dns?
Now that we have some thoughts about operating overlay networks, let’s talk about

There’s a question mark next to this one because I haven’t done this. Here I have more questions than answers.

Here’s how Kubernetes services work! A service is a collection of pods, which each have their own IP address (like 10.1.0.3, 10.2.3.5, 10.3.5.6)
- Every Kubernetes service gets an IP address (like 10.23.1.2)
- ```kube-dns``` resolves Kubernetes service DNS names to IP addresses (so ```my-svc.my-namespace.svc.cluster.local``` might map to 10.23.1.2)
- ```kube-proxy``` sets up iptables rules in order to do random load balancing between them. Kube-proxy also has a userspace round-robin load balancer but my impression is that they don’t recommend using it.

So when you make a request to ```my-svc.my-namespace.svc.cluster.local```, it resolves to 10.23.1.2, and then iptables rules on your local host (generated by kube-proxy) redirect it to one of 10.1.0.3 or 10.2.3.5 or 10.3.5.6 at random.

Some things that I can imagine going wrong with this:

- ```kube-dns``` is misconfigured
- ```kube-proxy``` dies and your iptables rules don’t get updated
- Some issue related to maintaining a large number of iptables rules

Let’s talk about the iptables rules a bit, since doing load balancing by creating a bajillion iptables rules is something I had never heard of before!

kube-proxy creates one iptables rule per target host like this: (these rules are from [this github issue](https://github.com/kubernetes/kubernetes/issues/37932))
```
-A KUBE-SVC-LI77LBOOMGYET5US -m comment --comment "default/showreadiness:showreadiness" -m statistic --mode random --probability 0.20000000019 -j KUBE-SEP-E4QKA7SLJRFZZ2DD[b][c]  
-A KUBE-SVC-LI77LBOOMGYET5US -m comment --comment "default/showreadiness:showreadiness" -m statistic --mode random --probability 0.25000000000 -j KUBE-SEP-LZ7EGMG4DRXMY26H  
-A KUBE-SVC-LI77LBOOMGYET5US -m comment --comment "default/showreadiness:showreadiness" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-RKIFTWKKG3OHTTMI  
-A KUBE-SVC-LI77LBOOMGYET5US -m comment --comment "default/showreadiness:showreadiness" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-CGDKBCNM24SZWCMS 
-A KUBE-SVC-LI77LBOOMGYET5US -m comment --comment "default/showreadiness:showreadiness" -j KUBE-SEP-RI4SRNQQXWSTGE2Y
```

So ```kube-proxy``` creates a lot of iptables rules. What does that mean? What are the implications of that in for my network? There’s a great talk from Huawei called [Scale Kubernetes to Support 50,000 services](https://www.youtube.com/watch?v=4-pawkiazEg) that says if you have 5,000 services in your kubernetes cluster, it takes 11 minutes to add a new rule. If that happened to your real cluster I think it would be very bad.

I definitely don’t have 5,000 services in my cluster, but 5,000 isn’t SUCH a bit number. The proposal they give to solve this problem is to replace this iptables backend for kube-proxy with **IPVS** which is a load balancer that lives in the Linux kernel.

It seems like kube-proxy is going in the direction of various Linux kernel based load balancers. I think this is partly because they support UDP load balancing, and other load balancers (like HAProxy) don’t support UDP load balancing.

But I feel comfortable with HAProxy! Is it possible to replace kube-proxy with HAProxy! I googled this and I found [this thread](https://groups.google.com/forum/#!topic/kubernetes-sig-network/3NlBVbTUUU0) on kubernetes-sig-network saying:

_kube-proxy is so awesome, we have used in production for almost a year, it works well most of time, but as we have more and more services in our cluster, we found it was getting hard to debug and maintain. There is no iptables expert in our team, we do have HAProxy&LVS experts, as we have used these for several years, so we decided to replace this distributed proxy with a centralized HAProxy. I think this maybe useful for some other people who are considering using HAProxy with kubernetes, so we just update this project and make it open source: https://github.com/AdoHe/kube2haproxy. If you found it’s useful , please take a look and give a try._

So that’s an interesting option! I definitely don’t have answers here, but, some thoughts:
- Load balancers are complicated
- DNS is also complicated
- If you already have a lot of experience operating one kind of load balancer (like HAProxy), it might make sense to do some extra work to use that instead of starting to use an entirely new kind of load balancer (like kube-proxy)
I’ve been thinking about where we want to be using kube-proxy or kube-dns at all – I think instead it might be better to just invest in Envoy and rely entirely on Envoy for all load balancing & service discovery. So then you just need to be good at operating Envoy.

As you can see my thoughts on how to operate your Kubernetes internal proxies are still pretty confused and I’m still not super experienced with them. It’s totally possible that kube-proxy and kube-dns are fine and that they will just work fine but I still find it helpful to think through what some of the implications of using them are (for example “you can’t have 5,000 Kubernetes services”).

# Ingress
If you’re running a Kubernetes cluster, it’s pretty likely that you actually need HTTP requests to get into your cluster so far. This blog post is already too long and I don’t know much about ingress yet so we’re not going to talk about that.

## Useful links
A couple of useful links, to summarize:
- [The Kubernetes networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model)
- How GKE networking works: https://www.youtube.com/watch?v=y2bhV81MfKQ
- The aforementioned talk on kube-proxy performance: https://www.youtube.com/watch?v=4-pawkiazEg

# I think networking operations is important
My sense of all this Kubernetes networking software is that it’s all still quite new and I’m not sure we (as a community) really know how to operate all of it well. This makes me worried as an operator because I really want my network to keep working! :) Also I feel like as an organization running your own Kubernetes cluster you need to make a pretty large investment into making sure you understand all the pieces so that you can fix things when they break. Which isn’t a bad thing, it’s just a thing.

My plan right now is just to keep learning about how things work and reduce the number of moving parts I need to worry about as much as possible.

As usual I hope this was helpful and I would very much like to know what I got wrong in this post!

