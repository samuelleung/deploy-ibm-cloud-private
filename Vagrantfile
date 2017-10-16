# Licensed under Apache License Version 2.0
# 
# @ Authur Tim Pouyer tpouyer@us.ibm.com
# https://raw.githubusercontent.com/IBM/deploy-ibm-cloud-private/master/LICENSE

# Software License Terms Acceptence (see https://hub.docker.com/r/ibmcom/icp-inception/)
# If you accept the Software license terms please change the value below to 'accept'
license = "accepted"

# most laptops have at least 8 cores nowadays (adjust based on your laptop hardware)
cpus = '4'

# this will cause memory swaping in the VM 
# performance is decent with SSD drives but may not be with spinning disks
#memory = '4096'

# use this setting for better performance if you have the ram available on your laptop
# uncomment the below line and comment out the above line "#memory = '4096'"
memory = '6000'

# Update version to pull a specific version i.e. version = '2.1.0-beta-1'
version = "2.1.0-beta-3"

# host-only network segment - in most cases you do not have to change this value
# on some systems this network segment may overlap another network already on your
# system. In those cases you will need to change this value to another value
# i.e. 192.168.56 or 192.168.16 etc...
base_segment = '192.168.27'

# enable/disable cluster federation
federation_enabled = 'false'

# enable the metering service
# only used if version < 2.1.0-beta-3 see disabled_management_services below
metering_enabled = 'true'

# disabled mgmt services list
# "va" turns off vulnerability advisor
# "metering" turns off prometheus and grafana metering
# add "metering" from list below to turn off metering
disabled_management_services = '["va"]'

# use apt-cacher-ng & docker registry cache servers
# see instructions in the `README.md` under #Advanced Cache Setup
use_cache = 'false'
cache_host = '192.168.27.99'
apt_cache_port = '3142'
docker_registry_port = '5000'

###############################################################################
#                  DO NOT MODIFY ANYTHING BELOW THIS POINT                    #
###############################################################################

# the following hosts will be accessible from your laptop once provisioning is complete
cfc_hosts = "
[master]
#{base_segment}.100

[worker]
#{base_segment}.101
#{base_segment}.102

[proxy]
#{base_segment}.100

[management]
#{base_segment}.111
"

cfc_config = "
---
# Network in IPv4 CIDR format
network_cidr: 10.1.0.0/16

# Kubernetes service IP range
service_cluster_ip_range: 10.0.0.1/24

# Flag to enable ldap with true, disabled by default.
ldap_enabled: false

# Kubernetes service cluster name
cluster_name: mycluster
cluster_access_ip: #{base_segment}.100
proxy_access_ip: #{base_segment}.100

# Config federation cluster
federation_enabled: false

# set calico tunnel mtu 
calico_tunnel_mtu: 8940

ansible_user: vagrant
ansible_become: true

# enabled/disable python docker install
install_docker_py: false

# maximum wait time for docker-py to start a container
docker_api_timeout: 300

# default timeout value for operations
wait_for_timeout: 3600

# enable the metering service
metering_enabled: false

# disabled mgmt services list
disabled_management_services: #{disabled_management_services}

# following variables are used to pickup internal builds
version: latest
image_tag: latest
image_repo: ibmcom
private_registry_enabled: false
private_registry_server: placeholder.com
docker_username: placeholder
docker_password: placeholder
"

vm_name = "IBM-Cloud-Private-dev-edition"
user_home_path = ENV['HOME']
if Vagrant::Util::Platform.windows?
  user_home_path = Vagrant::Util::Platform.fs_real_path(ENV['USERPROFILE'])
end
base_storage_disk_path = Vagrant::Util::Platform.fs_real_path("#{user_home_path}/VirtualBox VMs/#{vm_name}/ubuntu-16.04-amd64-disk001.vmdk")
lxd_storage_disk_path = Vagrant::Util::Platform.fs_real_path("#{user_home_path}/VirtualBox VMs/#{vm_name}/lxd_storage_disk.vmdk")
extra_storage_disk_path = Vagrant::Util::Platform.fs_real_path("#{user_home_path}/VirtualBox VMs/#{vm_name}/nfs_storage_disk.vmdk")
rsa_private_key = IO.read(Vagrant::Util::Platform.fs_real_path("#{Vagrant.user_data_path}/insecure_private_key"))

if !license.eql? 'accept'
  if !File.exists?("license_accepted")
    puts("################################################################################")
    puts("# You must accept the terms of the Software License under which we are         #")
    puts("# providing the IBM Cloud Private community edition software.                  #")
    puts("#                                                                              #")
    puts("# See license terms here: https://hub.docker.com/r/ibmcom/icp-inception/       #")
    puts("################################################################################")
    puts("Do You Accept the Terms of the Software License? [Y|n]")
    response = STDIN.gets.chomp
    if ['y','accept',''].include?(response.downcase)
      license = 'accept'
      puts("License Terms Accepted!")
      File.write('license_accepted', 'license accepted')
    else
      puts("License Terms Not Accepted... exiting.")
      exit 1
    end
  else # if we found the license_accepted file then proceed
    license = 'accept'
  end
end

image_repo = 'ibmcom'
private_registry_enabled = 'false'
private_registry_server = 'placeholder.com'
docker_username = 'placeholder'
docker_password = 'placeholder'

if File.exist?(".private")
  load File.expand_path(".private")
  version = $version
  image_repo = $image_repo
  private_registry_enabled = $private_registry_enabled
  private_registry_server = $private_registry_server
  docker_username = $docker_username
  docker_password = $docker_password
end

docker_mirror = ''
apt_proxy_conf = ''

if use_cache.downcase.eql? 'true'
	docker_mirror = "\"registry-mirrors\": [\"http://#{cache_host}:#{docker_registry_port}\"]"
	apt_proxy_conf = "
            Acquire {
            	http {
            		Proxy \"http://#{cache_host}:#{apt_cache_port}/\";
            		security.ubuntu.com \"DIRECT\";
            		get.docker.com \"DIRECT\";
            	};
            };"
