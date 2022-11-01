# nomad 集群 demo

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
-datacenter=nanjing \
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
-datacenter=nanjing \
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
-datacenter=nanjing \
-ui -client 0.0.0.0 \
-join 192.168.56.101 \
> /home/vagrant/consul-3.log & 2>&1
```


## nomad 集群

三个虚拟机下新建文件  `/etc/nomad.d/agent.hcl`

```hcl
datacenter = "nanjing"
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
datacenter = "nanjing"
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
-datacenter=nanjing \
-join 192.168.56.101 \
> /home/vagrant/consul-4.log & 2>&1

## 启动 nomad
nohup nomad agent -config=/etc/nomad.d/client.hcl -bind=192.168.56.104 > ./nomad-4.log & 2>&1
```

## 测试任务

```hcl
job "docs" {
  datacenters = ["nanjing"]
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
  datacenters = ["nanjing"]
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
  datacenters = ["nanjing"]
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
