HOMESUBSCRIBE
Consul + Consul-Template with Docker-Compose
14 SEPTEMBER 2015 on docker, Consul, consul-template, nginx, docker-compose
When developing microservices architecture, Docker is a natural way to help reduce system dependencies. Dockerising services allows artifacts to be moved around between different stages/environments, and therefore ensures that deployment stays consistent for all services. However, as the number of services increases, it gets tricky to orchestrate and manage communication between services. In this post, I will show a way to deal with communication with:

Consul
Consul-Template + Nginx
and pre-req:

docker
docker-compose
docker-machine
TL;DR

Check out code example at Consul-Demo
Take a look at How to Cook Microservices for advanced topic and details
Go through steps 1~7 in Running locally below
Running locally
Let's start from cloning the repo and get everything running first before going into details.

#1) clone the repo
$ git clone git@github.com:Neil-Ni/consul-demo.git && cd consul-demo
#2) create a development docker machine
consul-demo$ docker-machine create --driver virtualbox dev  
consul-demo$ eval $(docker-machine env dev)  
#3) start it up and run it in the background. It's your lucky day if this works right away :)
consul-demo$ docker-compose up -d  
#4) scale helloworld up
consul-demo$ docker-compose scale helloworld=3  
#5) get your dev box ip
consul-demo$ docker-machine ip dev  
192.168.99.100  
#6) and open your browser to your dev box ip (192.168.99.100 in this example)
#7) Refresh your browser and you should see the hash code round-robining between 3 hash codes
If you failed at step 4), that means you will need to make sure two ip addresses are correctly specified in docker-compose.yml

1) Docker ip
Same as step 5) docker-machine ip dev, and modify <YOUR DOCKER IP> under registrator:

registrator:  
  command:  -ip=<YOUR DOCKER IP> consul://consul:8500
  image: gliderlabs/registrator:latest
  volumes:
    - "/var/run/docker.sock:/tmp/docker.sock"
  links:
    - consul
2) Docker bridge ip:
Docker0 or docker's virtual ethernet bridge usually defaults to 172.18.42.1 or 172.17.42.1. You can find it with:

$ docker-machine ssh dev
Boot2Docker version 1.8.1, build master : 7f12e95 - Thu Aug 13 03:24:56 UTC 2015  
Docker version 1.8.1, build d12ea79  
docker@dev:~$ ifconfig docker0  
docker0   Link encap:Ethernet  HWaddr 02:42:C3:29:40:A1  
          inet addr:172.18.42.1  Bcast:0.0.0.0  Mask:255.255.0.0
          ......
Take the ip under inet addr (172.18.42.1 in this case), and modify <YOUR DOCKER0> under consul:

consul:  
  command: -server -bootstrap
  image: progrium/consul:latest
  ports:
    - "8400:8400"
    - "8500:8500"
    - "8600:53/udp"
    - "<YOUR DOCKER0>:53:53/udp"
Restart from step 3), and everything should work now!

Other notes on Consul and Consul-Template
Let's take a look at Consul-Template dry run:

consul-demo$ docker-compose run lb consul-template -consul consul:8500 -template "/etc/consul-template/conf/app.conf:/tmp/result:service nginx restart" -dry  
  ...
  upstream helloworld {
    least_conn; 
    server  192.168.99.100:32790 max_fails=3 fail_timeout=60 weight=1;
    server  192.168.99.100:32792 max_fails=3 fail_timeout=60 weight=1;
    server  192.168.99.100:32791 max_fails=3 fail_timeout=60 weight=1;
  }
  server {
    listen                80;
    server_name           helloworld;
    location / {
        proxy_pass        http://helloworld;
    }
  } 
server {  
  listen 80;

  location / {
    proxy_pass http://helloworld;
  }
}
If you still have helloworld scaled to 3 containers, you should see 3 servers under the helloworld block. This is done by Consul-Template. Consul-Template takes the healthy registered containers and automatically updates nginx template and restarts nginx. Try scaling back with docker-compose scale helloworld=2, now you should only see 2 servers under the same block. The actual template is under /lb/templates:

{{range services}}
  {{ if .Tags.Contains "production" }}
  upstream {{.Name}} {
    least_conn;
    {{range service .Name}}
    server  {{.Address}}:{{.Port}} max_fails=3 fail_timeout=60 weight=1;
    {{else}}server 127.0.0.1:65535;{{end}}
  }
  {{end}}
{{end}}
Notice the production here maps to the SERVICE_TAGS environment variable under helloworld in docker-compose.yml. Similarly {{.Name}} maps to SERVICE_NAME:

helloworld:  
  build: ./helloworld
  ports:
    - "3000"
  environment:
    SERVICE_NAME: helloworld
    SERVICE_TAGS: production
Open browser at http://:8500 to view Consul-Ui and check out the list of services. helloworld is under the list, but you will notice other services with names automatically generated by Consul.



Specifying SERVICE_NAME gives you a cleaner service name for DNS lookup, and this is how we can make use of it. Let's add a helloworld2 in docker-compose:

helloworld2:  
  build: ./helloworld
  dns: 
    - <YOUR DOCKER0>
  dns_search: service.consul
  ports:
    - "3001"
  environment:
    SERVICE_NAME: helloworld2
    SERVICE_TAGS: production
Now try this:

#bashing into helloworld2
consul-demo$ docker-compose run helloworld2 bash  
root@ad0d79cddf43:/helloworld# ping helloworld  
PING helloworld.service.consul (192.168.99.100) 56(84) bytes of data.  
64 bytes from 192.168.99.100: icmp_req=1 ttl=64 time=0.025 ms  
64 bytes from 192.168.99.100: icmp_req=2 ttl=64 time=0.059 ms  
64 bytes from 192.168.99.100: icmp_req=3 ttl=64 time=0.065 ms  
root@ad0d79cddf43:/helloworld# curl helloworld  
Hello World! from 500ceb626f99

#bashing into helloworld
consul-demo$ docker-compose run helloworld bash  
root@de0186ef6788:/helloworld# ping helloworld2  
ping: unknown host helloworld2  
helloworld2 container uses Consul's built-in DNS to look up where helloworld is, which is why it can ping helloworld (short for helloworld.service.consul). However the other way around is not possible because we didn't specify DNS ip for helloworld container. And now that we are using Consul, we can communicate between containers without using link.

Thanks for reading! Let me know if you have any questions :)

Neil Ni
Read more posts by this author.

NewYork/Taipei http://neilni.com
Share this post
  

Neil Ni © 2016Proudly published with Ghost