end

configure_master_ssh_keys = <<SCRIPT
echo "#{rsa_private_key}" >> /home/vagrant/.ssh/id_rsa
echo "$(cat /home/vagrant/.ssh/authorized_keys)" >> /home/vagrant/.ssh/id_rsa.pub
echo 'StrictHostKeyChecking no\nUserKnownHostsFile /dev/null\nLogLevel QUIET' >> /home/vagrant/.ssh/config
SCRIPT

configure_swap_space = <<SCRIPT
sudo rm -f /mnt/swap
sudo fallocate -l 8g /mnt/swap
sudo chmod 600 /mnt/swap
sudo mkswap /mnt/swap
sudo swapon /mnt/swap
echo "/mnt/swap swap swap defaults 0 0" | sudo tee --append /etc/fstab > /dev/null
echo "vm.swappiness = 60" | sudo tee --append /etc/sysctl.conf > /dev/null
echo "vm.vfs_cache_pressure = 10" | sudo tee --append /etc/sysctl.conf > /dev/null
sudo sysctl -p
SCRIPT

configure_performance_settings = <<SCRIPT
echo "net.ipv4.ip_forward = 1" | sudo tee --append /etc/sysctl.conf > /dev/null
echo "net.ipv4.conf.all.rp_filter = 0" | sudo tee --append /etc/sysctl.conf > /dev/null
echo "net.ipv4.conf.default.rp_filter = 0" | sudo tee --append /etc/sysctl.conf > /dev/null
echo "net.ipv6.conf.all.disable_ipv6 = 1" | sudo tee --append /etc/sysctl.conf > /dev/null
echo "net.ipv6.conf.default.disable_ipv6 = 1" | sudo tee --append /etc/sysctl.conf > /dev/null
echo "net.ipv6.conf.lo.disable_ipv6 = 1" | sudo tee --append /etc/sysctl.conf > /dev/null
echo "net.ipv4.tcp_mem = 182757 243679 365514" | sudo tee --append /etc/sysctl.conf > /dev/null
echo "net.core.netdev_max_backlog = 182757" | sudo tee --append /etc/sysctl.conf > /dev/null
echo "net.ipv4.conf.enp0s3.proxy_arp = 1" | sudo tee --append /etc/sysctl.conf > /dev/null
echo "fs.inotify.max_queued_events = 1048576" | sudo tee --append /etc/sysctl.conf > /dev/null
echo "fs.inotify.max_user_instances = 1048576" | sudo tee --append /etc/sysctl.conf > /dev/null
echo "fs.inotify.max_user_watches = 1048576" | sudo tee --append /etc/sysctl.conf > /dev/null
echo "vm.max_map_count = 262144" | sudo tee --append /etc/sysctl.conf > /dev/null
echo "kernel.dmesg_restrict = 0" | sudo tee --append /etc/sysctl.conf > /dev/null

echo "* soft nofile 1048576" | sudo tee --append /etc/security/limits.conf > /dev/null
echo "* hard nofile 1048576" | sudo tee --append /etc/security/limits.conf > /dev/null
echo "root soft nofile 1048576" | sudo tee --append /etc/security/limits.conf > /dev/null
echo "root hard nofile 1048576" | sudo tee --append /etc/security/limits.conf > /dev/null
echo "* soft memlock unlimited" | sudo tee --append /etc/security/limits.conf > /dev/null
echo "* hard memlock unlimited" | sudo tee --append /etc/security/limits.conf > /dev/null
sudo sysctl -p

echo Y | sudo tee /sys/module/fuse/parameters/userns_mounts
echo Y | sudo tee /sys/module/ext4/parameters/userns_mounts
sudo sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="/GRUB_CMDLINE_LINUX_DEFAULT="cgroup_enable=memory swapaccount=1 /g' /etc/default/grub
sudo update-grub
SCRIPT

configure_apt_proxy = <<SCRIPT
sudo bash -c 'cat > /etc/apt/apt.conf.d/02apt-cacher' <<'EOF'
Acquire::http::Proxy "http://#{cache_host}:#{apt_cache_port}/";
Acquire::http::Proxy::security.ubuntu.com "DIRECT";
Acquire::http::Proxy::get.docker.com "DIRECT";
EOF
SCRIPT

install_icp_prereqs = <<SCRIPT
export DEBIAN_FRONTEND=noninteractive
sudo bash -c 'cat > /etc/apt/apt.conf.d/01lean' <<'EOF'
APT::Install-Suggests "0";
APT::Install-Recommends "0";
APT::AutoRemove::SuggestsImportant "false";
APT::AutoRemove::RecommendsImportant "false";
Dir::Cache "";
Dir::Cache::archives "";
EOF
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] http://download.docker.com/linux/ubuntu $(lsb\_release -cs) stable"
sudo apt-get update --yes --quiet
sudo apt-get install --yes --quiet --target-release=xenial-backports lxd lxd-client bridge-utils dnsmasq thin-provisioning-tools \
    curl linux-image-extra-$(uname -r) linux-image-extra-virtual apt-transport-https ca-certificates software-properties-common \
    docker-ce python-setuptools python-pip build-essential python-dev nfs-kernel-server nfs-common aufs-tools ntp
sudo sed -i 's|ExecStart=/usr/bin/dockerd -H fd://|ExecStart=/usr/bin/dockerd -H fd:// --max-concurrent-downloads 10 --mtu=9000|g' /etc/systemd/system/multi-user.target.wants/docker.service
sudo systemctl daemon-reload
sudo systemctl restart docker
sudo -H pip install --upgrade pip
sudo -H pip install docker
sudo usermod -aG lxd vagrant
newgrp lxd
sudo usermod -aG docker vagrant
newgrp docker
sudo bash -c 'cat > /etc/ntp.conf' <<'EOF'
server time1.google.com
server time2.google.com
server time3.google.com
server time4.google.com
EOF
sudo systemctl restart ntp
SCRIPT

