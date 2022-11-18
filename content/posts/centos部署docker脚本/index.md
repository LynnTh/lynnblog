---
title: "Centos部署docker脚本"
date: 2022-11-18T12:24:37+08:00
draft: false
tags:
  - docker
categories:
  - docker
---

使用清华源下载docker，适用于国内环境。

使用systemd管理docker，`docker.service`和`/etc/docker/daemon.json`可根据需要更改。

```shell
#!/bin/bash

BASE=/opt
ARCH=x86_64
DOCKER_VER=19.03.9

DOCKER_URL="https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/static/stable/${ARCH}/docker-${DOCKER_VER}.tgz"

if [[ ! -d "$BASE/down" ]];then
  mkdir -p "$BASE/down"
fi
if [[ ! -d "/opt/kube/bin/" ]];then
  mkdir -p "/opt/kube/bin/"
fi

if [[ -f "$BASE/down/docker-${DOCKER_VER}.tgz" ]];then
  echo "docker binaries already existed"
else
  echo "downloading docker binaries, arch:$ARCH, version:$DOCKER_VER"
  if [[ -e /usr/bin/wget ]];then
    wget -c --no-check-certificate "$DOCKER_URL" || { echo "downloading docker failed"; exit 1; }
  else
    curl -k -C- -O --retry 3 "$DOCKER_URL" || { echo "downloading docker failed"; exit 1; }
  fi
  mv -f "./docker-$DOCKER_VER.tgz" "$BASE/down/docker-$DOCKER_VER.tgz"
fi

tar zxf "$BASE/down/docker-$DOCKER_VER.tgz" -C "$BASE/down" && \
mv -f "$BASE"/down/docker/* /opt/kube/bin && \
ln -sf /opt/kube/bin/docker /bin/docker

systemctl status docker|grep Active|grep -q running && { echo "docker is already running."; return 0; }

echo "generate docker service file"
cat > /etc/systemd/system/docker.service << EOF
[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.io
[Service]
Environment="PATH=/opt/kube/bin:/bin:/sbin:/usr/bin:/usr/sbin"
ExecStart=/opt/kube/bin/dockerd
ExecStartPost=/sbin/iptables -P FORWARD ACCEPT
ExecReload=/bin/kill -s HUP \$MAINPID
Restart=on-failure
RestartSec=5
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
Delegate=yes
KillMode=process
[Install]
WantedBy=multi-user.target
EOF

# configuration for dockerd
mkdir -p /etc/docker
DOCKER_VER_MAIN=$(echo "$DOCKER_VER"|cut -d. -f1)
CGROUP_DRIVER="cgroupfs"
((DOCKER_VER_MAIN>=20)) && CGROUP_DRIVER="systemd"
echo "generate docker config: /etc/docker/daemon.json"

cat > /etc/docker/daemon.json << EOF
{
  "exec-opts": ["native.cgroupdriver=$CGROUP_DRIVER"],
  "registry-mirrors": [
    "https://mirrors.tuna.tsinghua.edu.cn"
  ],
  "insecure-registries": [],
  "max-concurrent-downloads": 10,
  "log-driver": "json-file",
  "log-level": "warn",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
    },
  "data-root": "/var/lib/docker"
}
EOF

echo "enable and start docker"
systemctl enable docker
systemctl daemon-reload && systemctl restart docker && sleep 4

```