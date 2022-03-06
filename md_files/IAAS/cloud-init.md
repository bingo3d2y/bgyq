## cloud-init

保证使用私有镜像创建的新云服务器可以自定义配置的工具

### instance metedata and config drive

https://zhangchenchen.github.io/page/6/

所谓 config drive的方式就是将metadata信息写入虚拟机的一个特殊配置设备中，虚拟机启动的时候会挂载并读取metadata信息。要想实现这个功能，还需要宿主机和镜像满足一些条件：

1. 宿主机满足条件：
   - 虚拟化方式为以下几种：libvirt,xen,hyper-v,vmware.
   - 对于 libvirt, xen, vmware的虚拟化方式，需要安装genisoimage.并且需要设置mkisofs为 程序的安装目录，如果是跟nova-compute安装在一个目录内就不用了。
   - 如果是 hyper-v的虚拟化方式，需要用mkisofs的命令行指定mkisofs.exe 程序的绝对安装路径，且在hyperv的配置文件中设置qemu_img_cmd 为qemu-img 的命令行安装路径。
2. 镜像满足条件：
   - 安装了cloud-init，且最好是0.7.1+ 的版本。如果是没有安装的话，就需要自己去定制脚本了。
   - 如果使用xen 虚拟化，需要配置 xenapi_disable_agent 为true。

```bash
$ ls -l /dev/disk/by-label/config-2
lrwxrwxrwx. 1 root root 10 Mar  6 10:43 /dev/disk/by-label/config-2 -> ../../vdb1
$ ls -l /dev/vdb
vdb   vdb1
$ ls -l /dev/vdb1
brw-rw----. 1 root disk 253, 17 Mar  6 10:43 /dev/vdb1
$ mkdir /tmp/meta
$ mount  /dev/vdb1  /tmp/meta/
$ cd /tmp/meta/
# metedata
$ ls
ec2  openstack
$ cat ec2/latest/meta-data.json |jq
{
    "reservation-id":"r-8izy3khj",
    "hostname":"kafkatwo-kafka-master-1-8edf9-0.novalocal",
    "security-groups":[
        "default"
    ],
    "public-ipv4":"",
    "ami-manifest-path":"FIXME",
    "instance-type":"4*8*100",
    "instance-id":"i-00000022",
    "local-ipv4":"171.251.48.130",
    "local-hostname":"kafkatwo-kafka-master-1-8edf9-0.novalocal",
    "placement":{
        "availability-zone":"sdswzx"
    },
    "ami-launch-index":0,
    "public-hostname":"kafkatwo-kafka-master-1-8edf9-0.novalocal",
    "public-keys":{
        "0":{
            "openssh-key":"ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCgLoLx3FjwUikfvlnornBOUcSEI0oHpNw2RCrBtSaQv5ZmXOt/q8NH56p5/ITHIt72HGGUELUhyJgFF8lUZE38L3pdXIRA3jj7kIKR1ezYk0YBjJ3UVGWwCRa5y+Px50mASXL1sKPyzrGyL9oUgUXP8bQ5aK0Ko5fdycILI9wjoewGId/rFfXpwnVFywuYAuzcddTWQeTL3EQ8MyDLIT1i1KdN9xgjkI8SWuYTaXnw2+/ZS+qpMFHB7SIBFo5+9AD+t471aUIYOMI42CZmNcDTe/7nRYcVRpCUFDFdva27S1wgbohW7bfcZdvRf7UGoMNltyAo+dY/l/zz+4VurjEn Generated-by-Nova",
            "_name":"0=PaaS"
        }
    },
    "ami-id":"ami-00000001",
    "instance-action":"none",
    "block-device-mapping":{
        "ami":"vda",
        "root":"/dev/vda"
    }
}

```

end

### 169.254.169.254

https://support.huaweicloud.com/vpc_faq/vpc_faq_0084.html

各个云厂商定义的metadata file path都不一样

```bash
curl http://169.254.169.254/meta_file_path
```

华为：

**curl http://169.254.169.254/openstack/latest/meta_data.json**

华三

**curl http://169.254.169.254/2009-04-04/meta-data/instance-id**