configure_docker_mirror = <<SCRIPT
echo '{ "registry-mirrors": ["http://#{cache_host}:#{docker_registry_port}"] }' | sudo tee /etc/docker/daemon.json
sudo systemctl daemon-reload
sudo systemctl restart docker
SCRIPT

add_storage_vol = <<SCRIPT
sudo pvcreate /dev/sdb
sudo vgextend vagrant-vg /dev/sdb
sudo lvcreate -l 100%FREE -n storage vagrant-vg
sudo mkfs -t ext4 /dev/mapper/vagrant--vg-storage
sudo mkdir /storage
sudo mount -t ext4 /dev/mapper/vagrant--vg-storage /storage
echo "/dev/mapper/vagrant--vg-storage /storage        ext4  defaults  0    0"  | sudo tee --append /etc/fstab > /dev/null
sudo chmod 777 /storage
SCRIPT

setup_nfs_shares = <<SCRIPT
# share the '/storage' directory on the nfs server
sudo mkdir /storage/share01 -p
sudo chmod 777 /storage/share01
sudo mkdir /storage/share02 -p
sudo chmod 777 /storage/share02
sudo mkdir /storage/share03 -p
sudo chmod 777 /storage/share03
sudo mkdir /storage/share04 -p
sudo chmod 777 /storage/share04
sudo mkdir /storage/share05 -p
sudo chmod 777 /storage/share05
sudo mkdir /storage/share06 -p
sudo chmod 777 /storage/share06
sudo mkdir /storage/share07 -p
sudo chmod 777 /storage/share07
sudo mkdir /storage/share08 -p
sudo chmod 777 /storage/share08
sudo mkdir /storage/share09 -p
sudo chmod 777 /storage/share09
sudo mkdir /storage/share10 -p
sudo chmod 777 /storage/share10
echo "/storage           *(rw,sync,no_subtree_check,async,insecure,no_root_squash)" | sudo tee --append /etc/exports > /dev/null
sudo systemctl restart nfs-kernel-server
SCRIPT

configure_nat_iptable_rules = <<SCRIPT
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
SCRIPT

configure_lxd = <<SCRIPT
sudo bash -c 'cat >> /etc/network/interfaces' <<'EOF'
    post-up ifconfig enp0s3 mtu 9000
    post-up ifconfig enp0s8 mtu 9000
    post-up ifconfig enp0s8 promisc on

iface enp0s8 inet manual
EOF
sudo ufw disable
cat <<EOF | sudo -H lxd init --preseed
config:
  images.auto_update_interval: 15
storage_pools: 
  - name: lxd
    driver: lvm
    config: 
      volume.size: 40GB
      source: /dev/sdc
networks: 
  - name: lxdbr0
    type: bridge
    config: 
      bridge.driver: native
      bridge.external_interfaces: enp0s8
      bridge.mode: standard
      bridge.mtu: 9000
      ipv4.address: "#{base_segment}.100/24"
      ipv4.dhcp: true
      ipv4.dhcp.ranges: "#{base_segment}.100-#{base_segment}.254"
      ipv4.firewall: false
      ipv4.nat: false
      ipv4.routing: true
      ipv6.address: none
      dns.domain: icp
      dns.mode: managed
      raw.dnsmasq: |
        dhcp-option-force=26,9000
        server=127.0.0.1
profiles: 
  - name: default
    config: 
      boot.autostart: true
      linux.kernel_modules: bridge,br_netfilter,x_tables,ip_tables,ip_vs,ip_set,ipip,xt_mark,xt_multiport,ip_tunnel,tunnel4,netlink_diag,nf_conntrack,nfnetlink,overlay
      raw.lxc: |-
        lxc.aa_profile = unconfined
        lxc.cgroup.devices.allow = a
        lxc.mount.auto=proc:rw sys:rw cgroup:rw
        lxc.cap.drop =
      security.nesting: "true"
      security.privileged: "true"
      user.network-config: |
        version: 1
        config:
          - type: physical
            name: eth0
            mtu: 9000
            subnets:
              - type: dhcp
      user.user-data: |
        #cloud-config
        disable_root: false
        disable_root_opts:
        users:
          - default
          - name: vagrant
            sudo: ['ALL=(ALL) NOPASSWD:ALL']
            groups: sudo
            shell: /bin/bash
            ssh-authorized-keys:
              - $(cat ~/.ssh/id_rsa.pub)
        ntp:
          servers:
            - time1.google.com
            - time2.google.com
            - time3.google.com
            - time4.google.com
        apt:
          preserve_sources_list: true
          conf: |
            APT {
              Get {
                Assume-Yes "true";
                Fix-Broken "true";
              };
              Install-Suggests "false";
              Install-Recommends "false";
              AutoRemove {
                SuggestsImportant "false";
                RecommendsImportant "false";
              };
            };
            Dir {
              Cache {
                archives "";
              };
            };#{apt_proxy_conf}
          sources:
            source1:
              source: "deb [arch=amd64] http://download.docker.com/linux/ubuntu $(lsb\_release -cs) stable"
              key: |
