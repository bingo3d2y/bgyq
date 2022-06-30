## gitlab 

### 配置维度

#### Admin  Area

典型：User、Default Branch、Access Token、Network

#### project

典型：Webhook

#### group



### 搭建

#### 修改external_url和listen_port

这个gitlab登陆界面的地址，不要随便使用域名哦，免得无法解析

```bash
$ grep 'xx.com' /etc/gitlab/gitlab.rb
external_url 'http://git.xx.com'
gitlab_rails['gitlab_ssh_host'] = 'git.xx.com'
gitlab_rails['gitlab_email_from'] = 'gitlab@xx.com'
gitlab_rails['gitlab_email_reply_to'] = 'noreply@xx.com'
user['git_user_email'] = 'gitlab@.xx.com
# 修改external_url 为
http://IP:21100
```

修改 /etc/gitlab/gitlab.rb 文件，修改下行，和external_url中IP保持一致

```yaml
# nginx['listen_port'] = nil
```

为

```yaml
nginx['listen_port'] = 21100
```

保存，执行重新配置命令

```bash
$ gitlab-ctl reconfigure
```

通过浏览器 http://ip:8089 即可访问。

#### 修改git_http_url

安装的时候全部使用的默认配置，路径为 /var/opt/gitlab/gitlab-rails/etc/，配置文件为 gitlab.yml ,文件顶部配置如下：

```yaml
host: 10.10.10.19
port: 21100
https: false
```

修改host值为你想使用的外网域名或服务器IP地址即可，保存退出。

```bash
gitlab-ctl restart
```

这个是client 拉代码的地址，要使用正确

### push code to origin empty project

gitlab搭建好之后，如何推送本地代码到一个gitlab新建且是空的项目中呢？

```bash
# push but doing nothing
$ git push
...
No refs in common and none specified; doing nothing.
Perhaps you should specify a branch such as 'master'.

# nice
$ git push --set-upstream origin master
Username for 'http://192.168.40.90:21100': root
Password for 'http://root@192.168.40.90:21100':
Counting objects: 20, done.
Delta compression using up to 96 threads.
Compressing objects: 100% (13/13), done.
Writing objects: 100% (20/20), 6.45 KiB | 0 bytes/s, done.
Total 20 (delta 0), reused 0 (delta 0)
To http://192.168.40.90:21100/root/test.git
 * [new branch]      master -> master

```

end

### webhook：在project下

注意呦，webhook是安装project维度区分的，每个project配置单独的webhook...

创建webhook监听gitlab各类事件

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/gitlab-webhook-1.jpg)

点击test可以触发测试，点击`edit`可以看到test detail 

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/gitlab-webhook-2.jpg)

可以看到webhook发送的信息

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/gitlab-webhook-3.jpg)

下面包含类似这些信息

```json
  "repository": {
    "name": "mydemo",
    "url": "git@10.13.50.217:wyz/mydemo.git",
    "description": "",
    "homepage": "http://10.13.50.217/wyz/mydemo",
    "git_http_url": "http://10.13.50.217/wyz/mydemo.git",
    "git_ssh_url": "git@10.13.50.217:wyz/mydemo.git",
    "visibility_level": 20
  }
```

end

### fluexs：触发流水线

**http://10.13.50.136:21200/fluxes/v1.0/triggers**

使用fluexs进程接受gitlab webhook推送的evenet，然后触发相应的操作。

### Network: Outbound requests

Webhook 不允许本地连接--

添加 gitlab push 自动执行 jenkins, 在 gitlab 中添加 webhook,无法添加，报错： Integrations Settings Url is blocked: Requests to the local network are not allowed

或者提示代码库token不存在或权限不足，应该是代码库external_url地址选择了10.0.0.0/8,172.16.0.0/12,192.168.0.0/16这些私网地址，私网地址的代码库没法直接发布访问，还需要在代码库上配置下允许私网发布。

```bash
# gitlab 已添加 jenkins host

# gitlab 允许从系统钩子向本地网络发起请求
Admin area --> Settings --> Network --> Outbound requests --> 选中
Allow requests to the local network from system hooks

# gitlab 仓库中，配置jenkins 中 hook url 及 token，push 可自动执行 jenkins 任务
```

Requests to these domain(s)/address(es) on the local network will be allowed when local requests from hooks and services are not allowed. IP ranges such as 1:0:0:0:0:0:0:0/124 or 127.0.0.0/28 are supported. Domain wildcards are not supported currently. Use comma, semicolon, or newline to separate multiple entries. The whitelist can hold a maximum of 1000 entries. Domains should use IDNA encoding. Ex: example.com, 192.168.1.1, 127.0.0.0/28, xn--itlab-j1a.com.

### docker image 部署gitlab

```bash
## 创建数据目录
mkdir -p /data/gitlab/config
mkdir -p /data/gitlab/logs
mkdir -p /data/gitlab/data
 
## docker启动，直接安装镜像
## 容器指定端口映射82， ssh端口为2222
## 此时gitlab进程默认使用还是80和22端口，gitlab无法正常访问
sudo docker run --detach \
  --publish 5443:443 \
  --publish 82:82 \
  --publish 2222:22 \
  --name gitlab \
  --volume /data/gitlab/config:/etc/gitlab \
  --volume /data/gitlab/logs:/var/log/gitlab \
  --volume /data/gitlab/data:/var/opt/gitlab \
  gitlab/gitlab-ce:11.11.3-ce.0
 
## 修改配置
vim /data/gitlab/config/gitlab.rb
 
#修改如下语句
external_url 'http://192.168.2.102:82'
 
# https需要下面这句
# nginx['redirect_http_to_https_port'] = 82
 
nginx['listen_port'] = 82
 
# 配置2222端口
gitlab_rails['gitlab_shell_ssh_port'] = 2222
 
 
# 重启gitlab 或者 reconfigure gitlab进程
docker exec -it gitlab gitlab-ctl reconfigure
docker restart gitlab

```

end

### rpm离线安装gitlab

```bash
# 使用阿里云作加速
cd /etc/yum.repos.d/ && rm -f *.repo
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

# 添加 GitLab 镜像源并安装，建议切换国内资源加速访问
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash

# 创建 gitlab-ce 的 repo，使用清华大学加速
vim /etc/yum.repos.d/gitlab_gitlab-ce.repo

[gitlab-ce]
name=gitlab-ce
baseurl=http://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7
repo_gpgcheck=0
gpgcheck=0
enabled=1
gpgkey=https://packages.gitlab.com/gpg.key

# 安装 yum-plugin-downloadonly 插件
yum install -y yum-plugin-downloadonly

# 下载 gitlab-ce 相关 rpm 到指定目录
mkdir -p /tmp/repo/gitlab-ce/
yum install --downloadonly --downloaddir=/tmp/repo/gitlab-ce/ gitlab-ce

# 拷贝文件至内网离线安装
rpm -ivh /tmp/repo/gitlab-ce/*

# 配置并启动 GitLab
gitlab-ctl reconfigure

# 第一次访问 GitLab，系统会重定向页面到重定向到重置密码页面，你需要输入初始化管理员账号的密码，管理员的用户名为 root，重置密码后新密码即为刚输入的密码。
0.0.0.0:80
```

end

### gitlab 14.6删除默认main分支：admin area

#### 解除projectd branch

Settings --> Projectd branch --> unproject

#### 修改默认分支

Settings --> Repository --> Default Branch

#### 删除分支

git push origin --delete Brnach_name

或者界面

Repository --> Branches

