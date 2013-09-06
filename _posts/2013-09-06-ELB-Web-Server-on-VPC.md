---           
layout: post
title: ELB Web Server on VPC
date: 2013-09-06 00:00:00 UTC
comments: false
categories: aws vpc ruby
---

The background for this article is mostly spelled out in the last post I
did on how to create a [SSH Gateway Server on VPC](/2013/06/29/SSH-Gateway-Server-on-VPC/). 
This builds on that, and shows how to create a public-facing web server on using an Elastic
Load Balancer (ELB). However, this article may be run standalone.

___Note:___ In order to create a quick web server, I made an AMI out of
a stock Ubuntu 13.04 server that I used `sudo apt-get apache2 php5`, and
then added a `info.php` script to just get some output about the
machine. If you are following along, you can do the same, or configure
the web sever however you'd like.

Again, all of my examples will be in ruby using the `aws-sdk` gem, but the same
results can be achieved using any of their other SDKs.

### Assumptions

  * You have a basic undertsanding of AWS
  * You already have an active AWS account
  * You have a keypair called `key-lab` located at `~/.ec2/key-lab.pem` (rename where appropriate)
  * All commands are probably best done in `irb`, this way you can
    inspect objects as you go along. Make sure to look at the [aws-sdk](http://rdoc.info/gems/aws-sdk/frames/) docs.


### Creating the VPC

First, let's set up the VPC basics like we did with the [last article](/2013/06/29/SSH-Gateway-Server-on-VPC/).

{% highlight ruby %}
    require 'aws-sdk'

    AWS.config(access_key_id: "....", secret_access_key: "....")
    $ec2 = AWS::EC2.new
    $vpc = $ec2.vpcs.create("10.0.0.0/16")

    gateway = $ec2.internet_gateways.create
    gateway.attach($vpc)

    public_route = $vpc.route_tables.first
    public_route.create_route("0.0.0.0/0", internet_gateway: gateway.id)

    public_subnet = $vpc.subnets.create("10.0.0.0/24", availability_zone: "us-east-1d")
    private_subnet = $vpc.subnets.create("10.0.1.0/24", availability_zone: "us-east-1d")
{% endhighlight %}

With ELBs in VPC, you give them their own security group. Since this one
will accept http traffic, we should open `tcp/80` to the world.
 
{% highlight ruby %}
    elb_security_group = $ec2.security_groups.create("elb", vpc_id: $vpc.id)
    elb_security_group.authorize_ingress(:tcp, 80, "0.0.0.0/0")
{% endhighlight %}

Since the load balancer needs to talk to our web server that will get
the `private_security_group`, we should allow incoming requests on `tcp/80`
and from `elb_security_group` through.

{% highlight ruby %}
    private_security_group = $ec2.security_groups.create("private", vpc_id: $vpc.id)
    private_security_group.authorize_ingress(:tcp, 80, {group_id: elb_security_group.id})
{% endhighlight %}


### Launching the Instances

Now it's time to bring up the web server inside the private subnet

{% highlight ruby %}
    web_server = $ec2.instances.create(
        subnet: private_subnet,
        security_groups: private_security_group,
        availability_zone: "us-east-1d",
        instance_type: "t1.micro",
        key_name: "key-lab",
        image_id: 'ami-efd79886' ## Ubuntu 13.04 amd64 with apache/php running
    )

    ## wait around 30-60 seconds for the instance to launch
    sleep 2 until web_server.status == :running
{% endhighlight %}

### Creating the Elastic Load Balancer

Finally, create the ELB. We will put it in the public subnet since it
needs to communicate to the outside world though the internet gateway.
We also specify the `elb_security_group` and listener.

{% highlight ruby %}
    web_elb = $elb.load_balancers.create("test-web",
      security_groups: [elb_security_group],
      subnets: [public_subnet],
      listeners: [
        { protocol: :tcp, port: 80, instance_protocol: :tcp, instance_port: 80 }
      ]
    )
{% endhighlight %}

After we've created the ELB, we need to tell it about the instances that
it should use. Normally, you should have servers in multiple availbility
zones, but for this demonstration, we'll just use one. 
    
{% highlight ruby %}
    web_elb.instances.add(web_server)
    sleep 2 until web_elb.instances.health.first[:state] == "InService"
{% endhighlight %}

You might need to wait a bit for the DNS to propagate. 
`sudo dscacheutil -flushcache` on OS X may help

Finally, your instance should be up and available through the ELB!

{% highlight ruby %}
    system("curl -s http://#{web_elb.dns_name}/info.php | grep 'SERVER_ADDR'")
    #  <tr><td class="e">SERVER_ADDR </td><td class="v">10.0.1.12 </td></tr>
    #  <tr><td class="e">_SERVER["SERVER_ADDR"]</td><td class="v">10.0.1.12</td></tr>
{% endhighlight %}

### Clean-up

Amazon charges you per hour, so if you're just testing and want to shut
everything down, here are some clean-up steps. There's a mess of
dependencies, so you need to do it in a specific order.

{% highlight ruby %}
    web_server.terminate
    sleep 2 until web_server.status == :terminated
    
    ## Delete ELB
    web_elb.delete
    sleep 2 until web_elb.exists? == false
    
    ## Delete the security groups
    private_security_group.delete
    elb_security_group.delete
    
    ## Delete the subnets
    private_subnet.delete
    public_subnet.delete
    
    ## Get rid of the routes
    public_route.routes.last.delete
    
    ## Detach and delete the Internet Gateway
    gateway.detach($vpc)
    gateway.delete
    
    ## Finally, delete the VPC
    $vpc.delete
{% endhighlight %}
