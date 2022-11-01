# -*- mode: ruby -*-
# vi: set ft=ruby :

$script = <<SCRIPT
echo "Setting ustc.edu.cn repo"
sudo mv /etc/apt/sources.list /etc/apt/sources.list.bak
(
  cat <<-EOF
deb https://mirrors.ustc.edu.cn/ubuntu/ jammy main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ jammy main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ jammy-security main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ jammy-security main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ jammy-proposed main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ jammy-proposed main restricted universe multiverse  
EOF
) | sudo tee /etc/apt/sources.list


echo "Installing Docker..."
sudo apt-get update
sudo apt-get remove docker docker-engine docker.io
echo '* libraries/restart-without-asking boolean true' | sudo debconf-set-selections
sudo apt-get install apt-transport-https ca-certificates curl software-properties-common -y
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg |  sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88
sudo add-apt-repository \
      "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
      $(lsb_release -cs) \
      stable"
sudo apt-get update
sudo apt-get install -y docker-ce
# Restart docker to make sure we get the latest version of the daemon if there is an upgrade
sudo service docker restart
# Make sure we can actually use docker as the vagrant user
sudo usermod -aG docker vagrant
sudo docker --version

# Packages required for nomad & consul
sudo apt-get install unzip curl vim net-tools -y

echo "Installing Nomad..."
NOMAD_VERSION=1.3.1
cd /tmp/
curl -sSL https://releases.hashicorp.com/nomad/${NOMAD_VERSION}/nomad_${NOMAD_VERSION}_linux_amd64.zip -o nomad.zip
unzip nomad.zip
sudo install nomad /usr/bin/nomad
sudo mkdir -p /etc/nomad.d
sudo chmod a+w /etc/nomad.d


echo "Installing Consul..."
CONSUL_VERSION=1.12.2
curl -sSL https://releases.hashicorp.com/consul/${CONSUL_VERSION}/consul_${CONSUL_VERSION}_linux_amd64.zip > consul.zip
unzip /tmp/consul.zip
sudo install consul /usr/bin/consul

nomad -autocomplete-install

SCRIPT


Vagrant.configure("2") do |config|
  (1..4).each do |i|
    config.vm.define "host#{i}" do |host|
      # 设置虚拟机的Box
      host.vm.box = "bento/ubuntu-22.04" # 22.04 LTS, Jammy
      # 设置虚拟机的主机名
      host.vm.hostname="nomad#{i}"
      host.vm.provision "shell", inline: $script, privileged: false
      # 设置虚拟机的IP
      host.vm.network "private_network", ip: "192.168.56.10#{i}"

      # 设置主机与虚拟机的共享目录
      host.vm.synced_folder ".", "/vagrant", type: "virtualbox"

      # VirtaulBox相关配置
      host.vm.provider "virtualbox" do |v|
      # 启动时是否显示GUI
      # v.gui = true

      # 设置虚拟机的名称
      v.name = "nomad#{i}"
      # 设置虚拟机的内存大小  
      v.memory = 512
      # 设置虚拟机的CPU个数
      v.cpus = 1
      end
    end
  end
end