$(curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sed  's/^/                /')
        package_update: true
        package_upgrade: true
        package_reboot_if_required: true
        packages:
          - linux-image-extra-$(uname -r) 
          - linux-image-extra-virtual
          - apt-transport-https 
          - ca-certificates 
          - curl 
          - software-properties-common
          - docker-ce
          - python-setuptools
          - python-pip
          - build-essential
          - python-dev
          - aufs-tools
          - nfs-common
        runcmd:
          - [ mkdir, -p, /var/lib/kubelet ]
          - [ mount, -o, bind, /var/lib/kubelet, /var/lib/kubelet ]
          - [ mount, --make-shared, /var/lib/kubelet ]
          - sed -i 's|ExecStart=/usr/bin/dockerd -H fd://|ExecStart=/usr/bin/dockerd -H fd:// --max-concurrent-downloads 10 --mtu=9000|g' /etc/systemd/system/multi-user.target.wants/docker.service
          - [ systemctl, daemon-reload ]
          - [ systemctl, restart, docker ]
          - [ pip, install, --upgrade, pip ]
          - [ pip, install, docker ]
          - [ usermod, -aG, docker, vagrant ]
          - [ ufw, disable ]
          - [ touch, /DONE ]
        write_files:
          - content: |
              {
                #{docker_mirror}
              }
            path: /etc/docker/daemon.json
    devices: 
      eth0: 
        name: eth0
        nictype: bridged
        parent: lxdbr0
        type: nic
        mtu: 9000
      root: 
        path: /
        pool: lxd
        type: disk
      mem:
        type: unix-char
        path: /dev/mem
  - name: cfc-worker1
    config: 
      boot.autostart.delay: 15
      boot.autostart.priority: 2
      user.meta-data: |
        hostname: cfc-worker1
        fqdn: cfc-worker1.icp
        manage_etc_hosts: true
    devices:
      eth0: 
        name: eth0
        nictype: bridged
        parent: lxdbr0
        type: nic
        mtu: 9000
        ipv4.address: #{base_segment}.101
  - name: cfc-worker2
    config: 
      boot.autostart.delay: 15
      boot.autostart.priority: 3
      user.meta-data: |
        hostname: cfc-worker2
        fqdn: cfc-worker2.icp
        manage_etc_hosts: true
    devices:
      eth0: 
        name: eth0
        nictype: bridged
        parent: lxdbr0
        type: nic
        mtu: 9000
        ipv4.address: #{base_segment}.102
  - name: cfc-manager1
    config: 
      boot.autostart.delay: 15
      boot.autostart.priority: 1
      user.meta-data: |
        hostname: cfc-manager1
        fqdn: cfc-manager1.icp
        manage_etc_hosts: true
    devices:
      eth0: 
        name: eth0
        nictype: bridged
        parent: lxdbr0
        type: nic
        mtu: 9000
        ipv4.address: #{base_segment}.111
EOF
sudo sed -i 's/127.0.0.1\tmaster.icp/#{base_segment}.100\tmaster.icp/g' /etc/hosts
sudo ifconfig enp0s3 mtu 9000
sudo ifconfig enp0s8 mtu 9000
SCRIPT

bring_up_icp_host_interface = <<SCRIPT
sudo ip link set dev enp0s8 up
SCRIPT

set_dnsnameserver_to_lxd_dnsmasq = <<SCRIPT
echo "nameserver #{base_segment}.100\nsearch icp\n" | sudo tee /etc/resolv.conf > /dev/null
SCRIPT

configure_icp_install = <<SCRIPT
mkdir -p /home/vagrant/cluster
echo "#{cfc_hosts}" > /home/vagrant/cluster/hosts
echo "#{rsa_private_key}" > /home/vagrant/cluster/ssh_key
echo '#{cfc_config}' > /home/vagrant/cluster/config.yaml
sed -i "s/image_tag\: latest/image_tag\: #{version}/g" /home/vagrant/cluster/config.yaml
sed -i "s/version\: latest/version\: #{version}/g" /home/vagrant/cluster/config.yaml
sed -i "s|federation_enabled\: false|federation_enabled\: #{federation_enabled}|g" /home/vagrant/cluster/config.yaml
sed -i "s|metering_enabled\: false|metering_enabled\: #{metering_enabled}|g" /home/vagrant/cluster/config.yaml
sed -i "s|image_repo\: ibmcom|image_repo\: #{image_repo}|g" /home/vagrant/cluster/config.yaml
sed -i "s|private_registry_enabled\: false|private_registry_enabled\: #{private_registry_enabled}|g" /home/vagrant/cluster/config.yaml
sed -i "s|private_registry_server\: placeholder.com|private_registry_server\: #{private_registry_server}|g" /home/vagrant/cluster/config.yaml
sed -i "s|docker_username\: placeholder|docker_username\: #{docker_username}|g" /home/vagrant/cluster/config.yaml
sed -i "s|docker_password\: placeholder|docker_password\: #{docker_password}|g" /home/vagrant/cluster/config.yaml
SCRIPT

boot_lxd_worker_nodes = <<SCRIPT
lxc launch -p default -p cfc-manager1 ubuntu:16.04 cfc-manager1
lxc launch -p default -p cfc-worker1 ubuntu:16.04 cfc-worker1
lxc launch -p default -p cfc-worker2 ubuntu:16.04 cfc-worker2
SCRIPT

wait_for_worker_nodes_to_boot = <<SCRIPT
echo ""
echo "Preparing nodes for IBM Cloud Private community edition cluster installation."
echo "This process will take approximately 10-20 minutes depending on network speeds."
echo "Take a break and go grab a cup of coffee, we'll keep working on this while you're away ;-)"
echo ""
FILES="/home/vagrant/cluster/cfc-manager1
/home/vagrant/cluster/cfc-worker1
/home/vagrant/cluster/cfc-worker2"
for file in $FILES
do
  while [ ! -f "$file" ]
  do
    filename=$(basename "$file")
    lxc file pull -p $filename/DONE /home/vagrant/cluster/$filename &> /dev/null
    printf "."
    sleep 10
  done
done
echo "master.icp\t\t\t ready"
echo "cfc-worker1.icp\t\t ready"
echo "cfc-worker2.icp\t\t ready"
echo "cfc-manager1.icp\t\t ready"
SCRIPT

docker_login = <<SCRIPT
curl -fsSL https://clis.ng.bluemix.net/install/linux | bash &> /dev/null
bx plugin install container-registry -r Bluemix &> /dev/null
docker login -u #{docker_username} -p #{docker_password} #{private_registry_server}
SCRIPT

precache_images = <<SCRIPT
echo ""
echo "Seeding IBM Cloud Private installation by pre-caching required docker images."
echo "This may take a few minutes depending on your connection speed and reliability."

