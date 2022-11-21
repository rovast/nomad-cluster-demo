# nomad 集群 demo

## Quick overview

http://192.168.56.101:8500/

<img alt="image" src="https://user-images.githubusercontent.com/9459488/199217746-cde82584-e380-4e0d-93da-ee946f4e7b34.png">

<img alt="image" src="https://user-images.githubusercontent.com/9459488/199217811-c7969cea-9b62-44eb-9a36-03cb242cb496.png">

---

http://192.168.56.101:4646/ui/servers

<img alt="image" src="https://user-images.githubusercontent.com/9459488/199217923-70ae2b30-34bf-40b7-8f8b-677e13cc4c35.png">

<img alt="image" src="https://user-images.githubusercontent.com/9459488/199217958-67d959e5-66d6-448e-8da8-6e32afcef7da.png">

---

http://192.168.56.103:9998/routes?filter=

<img alt="image" src="https://user-images.githubusercontent.com/9459488/199217992-6ca37887-4498-47a8-9497-71bd6f61051f.png">


## Consul 集群

1. 项目根目录执行

```shell
vagrant up
```

创建三个虚拟机

- 192.168.56.101
- 192.168.56.102
- 192.168.56.103


2. consul 组网

```shell
vagrant ssh host1
sudo su

nohup consul agent -server -bootstrap-expect 3 \
-data-dir /etc/consul.d \
-node=node1 \
-bind=192.168.56.101 \
-datacenter=dc1 \
-ui -client 0.0.0.0 \
> /home/vagrant/consul-1.log & 2>&1
```

```shell
vagrant ssh host2
sudo su

nohup consul agent -server -bootstrap-expect 3 \
-data-dir /etc/consul.d \
-node=node2 \
-bind=192.168.56.102 \
-datacenter=dc1 \
-ui -client 0.0.0.0 \
-join 192.168.56.101 \
> /home/vagrant/consul-2.log & 2>&1
```

```shell
vagrant ssh host3
sudo su

nohup consul agent -server -bootstrap-expect 3 \
-data-dir /etc/consul.d \
-node=node3 \
-bind=192.168.56.103 \
-datacenter=dc1 \
-ui -client 0.0.0.0 \
-join 192.168.56.101 \
> /home/vagrant/consul-3.log & 2>&1
```


## nomad 集群

三个虚拟机下新建文件  `/etc/nomad.d/agent.hcl`

```hcl
datacenter = "dc1"
data_dir = "/home/vagrant/nomad.d"

client {
  enabled = true
}

server {
  enabled = true
  bootstrap_expect = 3
}
```


三个虚拟机，在各自 ip 中分别执行

```shell
sudo su # in /home/vagrant

nohup nomad agent -config=/etc/nomad.d/agent.hcl -bind=192.168.56.101 > ./nomad-1.log & 2>&1
nohup nomad agent -config=/etc/nomad.d/agent.hcl -bind=192.168.56.102 > ./nomad-2.log & 2>&1
nohup nomad agent -config=/etc/nomad.d/agent.hcl -bind=192.168.56.103 > ./nomad-3.log & 2>&1
```


## 加入更多 clients

增加 nomad client 配置 `/etc/nomad.d/client.hcl`

```hcl
datacenter = "dc1"
data_dir = "/home/vagrant/nomad.d"

client {
  enabled = true
}
```


```shell
vagrant ssh host4
sudo su

## 以非 server 的方式加入 consul 集群
nohup consul agent \
-data-dir /etc/consul.d \
-node=node4 \
-bind=192.168.56.104 \
-datacenter=dc1 \
-join 192.168.56.101 \
> /home/vagrant/consul-4.log & 2>&1

## 启动 nomad
nohup nomad agent -config=/etc/nomad.d/client.hcl -bind=192.168.56.104 > ./nomad-4.log & 2>&1
```

## 测试任务

```hcl
job "docs" {
  datacenters = ["dc1"]
  type = "service"

  group "example" {
    count = 2
    network {
      port "http" {
        to = 5678
      }
    }
    service {
      name = "doc"
      tags = ["urlprefix-/docs", "urlprefix-docs.test"]
      port = "http"
      check {
        name     = "alive"
        type     = "http"
        path     = "/"
        interval = "10s"
        timeout  = "2s"
      }
    }

    task "server" {
      driver = "docker"
      config {
        image = "hashicorp/http-echo"
        ports = ["http"]
        args = [
          "-listen",
          ":5678",
          "-text",
          "hello world",
        ]
      }
    }
  }
}
```


## 负载均衡（LB）

Ref: https://developer.hashicorp.com/nomad/tutorials/load-balancing/load-balancing-fabio

添加 Job 访问 http://192.168.56.101:4646/ui/jobs，粘贴以下任务

```hcl
job "fabio" {
  datacenters = ["dc1"]
  type = "system"

  group "fabio" {
    network {
      port "lb" {
        static = 9999
      }
      port "ui" {
        static = 9998
      }
    }
    task "fabio" {
      driver = "docker"
      config {
        image = "fabiolb/fabio"
        network_mode = "host"
        ports = ["lb","ui"]
      }

      resources {
        cpu    = 200
        memory = 128
      }
    }
  }
}
```


使用 Apache server 进行测试

```hcl
job "webserver" {
  datacenters = ["dc1"]
  type = "service"

  group "webserver" {
    count = 3
    network {
      port "http" {
        to = 80
      }
    }

    service {
      name = "apache-webserver"
      tags = ["urlprefix-/web", "urlprefix-a.test"]
      port = "http"
      check {
        name     = "alive"
        type     = "http"
        path     = "/"
        interval = "10s"
        timeout  = "2s"
      }
    }

    restart {
      attempts = 2
      interval = "30m"
      delay = "15s"
      mode = "fail"
    }

    task "apache" {
      driver = "docker"
      config {
        image = "httpd:latest"
        ports = ["http"]
      }
    }
  }
}
```
