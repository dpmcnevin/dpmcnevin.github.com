---           
layout: post
title: SSH Gateway Server on VPC
date: 2013-06-30 00:00:00 UTC
comments: false
categories: aws vpc ruby
---

My latest project at [Gobbler](http://gobbler.com) has been the trasition
of our infrastrcture from "EC2 Classic" to Virtual Private Cloud (VPC).

The AWS docs do a pretty good job explaining VPC concepts, and the web
console does a great job launching a basic VPC setup for you. But, for
me, I am best able to understand a concept when I see the code,
step-by-step.

All of my examples will be in ruby using the `aws-sdk` gem, but the same
results can be achieved using any of their other SDKs.

### Assumptions

  * You have a basic undertsanding of AWS
  * You already have an active AWS account
  * You have a keypair called `key-lab` located at `~/.ec2/key-lab.pem` (rename where appropriate)



### Creating the VPC

First, let's set up our AWS credentials

{% highlight ruby %}
  require 'aws-sdk'

  AWS.config(access_key_id: "....", secret_access_key: "....")
  $ec2 = AWS::EC2.new
{% endhighlight %}

Next, let's create a basic VPC (this will also create a route table
entry)

{% highlight ruby %}
  $vpc = $ec2.vpcs.create("10.0.0.0/16")
{% endhighlight %}

Create an Internet Gateway, this allows servers to have a way to get
back to the internet.

{% highlight ruby %}
  gateway = $ec2.internet_gateways.create
  gateway.attach($vpc)
{% endhighlight %}

Add a default route

{% highlight ruby %}
  public_route = $vpc.route_tables.first
  public_route.create_route("0.0.0.0/0", internet_gateway: gateway.id)
{% endhighlight %}

Now it's time to create 2 subnets, one public, one private

{% highlight ruby %}
  public_subnet = $vpc.subnets.create("10.0.0.0/24", availability_zone: "us-east-1d")
  private_subnet = $vpc.subnets.create("10.0.1.0/24", availability_zone: "us-east-1d")
{% endhighlight %}

We need to create 2 security groups, one for public access, and one for
private. 

{% highlight ruby %}
  public_security_group = $ec2.security_groups.create("public", vpc_id: $vpc.id)
  private_security_group = $ec2.security_groups.create("private", vpc_id: $vpc.id)
{% endhighlight %}

The public security group should be allowed to talk to the
world on tcp/22 (ssh) and the private one should only be allowed to talk
to members of the public security group over tcp/22

{% highlight ruby %}
  public_security_group.authorize_ingress(:tcp, 22, "0.0.0.0/0")
  private_security_group.authorize_ingress(:tcp, 22, {group_id: public_security_group.id})
{% endhighlight %}



### Launching the Instances

Create a network interface that you will attach a public IP to. This
will be on the public subnet and use the public security group.

{% highlight ruby %}
  interface = $ec2.network_interfaces.create(
    subnet: public_subnet,
    security_groups: public_security_group
  )

  sleep 2 until interface.status == :available

  elastic_ip = $ec2.elastic_ips.create(vpc: true)
  elastic_ip.associate(network_interface: interface)

  ## Use this to fill in ELASTIC_IP later on
  puts elastic_ip.public_ip
{% endhighlight %}

Launch an instance into your VPC, since you're specifiying to
use an interface that's already connected to your VPC you will be on
its' subnet and use its' security groups.

{% highlight ruby %}
  ssh_server = $ec2.instances.create(
    availability_zone: "us-east-1d",
    instance_type: "t1.micro",
    key_name: "key-lab",
    image_id: 'ami-e995e380', ## Ubuntu 13.04 amd64
    network_interfaces: [{device_index: 0, network_interface_id: interface.id}]
  )

  ## Wait around 30-60 seconds for the server to come up
  sleep 2 until ssh_server.status == :running
{% endhighlight %}

Now you should be able to SSH to the server

{% highlight bash %}
  $ ssh -i ~/.ec2/key-test.pem ubuntu@ELASTIC_IP
{% endhighlight %}

We can use an SSH config file to make that easier.

_note: You may want to consider adding a DNS CNAME entry for your
elastic IP. This way, if it changes later, there's nothing that needs
updating besides the DNS._

{% highlight bash %}
Host ssh-gateway
   HostName ELASTIC_IP
   User ubuntu
   StrictHostKeyChecking no
   UserKnownHostsFile /dev/null
   IdentityFile ~/.ec2/key-lab.pem
   LogLevel quiet
{% endhighlight %}

Now, just ssh to that server

{% highlight bash %}
  $ ssh ssh-gateway
{% endhighlight %}

Next, let's launch an instance inside the private subnet. This will only
have an internal IP, so it wouldn't otherwise have the ability to
be routable to the outside world.

{% highlight ruby %}
  internal_server = $ec2.instances.create(
    subnet: private_subnet,
    security_groups: private_security_group,
    availability_zone: "us-east-1d",
    instance_type: "t1.micro",
    key_name: "key-lab",
    image_id: 'ami-e995e380' ## Ubuntu 13.04 amd64
  )

  ## Again, wait around 30-60 seconds for the instance to launch
  sleep 2 until internal_server.status == :running

  ## Get the internal IP address of the server
  puts internal_server.private_ip_address
{% endhighlight %}

With some ssh proxying magic, you can get to it through the ssh
server that you've created.

{% highlight bash %}
Host 10.0.1.*
   User ubuntu
   StrictHostKeyChecking no
   UserKnownHostsFile /dev/null
   IdentityFile ~/.ec2/key-lab.pem
   ProxyCommand ssh -W %h:%p ssh-gateway
   LogLevel quiet
{% endhighlight %}

Now you can just ssh to that server's internal IP address

{% highlight bash %}
  $ ssh 10.0.1.247

  $ ssh 10.0.1.247 'ip addr show eth0'
  2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
      link/ether 1e:f1:ea:c9:58:45 brd ff:ff:ff:ff:ff:ff
      inet 10.0.1.247/24 brd 10.0.1.255 scope global eth0
      inet6 fe80::1cf1:eaff:fec9:5845/64 scope link 
         valid_lft forever preferred_lft forever
{% endhighlight %}

### Clean-up

Amazon charges you per hour, so if you're just testing and want to shut
everything down, here are some clean-up steps. There's a mess of
dependencies, so you need to do it in a specific order.

{% highlight ruby %}
  ## Shut down the instances
  internal_server.terminate
  ssh_server.terminate

  ## Wait for the ssh server to go down
  sleep 2 until ssh_server.status == :terminated

  ## Disassociate the Elastic IP and delete the network interface
  elastic_ip.disassociate
  interface.delete

  ## Delete the security groups
  private_security_group.delete
  public_security_group.delete

  ## Delete the subnets
  private_subnet.delete
  public_subnet.delete

  ## Get rid of the subnets
  public_route.routes.last.delete

  ## Detach and delete the Internet Gateway
  gateway.detach($vpc)
  gateway.delete

  ## Finally, delete the VPC
  $vpc.delete
{% endhighlight %}


### More to Come

There is, obviously, a lot more to VPC, and I hope to cover some more
topics in future blog posts. 