echo "Pre-caching docker images...."
echo "Pulling #{image_repo}/icp-inception:#{version}..."
docker pull #{image_repo}/icp-inception:#{version} &> /dev/null
echo "Pulling #{image_repo}/icp-datastore:#{version}..."
docker pull #{image_repo}/icp-datastore:#{version} &> /dev/null
echo "Pulling #{image_repo}/icp-platform-auth:#{version}..."
docker pull #{image_repo}/icp-platform-auth:#{version} &> /dev/null
echo "Pulling #{image_repo}/icp-auth:#{version}..."
docker pull #{image_repo}/icp-auth:#{version} &> /dev/null
echo "Pulling #{image_repo}/iam-token-service:#{version}..."
docker pull #{image_repo}/iam-token-service:#{version} &> /dev/null
echo "Pulling #{image_repo}/kubernetes:v1.7.3..."
docker pull #{image_repo}/kubernetes:v1.7.3 &> /dev/null
echo "Pulling #{image_repo}/helm:v2.5.0..."
docker pull #{image_repo}/helm:v2.5.0 &> /dev/null
SCRIPT

install_icp = <<SCRIPT
exec 3>&1 1>>icp_install_log 2>&1
docker run -e LICENSE=#{license} --net=host -v "$(pwd)/cluster":/installer/cluster #{image_repo}/icp-inception:#{version} install | tee /dev/fd/3
if grep -q fatal icp_install_log; then
	echo "FATAL ERROR OCCURRED DURING INSTALLATION :-(" 1>&3
	cat icp_install_log | grep -C 3 fatal 1>&3
	echo "The install log can be view with: " 1>&3
	echo "vagrant ssh" 1>&3
	echo "cat icp_install_log" 1>&3
	exit 1
fi
SCRIPT

install_kubectl = <<SCRIPT
sudo docker run -e LICENSE=#{license} --net=host -v /usr/local/bin:/data #{image_repo}/kubernetes:v1.7.3 cp /kubectl /data &> /dev/null
kubectl config set-credentials icpadmin --username=admin --password=admin &> /dev/null
kubectl config set-cluster icp --server=http://127.0.0.1:8888 --insecure-skip-tls-verify=true &> /dev/null
kubectl config set-context icp --cluster=icp --user=admin  --namespace=default &> /dev/null
kubectl config use-context icp &> /dev/null
SCRIPT

create_persistant_volumes = <<SCRIPT
sleep 120
cat > volumes.yaml <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
   name: vol01
spec:
   capacity:
      storage: 40Gi
   accessModes:
      - ReadWriteMany
   persistentVolumeReclaimPolicy: Recycle
   nfs:
      path: /storage/share01
      server: #{base_segment}.100
---
apiVersion: v1
kind: PersistentVolume
metadata:
   name: vol02
spec:
   capacity:
      storage: 40Gi
   accessModes:
      - ReadWriteMany
   persistentVolumeReclaimPolicy: Recycle
   nfs:
      path: /storage/share02
      server: #{base_segment}.100
---
apiVersion: v1
kind: PersistentVolume
metadata:
   name: vol03
spec:
   capacity:
      storage: 40Gi
   accessModes:
      - ReadWriteMany
   persistentVolumeReclaimPolicy: Recycle
   nfs:
      path: /storage/share03
      server: #{base_segment}.100
---
apiVersion: v1
kind: PersistentVolume
metadata:
   name: vol04
spec:
   capacity:
      storage: 40Gi
   accessModes:
      - ReadWriteMany
   persistentVolumeReclaimPolicy: Recycle
   nfs:
      path: /storage/share04
      server: #{base_segment}.100
---
apiVersion: v1
kind: PersistentVolume
metadata:
   name: vol05
spec:
   capacity:
      storage: 40Gi
   accessModes:
      - ReadWriteMany
   persistentVolumeReclaimPolicy: Recycle
   nfs:
      path: /storage/share05
      server: #{base_segment}.100
---
apiVersion: v1
kind: PersistentVolume
metadata:
   name: vol06
spec:
   capacity:
      storage: 40Gi
   accessModes:
      - ReadWriteOnce
   persistentVolumeReclaimPolicy: Recycle
   nfs:
      path: /storage/share06
      server: #{base_segment}.100
---
apiVersion: v1
kind: PersistentVolume
metadata:
   name: vol07
spec:
   capacity:
      storage: 40Gi
   accessModes:
      - ReadWriteOnce
   persistentVolumeReclaimPolicy: Recycle
   nfs:
      path: /storage/share07
      server: #{base_segment}.100
---
apiVersion: v1
kind: PersistentVolume
metadata:
   name: vol08
spec:
   capacity:
      storage: 40Gi
   accessModes:
      - ReadWriteOnce
   persistentVolumeReclaimPolicy: Recycle
   nfs:
      path: /storage/share08
      server: #{base_segment}.100
---
apiVersion: v1
kind: PersistentVolume
metadata:
   name: vol09
spec:
   capacity:
      storage: 40Gi
   accessModes:
      - ReadWriteOnce
   persistentVolumeReclaimPolicy: Recycle
   nfs:
      path: /storage/share09
      server: #{base_segment}.100
---
apiVersion: v1
kind: PersistentVolume
metadata:
   name: vol10
spec:
   capacity:
      storage: 40Gi
   accessModes:
      - ReadWriteOnce
   persistentVolumeReclaimPolicy: Recycle
   nfs:
      path: /storage/share10
      server: #{base_segment}.100
EOF
kubectl create -f ./volumes.yaml &> /dev/null
SCRIPT

install_helm = <<SCRIPT
sudo docker run -t --entrypoint=/bin/cp -v /usr/local/bin:/data #{image_repo}/helm:v2.5.0  /helm /data/ &> /dev/null
sudo mkdir -p /var/lib/helm &> /dev/null
export HELM_HOME=/var/lib/helm &> /dev/null
echo "HELM_HOME=/var/lib/helm" >> ~/.bash_profile &> /dev/null
sudo chown -R vagrant:vagrant /var/lib/helm &> /dev/null
helm init --client-only &> /dev/null
SCRIPT

