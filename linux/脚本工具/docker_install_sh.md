```shell
#!/bin/bash

DOCKER_VERSION=${DEFAULT_DOCKER_VERSION:-'19.03.13'}
DOCKER_COMPOSE_VERSION=${DEFAULT_DOCKER_COMPOSE_VERSION:-'1.27.4'}

# 安装docker
install_docker() {
	# 1、卸载旧版本
	sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
	
	# 2、配置docker仓库
	sudo yum install -y yum-utils
	sudo yum-config-manager \
    --add-repo \
    https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/centos/docker-ce.repo
	
	# 3、安装docker
	sudo yum install docker-ce-$DOCKER_VERSION docker-ce-cli-$DOCKER_VERSION containerd.io
	
	# 4、启动docker
	sudo systemctl start docker
}

# 安装docker compose
install_docker_compose() {
	# 1、安装docker compose
	sudo curl -L "https://github.com/docker/compose/releases/download/${DOCKER_COMPOSE_VERSION}/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
	
	# 2、加可执行权限
	sudo chmod +x /usr/local/bin/docker-compose
	
	# 3、创建软链
	sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
	
	# 4、测试是否安装成功
	docker-compose --version
}

install_docker
install_docker_compose
```

