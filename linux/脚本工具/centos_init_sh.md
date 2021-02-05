```shell
#!/bin/bash

NTP_SERVER=${INIT_NTPSERVER:-'ntp1.aliyun.com'} # NTP服务器，默认为：ntp1.aliyun.com

# 设置主机名
hostname_config() {
	read -p "主机名是否需要更改（yes/no）:" input
	if [ $input == "y" -o $input == "Y" -o $input == "yes" -o $input == "YES" ];then
		read -p "输入需要更改的主机名：" change_hostname
		hostnamectl set-hostname $change_hostname
		echo -e "\033[32m 主机名已设置为：$(hostname) \033[0m"
	else
		echo -e "\033[32m 当前主机名为：$(hostname) \033[0m"
	fi
}

# 设置静态IP地址
ip_config() {
	# 获取IP地址
	static_ip=$(ip a|grep ens33|grep inet|awk -F "/" '{ print $1 }'|awk -F " " '{ print $2 }')
	# 获取gateway
	static_gateway=$(echo $static_ip|awk -F "." '{ print $1"."$2"."$3".2" }')
	# 获取网络配置文件名
	network_name=ifcfg-$(ip a |grep inet |grep brd |awk -F ' ' '{ print $NF }')
	# 备份网络配置
	cd /etc/sysconfig/network-scripts
	mkdir -p back/$(date +%Y%m%d)
	cp $network_name back/$(date +%Y%m%d)/${network_name}.$(date +%H%M)
	
	grep -aiw "dhcp" $network_name
	if [ $? -eq 0 ];then
		sed -i "s/dhcp/static/g" $network_name
		echo "IPADDR=$static_ip" >>$network_name
		echo "GATEWAY=$static_gateway" >>$network_name
		echo "NETMASK=255.255.255.0" >>$network_name
		echo "DNS1=114.114.114.114" >>$network_name
		systemctl restart network
		echo -e "\033[32m 当前IP为 $static_ip \033[0m"
		echo -e "\033[32m 当前网关为 $static_gateway \033[0m"
		echo -e "\033[32m 当前子网掩码为 255.255.255.0 \033[0m"
	else
		#echo -e "\033[32m 当前为静态IP，不需要更改 \033[0m"
		read -p "当前IP $hhm_ip 为静态IP,是否需要更改（yes/no）:" input
		if [ $input == "y" -o $input == "Y" -o $input == "yes" -o $input == "YES" ];then
			read -p "输入需要更改的IP：" change_ip
			sed -i "s/IPADDR=$static_ip/IPADDR=$change_ip/g" $network_name
			change_gateway=$(echo $change_ip|awk -F "." '{ print $1"."$2"."$3".2" }')
			sed -i "s/GATEWAY=$static_gateway/GATEWAY=$change_gateway/g" $network_name
			echo -e "\033[32m 当前IP为 $change_ip \033[0m"
			echo -e "\033[32m 当前网关为 $change_gateway \033[0m"
			echo -e "\033[32m 当前子网掩码为 255.255.255.0 \033[0m"
			systemctl restart network
		else
			echo -e "\033[32m 当前IP为 $static_ip \033[0m"
			echo -e "\033[32m 当前网关为 $static_gateway \033[0m"
        	echo -e "\033[32m 当前子网掩码为 255.255.255.0 \033[0m"
		fi
	fi
}

# 配置时区
timezone_config() {
	timedatectl set-timezone Asia/Shanghai
	echo -e "\033[32m 时区已配置 \033[0m"
}

# 配置网络时钟源
ntp_config() {
	echo "* * * * * /usr/sbin/ntpdate ${NTP_SERVER}" >/var/spool/cron/root
	echo -e "\033[32m 网络时钟源已配置 \033[0m"
}

# 设置系统语言编码
locale_config() {
	echo 'LANG="en_US.UTF-8"' >/etc/locale.conf
	echo 'export LANG="en_US.UTF-8"' >>/etc/profile.d/custom.sh
	echo -e "\033[32m 系统语言编码已配置 \033[0m"
}

# 关闭SELINUX
selinux_config() {
	if [ -f /etc/selinux/config ]; then
		sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config
	fi
	echo -e "\033[32m SELINUX已关闭，重启操作系统后生效,请稍后重启操作系统。 \033[0m"
}

# 关闭firewalld
firewalld_config() {
	systemctl disable firewalld && systemctl stop firewalld

	firewall-cmd --state &>/dev/null

	if [ $? -eq 252 ]; then
		echo -e "\033[32m firewalld已关闭 \033[0m"
	fi
}

# 配置文件描述符大小
ulimit_config() {
	# ulimit -SHn 102400命令临时生效，或者加入到/etc/rc.local，然后每次重启生效
	echo "ulimit -SHn 102400" >>/etc/rc.local
	cat >>/etc/security/limits.conf <<EOF
     *           soft   nofile       102400
     *           hard   nofile       102400
     *           soft   nproc        102400
     *           hard   nproc        102400
EOF
	echo -e "\033[32m 文件描述符已配置 \033[0m"
}

# 配置SSH
sshd_config() {
	sed -i 's/^GSSAPIAuthentication yes$/GSSAPIAuthentication no/' /etc/ssh/sshd_config
	sed -i 's/#UseDNS yes/UseDNS no/' /etc/ssh/sshd_config
	systemctl start sshd
	echo -e "\033[32m SSH已配置 \033[0m"
}

# 配置VIM
vim_config() {
	sed -i "8 s/^/alias vi='vim'/" /root/.bashrc
	cat >>/root/.vimrc <<EOF
	set fenc=utf-8 "设定默认解码
	set fencs=utf-8,usc-bom,euc-jp,gb18030,gbk,gb2312,cp936
	set nocp "或者 set nocompatible 用于关闭VI的兼容模式
	set number "显示行号
	set ai "或者 set autoindent vim使用自动对齐，也就是把当前行的对齐格式应用到下一行
	set si "或者 set smartindent 依据上面的对齐格式，智能的选择对齐方式
	set tabstop=4 "设置tab键为4个空格
	set sw=4 "或者 set shiftwidth 设置当行之间交错时使用4个空格
	set ruler "设置在编辑过程中,于右下角显示光标位置的状态行
	set incsearch "设置增量搜索,这样的查询比较smart
	set showmatch "高亮显示匹配的括号
	set matchtime=5 "匹配括号高亮时间(单位为 1/10 s)
	set ignorecase "在搜索的时候忽略大小写
	syntax on
EOF
	echo -e "\033[32m VIM已配置 \033[0m"
}

# 配置内核参数
kernel_parameter_config() {
	cp /etc/sysctl.conf /etc/sysctl.conf.bak
	cat >/etc/sysctl.conf <<EOF
     net.ipv4.ip_forward = 0
     net.ipv4.conf.default.rp_filter = 1
     net.ipv4.conf.default.accept_source_route = 0
     kernel.sysrq = 0
     kernel.core_uses_pid = 1
     net.ipv4.tcp_syncookies = 1
     kernel.msgmnb = 65536
     kernel.msgmax = 65536
     kernel.shmmax = 68719476736
     kernel.shmall = 4294967296
     net.ipv4.tcp_max_tw_buckets = 6000
     net.ipv4.tcp_sack = 1
     net.ipv4.tcp_window_scaling = 1
     net.ipv4.tcp_rmem = 4096 87380 4194304
     net.ipv4.tcp_wmem = 4096 16384 4194304
     net.core.wmem_default = 8388608
     net.core.rmem_default = 8388608
     net.core.rmem_max = 16777216
     net.core.wmem_max = 16777216
     net.core.netdev_max_backlog = 262144
     net.core.somaxconn = 262144
     net.ipv4.tcp_max_orphans = 3276800
     net.ipv4.tcp_max_syn_backlog = 262144
     net.ipv4.tcp_timestamps = 0
     net.ipv4.tcp_synack_retries = 1
     net.ipv4.tcp_syn_retries = 1
     net.ipv4.tcp_tw_recycle = 1
     net.ipv4.tcp_tw_reuse = 1
     net.ipv4.tcp_mem = 94500000 915000000 927000000
     net.ipv4.tcp_fin_timeout = 1
     net.ipv4.tcp_keepalive_time = 1200
     net.ipv4.ip_local_port_range = 1024 65535
EOF
	# 使配置生效
	/sbin/sysctl -p
	echo -e "\033[32m 内核参数已配置 \033[0m"
}

# 配置YUM国内源镜像
yum_config() {
	# 1、备份CentOS-Base.repo
	cd /etc/yum.repos.d
	mkdir -p back/$(date +%Y%m%d)
	cp CentOS-Base.repo back/$(date +%Y%m%d)/CentOS-Base.repo.$(date +%H%M)
	
	# 2、修改配置文件
	cat >/etc/yum.repos.d/CentOS-Base.repo <<EOF
[base]
name=CentOS-\$releasever - Base
baseurl=http://mirrors.tuna.tsinghua.edu.cn/centos/\$releasever/os/\$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

#released updates
[updates]
name=CentOS-\$releasever - Updates
baseurl=http://mirrors.tuna.tsinghua.edu.cn/centos/\$releasever/updates/\$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

#additional packages that may be useful
[extras]
name=CentOS-\$releasever - Extras
baseurl=http://mirrors.tuna.tsinghua.edu.cn/centos/\$releasever/extras/\$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

#additional packages that extend functionality of existing packages
[centosplus]
name=CentOS-\$releasever - Plus
baseurl=http://mirrors.tuna.tsinghua.edu.cn/centos/\$releasever/centosplus/\$basearch/
gpgcheck=1
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
EOF
	# 3、清除系统所有的yum缓存
	yum clean all
	# 4、生成yum缓存
	yum makecache
	echo -e "\033[32m 清华大学开源软件镜像站YUM源已配置 \033[0m"
}

# 安装软件并更新
update_system() {
	yum -y install net-tools lrzsz gcc gcc-c++ make cmake libxml2-devel openssl-devel curl curl-devel unzip sudo ntp libaio-devel wget vim ncurses-devel autoconf automake zlib* fiex* libxml* libmcrypt* libtool-ltdl-devel* python-devel
	yum update
	echo -e "\033[32m 系统软件安装并已更新 \033[0m"
}

main() {
	start_menu

	while true; do
		case $choice in
		0)
			clear
			yum_config
			update_system
			hostname_config
			ip_config
			timezone_config
			ntp_config
			locale_config
			ulimit_config
			sshd_config
			vim_config
			kernel_parameter_config
			sleep 2
			;;
		1)
			clear
			hostname_config $1
			sleep 2
			;;
		2)
			clear
			ip_config
			sleep 2
			;;
		3)
			clear
			firewalld_config
			sleep 2
			;;
		4)
			clear
			selinux_config
			sleep 2
			;;
		5)
			clear
			yum_config
			sleep 2
			;;
		6)
			clear
			ntp_config
			sleep 2
			;;
		7)
			clear
			update_system
			sleep 2
			;;
		Q | q)
			echo "退出系统初始化程序。"
			sleep 2
			break
			;;
		*)
			echo -e "\t\t\t\033[31m 请按菜单选择 \033[0m"
			;;
		esac

		start_menu

	done
}

start_menu() {
	cat <<EOF
	==========Linux System INIT==========
			0)全局设置
			1)设置主机名
			2)设置IP地址
			3)关闭防火墙
			4)关闭SELINUX
			5)配置YUM国内镜像
			6)配置网络时钟源
			7)安装软件并更新
			Q|q)退出
	=====================================
EOF
	read -p "请选择相应选项完成系统初始化[输入Q退出本程序]：" choice
}

# 执行主程序
main
```

