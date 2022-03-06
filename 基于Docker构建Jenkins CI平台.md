## 1、部署Gitlab

### 1.1 部署Gitlab

```
mkdir gitlab
cd gitlab
docker run -d \
  --name gitlab \
  -p 8443:443 \
  -p 9999:80 \
  -p 9998:22 \
  -v $PWD/config:/etc/gitlab \
  -v $PWD/logs:/var/log/gitlab \
  -v $PWD/data:/var/opt/gitlab \
  -v /etc/localtime:/etc/localtime \
  --restart=always \
  lizhenliang/gitlab-ce-zh:latest
```

访问地址：http://IP:9999

初次会先设置管理员密码 ，然后登陆，默认管理员用户名root，密码就是刚设置的。

### 1.2 创建项目，提交测试代码

进入后先创建项目，提交代码，以便后面测试。

```
unzip tomcat-java-demo-master.zip
cd tomcat-java-demo-master
git init
git remote add origin http://192.168.31.70:9999/root/java-demo.git
git add .
git config --global user.email "you@example.com"
git config --global user.name "Your Name"
git commit -m 'all'
git push origin master
```

## 2、部署Harbor镜像仓库

### 2.1 安装docker与docker-compose

### 2.2 解压离线包部署

```
# tar zxvf harbor-offline-installer-v2.0.0.tgz
# cd harbor
# cp harbor.yml.tmpl harbor.yml
# vi harbor.yml
hostname: reg.ctnrs.com
https:   # 先注释https相关配置
harbor_admin_password: Harbor12345
# ./prepare
# ./install.sh
```

### 2.3 在Jenkins主机配置Docker可信任，如果是HTTPS需要拷贝证书

由于habor未配置https，还需要在docker配置可信任。

```
# cat /etc/docker/daemon.json 
{"registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"],
  "insecure-registries": ["192.168.31.70"]
}
# systemctl restart docker
```

## 3、部署Jenkins

### 3.1 准备JDK和Maven环境

将二进制包上传到服务器并解压到工作目录，用于让Jenkins容器挂载使用。

```
# tar zxvf jdk-8u45-linux-x64.tar.gz
# mv jdk1.8.0_45 /usr/local/jdk
# tar zxf apache-maven-3.5.0-bin.tar.gz
# mv apache-maven-3.5.0 /usr/local/maven
```

修改Maven源：

    vi /usr/local/maven/conf/setting.xml
    
    <mirrors>
    
    <mirror>     
      <id>central</id>     
      <mirrorOf>central</mirrorOf>     
      <name>aliyun maven</name>
      <url>https://maven.aliyun.com/repository/public</url>     
    </mirror>
    
    </mirrors>
```
docker run -d --name jenkins -p 80:8080 -p 50000:50000 -u root  \
   -v /opt/jenkins_home:/var/jenkins_home \
   -v /var/run/docker.sock:/var/run/docker.sock   \
   -v /usr/bin/docker:/usr/bin/docker \
   -v /usr/local/maven:/usr/local/maven \
   -v /usr/local/jdk:/usr/local/jdk \
   -v /etc/localtime:/etc/localtime \
   --restart=always \
   --name jenkins jenkins/jenkins
```

访问地址：http://IP

### 3.2 安装插件

**管理Jenkins->系统配置-->管理插件**-->搜索git/pipeline，选中点击安装。

默认从国外网络下载插件，会比较慢，建议修改国内源：

```
cd /opt/jenkins_home/updates
sed -i 's/http:\/\/updates.jenkins-ci.org\/download/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g' default.json && \
sed -i 's/http:\/\/www.google.com/https:\/\/www.baidu.com/g' default.json

docker restart jenkins
```

## 4、发布测试

### 4.1 创建项目并配置

**New Item -> Pipeline -> This project is parameterized -> String Parameter**

- Name：Branch    # 变量名，下面脚本中调用 

- Default Value：master   # 默认分支

- Description：发布的代码分支  # 描述 

### 4.2 Pipeline脚本

```
#!/usr/bin/env groovy

def registry = "192.168.31.70"
def project = "dev"
def app_name = "java-demo"
def image_name = "${registry}/${project}/${app_name}:${Branch}-${BUILD_NUMBER}"
def git_address = "http://192.168.31.70:9999/root/java-demo.git"
def docker_registry_auth = "796b3651-c0b1-4429-b32a-288e0fa66c77"
def git_auth = "35ad5e6a-d132-4cac-9d74-d65223a2d8c0"

pipeline {
    agent any
    stages {
        stage('拉取代码'){
            steps {
              checkout([$class: 'GitSCM', branches: [[name: '${Branch}']], userRemoteConfigs: [[credentialsId: "${git_auth}", url: "${git_address}"]]])
            }
        }

        stage('代码编译'){
           steps {
             sh """
                pwd
                ls
                JAVA_HOME=/usr/local/jdk
                PATH=$JAVA_HOME/bin:/usr/local/maven/bin:$PATH
                mvn clean package -Dmaven.test.skip=true
                """ 
           }
        }

        stage('构建镜像'){
           steps {
                withCredentials([usernamePassword(credentialsId: "${docker_registry_auth}", passwordVariable: 'password', usernameVariable: 'username')]) {
                sh """
                  echo '
                    FROM ${registry}/library/tomcat:v1
                    LABEL maitainer codehorse
                    RUN rm -rf /usr/local/tomcat/webapps/*
                    ADD target/*.war /usr/local/tomcat/webapps/ROOT.war
                  ' > Dockerfile
                  docker build -t ${image_name} .
                  docker login -u ${username} -p '${password}' ${registry}
                  docker push ${image_name}
                """
                }
           } 
        }

        stage('部署到Docker'){
           steps {
              sh """
              REPOSITORY=${image_name}
              docker rm -f tomcat-java-demo |true
              docker container run -d --name tomcat-java-demo -p 88:8080 ${image_name}
              """
            }
        }
    }
}
```

上述脚本中，docker_registry_auth 和git_auth变量的值为Jenkins凭据ID，添加凭据后修改。

### 4.3 添加凭据

**管理Jenkins->安全-->管理凭据->Jnekins->添加凭据->Username with password**

- Username：用户名

- Password：密码

- ID：留空

- Description：描述

分别添加连接git和harbor凭据，并修改脚本为实际凭据ID。

