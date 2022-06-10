# 实验环境docekr相关命令

创建学习容器：

    docker run -idt -p 22:22 -p 80:80 -p 8080:8080 -p 443:443 -p 3306:3306 -p 3307:3307 -p 3308:3308 -p 3309:3309 -p 3310:3310 -p 5432:5432 --name mysql1 -v D:/wmnt/:/wmnt centos

    docker run --privileged -idt -p 22:22 -p 80:80 -p 8080:8080 -p 443:443 -p 3306:3306 -p 3307:3307 -p 3308:3308 -p 3309:3309 -p 3310:3310 -p 5432:5432 --name mysql1 -v D:/wmnt/:/wmnt centos:centos8.2.2004 /usr/sbin/init

    docker run --privileged -idt -p 22:22 -p 80:80 -p 8080:8080 -p 443:443 -p 3306:3306 -p 6379:6379 -p 27017:27017 --name goaction -v D:/wmnt/:/wmnt centos:centos8.2.2004 /usr/sbin/init

    docker exec -it mysql1 bash

    or

    docker exec -it mysql1 zsh
    
    docker run -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=123123 -e POSTGRES_DB=docker -p 5432:5432 -d --name pg1 -v D:/dshare/:/dshare postgres

    docker exec -it pg1 bash

    su postgres
    createdb chitchat
    psql -f setup.sql -d chitchat

    docker run -d --name sonar -p 9000:9000 -e sonar.jdbc.username=postgres -e sonar.jdbc.password=123123 -e sonar.jdbc.url=jdbc:postgresql://localhost:5432/sonar -v D:/dshare/:/dshare sonarqube

    # 自封装
    docker run --privileged -idt -p 3306:3306 -p 3307:3307 -p 3308:3308 -p 3309:3309 -p 3310:3310 -p 5432:5432 --name learnmysql -v D:/wmnt/:/wmnt hou/mysql1:version1 /usr/sbin/init

    docker exec -it learnmysql zsh

安装oh-my-zsh

    vim /etc/resolv.conf
    添加阿里的dns
    nameserver  223.5.5.5
    nameserver  223.6.6.6

    yum install git-core(安装git)
    git --version(查看git版本)
    yum install wget（安装wget）
    sudo yum update && sudo yum -y install zsh
    sh -c "$(wget https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh -O -)"

    如果不能安装可以尝试添加github hosts

    https://github.com/ineo6/hosts

php开发使用容器：

    docker run -idt -p 22:22 -p 80:80 -p 8080:8080 -p 443:443 -p 3306:3306 -p 3307:3307 -p 3308:3308 -p 3309:3309 -p 3310:3310 -p 5432:5432 --name mydev -v /Users/hou/repo:/repo IMAGEID


安装Git
cd /tmp
wget https://mirrors.edge.kernel.org/pub/software/scm/git/git-2.30.2.tar.gz
tar -xvzf git-2.30.2.tar.gz
cd git-2.30.2/
./configure
make
sudo make install
git --version          # 输出 git 版本号，说明安装成功
git version 2.30.2