install_startup_script = <<SCRIPT
sudo bash -c 'cat > /usr/local/bin/icp-ce-startup.sh' <<'EOF'
#!/bin/bash

sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
sudo ip link set dev enp0s8 up
echo "nameserver #{base_segment}.100" | sudo tee /etc/resolv.conf > /dev/null
echo "search icp" | sudo tee --append /etc/resolv.conf > /dev/null
EOF
sudo chmod 744 /usr/local/bin/icp-ce-startup.sh

sudo bash -c 'cat > /etc/systemd/system/icp-ce-startup.service' <<'EOF'
[Unit]
After=network.target

[Service]
ExecStart=/usr/local/bin/icp-ce-startup.sh

[Install]
WantedBy=default.target
EOF
sudo chmod 644 /etc/systemd/system/icp-ce-startup.service
sudo systemctl daemon-reload
sudo systemctl enable icp-ce-startup.service
SCRIPT

install_shellinabox = <<SCRIPT
sudo apt-get install -y shellinabox &> /dev/null
SCRIPT

ensure_services_up = <<SCRIPT
echo "Waiting for all IBM Cloud Private Services to start..."
count=0
while [[ '' != $(kubectl get pods --namespace kube-system | sed -n '1!p' | grep -v Running) ]]
do
  if [ "90" -lt "$count" ]; then
  	echo "The following services are still not available after 30 minutes..."
  	kubectl get pods --namespace kube-system | grep -v Running
  	exit
  fi
  count=$(($count+1))
  echo "."
  sleep 20
done
if [ "90" -gt "$count" ]; then
	echo "All IBM Cloud Private Services have been successfully started..."
	kubectl get pods --namespace kube-system &> kube-system-services.list
	cat kube-system-services.list
	rm -f kube-system-services.list
fi
curl -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_10_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/44.0.2403.89 Safari/537.36" -s 'http://bit.ly/2fqp98V' > /dev/null || true
SCRIPT

happy_dance = <<SCRIPT
cat << 'EOF'

                                O MMM .MM  MM7                                  
                         ..M MMMM MMM DMMM MMM.MMMMO.                           
                       M.MM MMMM..MMM MMMM.MMM.NMMMMMM.                         
                     MM MM+MMM:. ,MM: MMMM MMM  MMMMMM                          
                    MM=.MM MMM   MMM. MMMM?MMM  MMM ..~ :MMM.                   
                MM..MM.MM,OMMI   MMM .MMMMMMMM  MMM    MMMMMMM                  
              MMMM MM.NMM ~MMM   MMM..MMMMMMMM  MMM.  OMMMMMMMM                 
           ..MM.MMNMM.MM?  MMM  .MMM..MMMMMMMM  MMM   ~MM+  ,MM                 
          .MMM.MM:MM.?MM. .MMM  IMMM :MMMMMMMM  MMM   .MMM.  .                  
         .MMMMMMMMMM.MMM  .MMM~.MMMD MMMIMMMMM  MMMMMM. MMM   .,MMMM.           
         NMMMMMM.MM: MM.    MMM.MMM, MMM.MMMMM  MMMMMM: =MMM   MMMMMM           
      ..=MM,MMM.MMM MMM.   .MMM.MMM  MMM.8MMMM  MMM.  ,  .MMM .MMM  :           
     MM MMM.MMM.MMM MMM     MMM MMM. MMM  MMMM  MMM.      MMMM.MMMM8.           
   M . MMM.MMMZ7MM  MM.    .MMM MMM  MMM. MMMM  MMM.      .MMMM. MMMMMM   .     
  M M.:MMMMMMM MMM.=MM      MMM.MMM  MMM  MMMM  MMM         MMM   .,MMMM MM     
  M, M.7MMMMM..MMMMMMM.M   ~MMM.MMM .MMM. MMMM  MMM        .MMM ?   .MMM.M.M.   
   MM. N       MMMMMM MMM=MMMM  MMM. MMM  MMMM  MMM. .DMM, MMMM MMMMMMMM  M M   
   MMMM.  MM.   MMMM..MMMMMMM   MMM. MMM  MMMM :MMMMMZMMMMMMMMM  MMMMM  .M.$M   
   MM.MMMMM.. MMM   .  ,MMM8    MMM .MMM  .MMM DMMMMM,  MMMMM      ..MM.. MM    
   MM MM.MMMMMMM:   ~MMMM?       ... :IZ   MM,.NNO?,..         ZMM7.. MMMMMM    
   MM MM MM MMMMMMMMMMMM=     .=MMMMMMMMMMMMNNMMMMMMMMMMMO...  .8MMMMMMMMMMM    
   MM MM .M.M:  .MZMMMMMMMMMMMMMMMMMZ: .  . ......:INMMMMMMMMMMMMMMMMM :MMMM    
   MM MM..M.MM,7MM :  MM  .MMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMM.MMM  MM :MMMM    
   MM MM M..MM,7MM :MMMM MM.MM  MM =MMM.MM:  ...M .MM.. .MM.:M MMM. MM :MMMM    
   NMMMM M..MM,7MM  .:MM MM MM ..M =MM   MMMM MMM .M MMM MM. M MM8~ MM :M MM    
    $MMMMMM.MM,7MM :MMMM .  MM Z.M =MM M MMMM MMM .M MMM =M..  MM +.MM  .NM8    
      .MMMMMMM+7MM ~MMMM M: MM ZM .=MM M =MMM MMM .M.MMM.MM.M. MM.MMIMMMMMM     
     MM  . MMMMMMMMMN.MM MM.MM ZM, =M .   MMM MMM .M .MI MM.MM M?MMMMMMMM       
     MMM    ..  8MMMMMMMMMMMMMDOMM.~M.MMM.MMM.MMM..MM, 7MMMMMMMMMMMMM.          
     MMMM  MMM  :   ..,MMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMM$   .            
     .MMMMMMMM  MM$       .  .. ..:$MMMMMMMMMMMMMMMN=..     .    MMMMMMM.       
      MMMMMMMM  MMM     MMMM  MMM. .,..   .   . .  IMM  MMMMMM  MMM7. I.        
       MMMMOMM.~MMM.   MMMMM7.MMZ  ,MM: .MMM .MMM  MMM .MMMMMM $MM,             
       DM M MM MMMM   MMM  8  MM,  OMM   MMM .MMM. MMM ?MM      MMMM.           
       .MMZ MM.MM=MN  MMM     MM   MMM  .MMM  MMMM MMM MMM       MMMM           
         ,  MM MM MM  MM+     MM.  MMM.  MMM  MMMM:MMM MMMMMM    MMMM           
          ..MM MM MM. MMI    .MM=  MMM   MMM .MMMMMMM..MMMMMM    MMM            
            MMM7MMMMM MMM     MMMMMMMM. .MMM $MMMMMMM  MMM    .MMMM.            
             $M.MMMMM MMM     MMMMMMMM. .MMM.MMMMMMMM.=MM~     MMM              
              . MMMMMM MM    .MMM. MMM. ZMM: MMMMMMMM MMM. IMM..                
                 M :MM MM:    MMM. MMM  MMM  MMMMMMM  MMMMMMMM                  
                    MMM7MM. . MMM  MMM. MMM  MMMDMMM  MMMMMM+                   
                     MM.MMMMMMMMM  MMM..MMM ?MM=~MMM.~MMMM                      
                       . MMMM OMM. MMM .MMM MMM .MM= MM                         
                         . M   MMI MMM  MM  MMM..MM                             
                              ..NM =MM .MM .MD..  .                            


###############################################################################
#          IBM Cloud Private community edition installation complete!         #
#                  The web console is now available at:                       #
#                                                                             #
#                          https://#{base_segment}.100:8443                        #
#                   default username/password is admin/admin                  #
#                                                                             #
#                          Documentation available at:                        #
#               https://www.ibm.com/support/knowledgecenter/SSBS6K            #
#                                                                             #
#                 Request access to the ICP-ce Public Slack!:                 #
#                            http://ibm.biz/BdsHmN                            #
###############################################################################
EOF
SCRIPT

Vagrant.configure(2) do |config|

  if Vagrant.has_plugin?("vagrant-vbguest")
    config.vbguest.auto_update = false
  end

  config.vm.synced_folder ".", "/vagrant", disabled: true

  config.ssh.forward_agent = true
  config.ssh.forward_x11 = true
  config.ssh.insert_key = false
  config.ssh.shell = "bash -c 'BASH_ENV=/etc/profile exec bash'"

  config.vm.provision "shell", privileged: false, inline: configure_master_ssh_keys, keep_color: true, name: "configure_master_ssh_keys"
  config.vm.provision "shell", privileged: false, inline: configure_swap_space, keep_color: true, name: "configure_swap_space"
  config.vm.provision "shell", privileged: false, inline: configure_performance_settings, keep_color: true, name: "configure_performance_settings"
  if use_cache.downcase.eql? 'true'
  	config.vm.provision "shell", privileged: false, inline: configure_apt_proxy, keep_color: true, name: "configure_apt_proxy"
  end
  config.vm.provision "shell", privileged: false, inline: install_icp_prereqs, keep_color: true, name: "install_icp_prereqs"
  if use_cache.downcase.eql? 'true'
  	config.vm.provision "shell", privileged: false, inline: configure_docker_mirror, keep_color: true, name: "configure_docker_mirror"
  end
  config.vm.provision "shell", privileged: false, inline: add_storage_vol, keep_color: true, name: "add_storage_vol"
  config.vm.provision "shell", privileged: false, inline: setup_nfs_shares, keep_color: true, name: "setup_nfs_shares"
  config.vm.provision "shell", privileged: false, inline: configure_nat_iptable_rules, keep_color: true, name: "configure_nat_iptable_rules"
  config.vm.provision "shell", privileged: false, inline: configure_lxd, keep_color: true, name: "configure_lxd"
  config.vm.provision "shell", privileged: false, inline: bring_up_icp_host_interface, keep_color: true, name: "bring_up_icp_host_interface"
  config.vm.provision "shell", privileged: false, inline: set_dnsnameserver_to_lxd_dnsmasq, keep_color: true, name: "set_dnsnameserver_to_lxd_dnsmasq"

  config.vm.define "icp" do |icp|
    icp.vm.box = "bento/ubuntu-16.04"
    icp.vm.hostname = "master.icp"
    icp.vm.box_check_update = false
    icp.vm.network "private_network", ip: "#{base_segment}.100", adapter_ip: "#{base_segment}.1", netmask: "255.255.255.0", mtu: 9000, auto_config: false
    icp.vm.provider "virtualbox" do |virtualbox|
      virtualbox.name = "#{vm_name}"
      virtualbox.gui = false
      virtualbox.customize ["modifyvm", :id, "--apic", "on"] # turn on I/O APIC
      virtualbox.customize ["modifyvm", :id, "--ioapic", "on"] # turn on I/O APIC
      virtualbox.customize ["modifyvm", :id, "--x2apic", "on"] # turn on I/O APIC
      virtualbox.customize ["modifyvm", :id, "--biosapic", "x2apic"] # turn on I/O APIC
      virtualbox.customize ["modifyvm", :id, "--cpus", "#{cpus}"] # set number of vcpus
      virtualbox.customize ["modifyvm", :id, "--memory", "#{memory}"] # set amount of memory allocated vm memory
      virtualbox.customize ["modifyvm", :id, "--ostype", "Ubuntu_64"] # set guest OS type
      virtualbox.customize ["modifyvm", :id, "--natdnshostresolver1", "on"] # enables DNS resolution from guest using host's DNS
      virtualbox.customize ["modifyvm", :id, "--natdnsproxy1", "on"] # enables DNS requests to be proxied via the host
      virtualbox.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"] # turn on promiscuous mode on nic 2
      virtualbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
      virtualbox.customize ["modifyvm", :id, "--nictype2", "virtio"]
      virtualbox.customize ["modifyvm", :id, "--pae", "on"] # enables PAE
      virtualbox.customize ["modifyvm", :id, "--longmode", "on"] # enables long mode (64 bit mode in GUEST OS)
      virtualbox.customize ["modifyvm", :id, "--hpet", "on"] # enables a High Precision Event Timer (HPET)
      virtualbox.customize ["modifyvm", :id, "--hwvirtex", "on"] # turn on host hardware virtualization extensions (VT-x|AMD-V)
      virtualbox.customize ["modifyvm", :id, "--nestedpaging", "on"] # if --hwvirtex is on, this enables nested paging
      virtualbox.customize ["modifyvm", :id, "--largepages", "on"] # if --hwvirtex & --nestedpaging are on
      virtualbox.customize ["modifyvm", :id, "--vtxvpid", "on"] # if --hwvirtex on
      virtualbox.customize ["modifyvm", :id, "--vtxux", "on"] # if --vtux on (Intel VT-x only) enables unrestricted guest mode
      virtualbox.customize ["modifyvm", :id, "--boot1", "disk"] # tells vm to boot from disk only
      virtualbox.customize ["modifyvm", :id, "--rtcuseutc", "on"] # lets the real-time clock (RTC) operate in UTC time
      virtualbox.customize ["modifyvm", :id, "--audio", "none"] # turn audio off
      virtualbox.customize ["modifyvm", :id, "--clipboard", "disabled"] # disable clipboard
      virtualbox.customize ["modifyvm", :id, "--usbehci", "off"] # disable usb hot-plug drivers
      virtualbox.customize ["modifyvm", :id, "--vrde", "off"]
      virtualbox.customize [ "setextradata", :id, "VBoxInternal/Devices/VMMDev/0/Config/GetHostTimeDisabled", 0 ] # turns the timesync on
      virtualbox.customize [ "guestproperty", "set", :id, "/VirtualBox/GuestAdd/VBoxService/--timesync-interval", 10000 ] # sync time every 10 seconds
      virtualbox.customize [ "guestproperty", "set", :id, "/VirtualBox/GuestAdd/VBoxService/--timesync-min-adjust", 100 ] # adjustments if drift > 100 ms
      virtualbox.customize [ "guestproperty", "set", :id, "/VirtualBox/GuestAdd/VBoxService/--timesync-set-on-restore", 1 ] # sync time on restore
      virtualbox.customize [ "guestproperty", "set", :id, "/VirtualBox/GuestAdd/VBoxService/--timesync-set-start", 1 ] # sync time on start
      virtualbox.customize [ "guestproperty", "set", :id, "/VirtualBox/GuestAdd/VBoxService/--timesync-set-threshold", 1000 ] # at 1 second drift, the time will be set and not "smoothly" adjusted 
      virtualbox.customize ['modifyvm', :id, '--cableconnected1', 'on'] # fix for https://github.com/mitchellh/vagrant/issues/7648
      virtualbox.customize ['modifyvm', :id, '--cableconnected2', 'on'] # fix for https://github.com/mitchellh/vagrant/issues/7648
      virtualbox.customize ['storagectl', :id, '--name', 'SATA Controller', '--hostiocache', 'on'] # use host I/O cache
      virtualbox.customize ['storageattach', :id, '--storagectl', 'SATA Controller', '--port', 0, '--device', 0, '--type', 'hdd', '--nonrotational', 'on', '--medium', "#{base_storage_disk_path}"]
      if !File.exists?("#{extra_storage_disk_path}")
        virtualbox.customize ['createhd', '--filename', "#{extra_storage_disk_path}", '--format', 'VMDK', '--size', 500 * 1024]
      end
      if !File.exists?("#{lxd_storage_disk_path}")
        virtualbox.customize ['createhd', '--filename', "#{lxd_storage_disk_path}", '--format', 'VMDK', '--size', 500 * 1024]
      end
      virtualbox.customize ['storageattach', :id, '--storagectl', 'SATA Controller', '--port', 1, '--device', 0, '--type', 'hdd', '--nonrotational', 'on', '--medium', "#{extra_storage_disk_path}"]
      virtualbox.customize ['storageattach', :id, '--storagectl', 'SATA Controller', '--port', 2, '--device', 0, '--type', 'hdd', '--nonrotational', 'on', '--medium', "#{lxd_storage_disk_path}"]
    end

    icp.vm.provision "shell", privileged: false, inline: configure_icp_install, keep_color: true, name: "configure_icp_install"
    icp.vm.provision "shell", privileged: false, inline: boot_lxd_worker_nodes, keep_color: true, name: "boot_lxd_worker_nodes"
    icp.vm.provision "shell", privileged: false, inline: wait_for_worker_nodes_to_boot, keep_color: true, name: "wait_for_worker_nodes_to_boot"
    if !private_registry_enabled.eql? 'false'
      icp.vm.provision "shell", inline: docker_login, keep_color: true, name: "docker_login"
    end
    icp.vm.provision "shell", inline: precache_images, keep_color: true, name: "precache_images"
    icp.vm.provision "shell", inline: install_icp, keep_color: true, name: "install_icp"
    icp.vm.provision "shell", privileged: false, inline: install_kubectl, keep_color: true, name: "install_kubectl"
    icp.vm.provision "shell", privileged: false, inline: create_persistant_volumes, keep_color: true, name: "create_persistant_volumes"
    icp.vm.provision "shell", privileged: false, inline: install_helm, keep_color: true, name: "install_helm"
    icp.vm.provision "shell", privileged: false, inline: install_startup_script, keep_color: true, name: "install_startup_script"
    icp.vm.provision "shell", privileged: false, inline: install_shellinabox, keep_color: true, name: "install_shellinabox"
    icp.vm.provision "shell", privileged: false, inline: ensure_services_up, keep_color: true, name: "ensure_services_up"
    icp.vm.provision "shell", privileged: false, inline: happy_dance, keep_color: true, name: "happy_dance"
  end
end
