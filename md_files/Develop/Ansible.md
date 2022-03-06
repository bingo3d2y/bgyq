## Ansible

官方文档真的不咋样...

### YAML

YAML 支持的数据结构有三种。

对象：键值对的集合，又称为映射（mapping）/ 哈希（hashes） / 字典（dictionary）
数组：一组按次序排列的值，又称为序列（sequence） / 列表（list）
纯量（scalars）：单个的、不可再分的值



YAML使用缩进表示层级关系，同一层级的缩进必须保持一致。缩进只能使用空格不能使用tab，不过空多少格没有规定。

YAML中的数组表示方式为在元素前加上`-`（和Markdown很像）；键值对的表示方式和JSON差不多，不过YAML中的key和字符串不用加引号。

YAML的库几乎和JSON一样无处不在. 除了**支持注释**,换行符分隔,多行字符串***,***裸字符串和更灵活的类型系统之外,YAML也支持**引用文件**,以避免重复代码.

json的注释不是标准, 大多数解析器不支持注释，一个配置文件不加注释是一种怎样的体验? 

> JSON不是框架协议。 它是一种无语言格式。 因此，没有为JSON定义注释的格式。
>
> JSON使用为`"_comment"`（或其他key）的指定数据元素，该数据元素将被使用JSON数据的应用程序忽略。

yaml容易发生空格引发的惨案

```yaml
# 要求读环境变量的值[test, pts_zest, dev_con] 作为 excludes的值
# app.yaml
cluster:
  excludes:
  - zest_api
  - pts_gateway
  tag: pts
## 对象
animal: pets
## 行内写法
hash: { name: Steve, foo: bar } 

## 数组写法： 展开式
animal:
- Cat
- Dog
- Goldfish
## 上面那种 写法得写死人啊
animal: [Cat, Dog]
```

使用场景，shell输出yaml 配置文件：

一行搞定--

```bash
 if [ -n "${CLUSTER_NO_TAG_SERVICES}" ];then
       echo "cluster: {excludes: [${CLUSTER_NO_TAG_SERVICES}], tag: ${CLUSTER_TAG}}" >> /app.yml

```

end

### buildin plugin

https://docs.ansible.com/ansible/latest/collections/ansible/builtin/index.html

### Inventory

The default location for inventory is a file called `/etc/ansible/hosts`. You can specify a different inventory file at the command line using the `-i <path>` option.

可以在host inventory，定义额外的key-value信息，在执行playbook时通过`{{key_name}}`引用。

```bash
## 定义额外信息
root@Dodo:/tmp/task# cat hosts_ip
[local]
127.0.0.1  myid=2
root@Dodo:/tmp/task# cat test.yaml
- hosts: all
  vars:
    items:
    - [2,3]
    - 8
  tasks:
  ## 输出
  - name: "echo"
    debug:
      msg: "{{ myid }}"
## output
TASK [echo] ******************************************************************************************************************
ok: [127.0.0.1] => {
    "msg": 2
}

```

实际应用

```yaml
## template 定义模块，引用变量
# roles/kafka/templates/server.properties.j2
# The id of the broker. This must be set to a unique integer for each broker.
broker.id={{ myid }}

## 使用jinja2 模板 作为 kafka config
- name: config kafka.service
  template: src=kafka.service.j2 dest=/usr/lib/systemd/system/kafka.service
  
## inventory

[kafka]
10.249.0.134 hostname=kafka1 myid=1
10.249.0.135 hostname=kafka2 myid=2
10.249.0.136 hostname=kafka3 myid=3
```

end

### gather-facts

Gathers facts about remote hosts。

实际应用中都取消了Gathers facts

```bash
# one host
root@Dodo:/tmp/task# cat hosts_ip
[local]
127.0.0.1

# 对比时间差距
root@Dodo:/tmp/task# time ansible-playbook --private-key /root/.ssh/id_rsa -i hosts_ip  test.yaml &> /dev/null

real    0m3.392s
user    0m1.292s
sys     0m0.350s

#   gather_facts: no
root@Dodo:/tmp/task# time ansible-playbook --private-key /root/.ssh/id_rsa -i hosts_ip  test.yaml-gather-facts-no &> /dev/null

real    0m1.178s
user    0m0.846s
sys     0m0.212s

```

end

### Ignore errors

By default Ansible stops executing tasks on a host when a task fails on that host. You can use `ignore_errors` to continue on in spite of the failure.

```yaml
- name: Do not count this as a failure
  ansible.builtin.command: /bin/false
  ignore_errors: yes
```

The `ignore_errors` directive only works when the task is able to run and returns a value of ‘failed’. It does not make Ansible ignore undefined variable errors, connection failures, execution issues (for example, missing packages), or syntax errors.

### Tags

对于希望执行特定task的场景，可以通过使用tags进行限定。此种情况下只有在定义时设定了此tag的task才会被执行，执行时多个tag中间用逗号分开。

If you have a large playbook, it may be useful to run only specific parts of it instead of running the entire playbook. You can do this with Ansible tags. Using tags to execute or skip selected tasks is a two-step process:

1. Add tags to your tasks, either individually or with tag inheritance from a block, play, role, or import.
2. Select or skip tags when you run your playbook

实际栗子：

```bash
## 定义role
root@Dodo:/tmp/task/roles/hello# cat /tmp/task/roles/hello/tasks/main.yaml
- name: "test, hi"
  debug:
    msg: "hi, chandler"
  tags: "hi"
- name: "test, bye"
  debug:
    msg: "bye, chandler"
  tags: "bye"

## 这个task和被执行hello role 没有关系
root@Dodo:/tmp/task# cat cat main.yaml
cat: cat: No such file or directory
- hosts: all
  gather_facts: no
  roles:
   - hello
  tags:
   - hil

## output
root@Dodo:/tmp/task# ansible-playbook --private-key /root/.ssh/id_rsa -i hosts_ip   main.yaml

PLAY [all] *******************************************************************************************************************

TASK [hello : test, hi] ******************************************************************************************************
ok: [127.0.0.1] => {
    "msg": "hi, chandler"
}

TASK [hello : test, bye] *****************************************************************************************************
ok: [127.0.0.1] => {
    "msg": "bye, chandler"
}

PLAY RECAP *******************************************************************************************************************
127.0.0.1                  : ok=2    changed=0    unreachable=0    failed=0

## use --tags
root@Dodo:/tmp/task# ansible-playbook --private-key /root/.ssh/id_rsa -i hosts_ip --tags="hi"  main.yaml

PLAY [all] *******************************************************************************************************************

TASK [hello : test, hi] ******************************************************************************************************
ok: [127.0.0.1] => {
    "msg": "hi, chandler"
}

PLAY RECAP *******************************************************************************************************************
127.0.0.1                  : ok=1    changed=0    unreachable=0    failed=0

```

You can apply the same tag to more than one individual task. This example tags several tasks with the same tag, “ntp”:

```yaml
---
# file: roles/common/tasks/main.yml

- name: Install ntp
  ansible.builtin.yum:
    name: ntp
    state: present
  tags: ntp

- name: Configure ntp
  ansible.builtin.template:
    src: ntp.conf.j2
    dest: /etc/ntp.conf
  notify:
  - restart ntpd
  tags: ntp

- name: Enable and run ntpd
  ansible.builtin.service:
    name: ntpd
    state: started
    enabled: yes
  tags: ntp

- name: Install NFS utils
  ansible.builtin.yum:
    name:
    - nfs-utils
    - nfs-util-lib
    state: present
  tags: filesharing
```

If you ran these four tasks in a playbook with `--tags ntp`, Ansible would run the three tasks tagged `ntp` and skip the one task that does not have that tag.



### Debug:  Print statements during execution

This module prints statements during execution and can be useful for debugging variables or expressions without necessarily halting the playbook.

#### msg

The customized message that is printed. If omitted, prints a generic message. 

**Default:**"Hello world!"

```yaml
- hosts: all
  tasks:
  - name: "test register"
    shell: grep 65535 /etc/security/limits.conf
    ignore_errors: True
    register: returnmsg2
  - name: "echo  register"
    debug: msg="{{returnmsg2.stdout}}"
    debug: msg="{{returnmsg2.cmd}}"

## output
## 第一个 msg="{{returnmsg2.stdout}}"被覆盖--
ok: [127.0.0.1] => {
    "msg": "grep 65535 /etc/security/limits.conf"
}

```



#### var

Mutually exclusive with the `msg` option.

```yaml
- hosts: all
  tasks:
  - name: "test register"
    shell: grep 65535 /etc/security/limits.conf
    ignore_errors: True
    register: returnmsg2
  - name: "echo  register"
    debug: var=returnmsg2.cmd
    debug: var=returnmsg2.stdout

## output
## 第一个var=returnmsg2.cmd，被覆盖
ok: [127.0.0.1] => {
    "returnmsg2.stdout": "root soft nofile 65535\nroot hard nofile 65535\n* soft nofile 65535\n* hard nofile 65535"
}

```

end

### with_items: list of items

`with_items`会隐式自动将list-in-list的sub-list items自动展开

使用`with_items`关键字，声明list，使用`item`关键字读取list items

气人的时，官方文档界面并没有明说。

#### list: simply returns what it is given

```yaml
# list
- hosts: all
  tasks:
  - name: "loop through list from a variable"
    debug:
      msg: "An item: {{ item }}"
    with_items:
    - 1
    - 3
    - 6
## output
ok: [127.0.0.1] => (item=None) => {
    "msg": "An item: 1"
}
ok: [127.0.0.1] => (item=None) => {
    "msg": "An item: 3"
}
ok: [127.0.0.1] => (item=None) => {
    "msg": "An item: 6"
}

## list-in-list
- hosts: all
  tasks:
  - name: get all items
    debug:
      msg: "{{ item }}"
    with_items:
    - 1
    - [2,3]
    - 4

## output
ok: [127.0.0.1] => (item=None) => {
    "msg": 1
}
ok: [127.0.0.1] => (item=None) => {
    "msg": 2
}
ok: [127.0.0.1] => (item=None) => {
    "msg": 3
}
ok: [127.0.0.1] => (item=None) => {
    "msg": 4
}

```

#### dict

```yaml
- hosts: all
  tasks:
  - name: "loop through list from a variable"
    debug:
      msg: "An item: {{ item }}"
    with_items:
    - 1
    - 3
    - 6
  - name: more complex items to add several users
    debug:
      msg: "Name: {{ item.name}} Group: {{ item.groups }}"
    with_items:
    - { name: testuser1, uid: 1002, groups: "wheel, staff" }
    - { name: testuser2, uid: 1003, groups: "staff" }
## output

ok: [127.0.0.1] => (item=None) => {
    "msg": "Name: testuser1 Group: wheel, staff"
}
ok: [127.0.0.1] => (item=None) => {
    "msg": "Name: testuser2 Group: staff"
}

```



### with_list： simply returns what it is given

当list中有sub-list时，显示`item`时，不会自动拆分

```yaml
- hosts: all
  tasks:
  - name: unlike with_items you will get 3 items from this loop, the 2nd one being a list
    debug:
      msg: "{{ item }}"
    with_list:
    - 1
    - [2,3]
    - 4

## output
ok: [127.0.0.1] => (item=None) => {
    "msg": 1
}
ok: [127.0.0.1] => (item=None) => {
    "msg": [
        2,
        3
    ]
}
ok: [127.0.0.1] => (item=None) => {
    "msg": 4
}

```

end

### loop: Migrating from with_X to loop

https://docs.ansible.com/ansible/latest/user_guide/playbooks_loops.html#migrating-from-with-x-to-loop

loop 更加同一和丰富

```yaml
## with_list and loop
- hosts: all
  tasks:
  - name: with_list
    debug:
      msg: "{{ item }}"
    with_list:
    - [2,3]
    - 8
  - name: with_list -> loop
    debug:
      msg: "{{ item }}"
    loop:
    - [2,3]
    - 8

## output
TASK [with_list] *************************************************************************************************************
ok: [127.0.0.1] => (item=None) => {
    "msg": [
        2,
        3
    ]
}
ok: [127.0.0.1] => (item=None) => {
    "msg": 8
}

TASK [with_list -> loop] *****************************************************************************************************
ok: [127.0.0.1] => (item=None) => {
    "msg": [
        2,
        3
    ]
}
ok: [127.0.0.1] => (item=None) => {
    "msg": 8
}
## with_item and loop
- hosts: all
  vars:
    items:
    - [2,3]
    - 8
  tasks:
  - name: with_list
    debug:
      msg: "{{ item }}"
    with_items: "{{ items }}"
  - name: with_items -> loop
    debug:
      msg: "{{ item }}"
    loop: "{{ items | flatten(levels=1)}}
    
## output

TASK [with_list] ***************************************************************
ok: [127.0.0.1] => (item=None) => {
    "msg": 2
}
ok: [127.0.0.1] => (item=None) => {
    "msg": 3
}
ok: [127.0.0.1] => (item=None) => {
    "msg": 8
}

TASK [with_items -> loop] ******************************************************
ok: [127.0.0.1] => (item=None) => {
    "msg": 2
}
ok: [127.0.0.1] => (item=None) => {
    "msg": 3
}
ok: [127.0.0.1] => (item=None) => {
    "msg": 8
}
```

end

#### dict2items

ansible 2.6 版本才行--

```yaml
- name: Using dict2items
  ansible.builtin.debug:
    msg: "{{ item.key }} - {{ item.value }}"
  loop: "{{ tag_data | dict2items }}"
  vars:
    tag_data:
      Environment: dev

```

end







### loop_control

“loop_control”关键字可以用于控制循环的行为

#### pause

pause选项能够让我们设置每次循环之后的暂停时间，以秒为单位，换句话说就是设置每次循环之间的间隔时间，示例如下

#### index_var

“index_var”是”loop_control”的一个设置选项，”index_var”的作用是让我们指定一个变量，”loop_control”会将列表元素的索引值存放到这个指定的变量中，比如如下配置

```yaml
  loop_control:
    index_var: my_idx
```

上述设置表示，在遍历列表时，当前被遍历元素的索引会被放置到”my_idx”变量中，也就是说，当进行循环操作时，只要获取到”my_idx”变量的值，就能获取到当前元素的索引值，当然，除了”index_var”选项，”loop_control”还有一些其他的选项可以使用，我们之后再进行总结。



#### label

label选项的作用，它可以在循环输出信息时，简化输出item的信息。

```yaml
- hosts: all
  remote_user: root
  gather_facts: no
  vars:
    testlist:
    - aaaaaaaaaaaaaaaaaaaaaaaaaaaa
    - [b,c]
    - daaaaaaaaaaaaaaaaaaaaaa
  tasks:
  - debug:
      msg: "{{oute}}--{{item}}"
    loop: "{{testlist | flatten(levels=1)}}"
    loop_control:x
      index_var: oute
      label: "test-labe
      
## 使用label 简化item输出
ok: [127.0.0.1] => (item=test-label) => {
    "msg": "0--aaaaaaaaaaaaaaaaaaaaaaaaaaaa",
    "oute": 0
}
ok: [127.0.0.1] => (item=test-label) => {
    "msg": "1--b",
    "oute": 1
}
ok: [127.0.0.1] => (item=test-label) => {
    "msg": "2--c",
    "oute": 2
}
## 不使用label，输出如下
ok: [127.0.0.1] => (item=aaaaaaaaaaaaaaaaaaaaaaaaaaaa) => {
    "msg": "0--aaaaaaaaaaaaaaaaaaaaaaaaaaaa",
    "oute": 0
}
ok: [127.0.0.1] => (item=b) => {
    "msg": "1--b",
    "oute": 1
}
ok: [127.0.0.1] => (item=c) => {
    "msg": "2--c",
    "oute": 2
}
ok: [127.0.0.1] => (item=daaaaaaaaaaaaaaaaaaaaaa) => {
    "msg": "3--daaaaaaaaaaaaaaaaaaaaaa",
    "oute": 3
}

```

end

#### loop_var：高级

从执行结果可以看出，”outer_item”变量中的值正是外层循环中item的值，这就是loop_var选项的作用。

当出现这种”双层循环”的情况时，则可以在外层循环中使用loop_var选项指定一个变量，这个变量可以用来代替外层循环中的item变量，以便在内层循环中获取到外层循环的item的值，从而避免了两层循环中”item”变量名的冲突

```yaml
# cat A.yml
---
- hosts: test70
  remote_user: root
  gather_facts: no
  tasks:
  - include: B.yml
    loop:
    - 1
    - 2
    loop_control:
      loop_var: outer_item
 
# cat B.yml
- debug:
    msg: "{{outer_item}}--{{item}}--task in B.yml"
  loop:
  - a
  - b
  - f
  
 ## output
 TASK [debug] *************************************************
ok: [test70] => (item=a) => {
    "msg": "1--a--task in B.yml"
}
ok: [test70] => (item=b) => {
    "msg": "1--b--task in B.yml"
}
ok: [test70] => (item=c) => {
    "msg": "1--c--task in B.yml"
}
 
TASK [debug] *************************************************
ok: [test70] => (item=a) => {
    "msg": "2--a--task in B.yml"
}
ok: [test70] => (item=b) => {
    "msg": "2--b--task in B.yml"
}
ok: [test70] => (item=c) => {
    "msg": "2--c--task in B.yml"
}
```

end

#### loop_var: 实际应用

```yaml
## 定义out_item
- name: 检查必要的crt和key存在否
  include_tasks: check.yml
  loop_control:
    loop_var: out_item
  loop: "{{ files }}"
  when: ( redo | default(false) ) 
  connection: local
  run_once: true
## 使用 out_item
- stat: path={{ out_item }}
  register: need_file

- fail: msg=当前主机缺失文件{{ out_item }}
  when: not need_file.stat.exists
  
```

end

### when

The `when` clause is a raw Jinja2 expression without double curly braces . 

判断是否执行task 

```yaml
- name: 停止所有节点kubelet
  systemd: name=etcd state=stopped
  when: inventory_hostname in groups['Allnode'] and ( redoAll | default(false) )  # 换vip或者所有ip的时候带上-e redoAll=true才会执行

- name: 停止etcd
  systemd: name=etcd state=stopped
  when: inventory_hostname in groups['etcd']
```

end

### Return Value

https://docs.ansible.com/ansible/latest/reference_appendices/common_return_values.html

#### Common return value

ansible执行task的返回信息

```yaml
ok: [127.0.0.1] => {
    "msg": {
        "changed": true,
        "cmd": "grep 1024000 /etc/security/limits.conf",
        "delta": "0:00:00.032354",
        "end": "2022-02-28 09:07:53.172157",
        "failed": true,
        "msg": "non-zero return code",
        "rc": 1,
        "start": "2022-02-28 09:07:53.139803",
        "stderr": "",
        "stderr_lines": [],
        "stdout": "",
        "stdout_lines": []
    }
}

```

字段解析：

* changed：A boolean indicating if the task had to make changes to the target or delegated host.
* cmd： The command executed by the task.
* delta： The command execution delta time.
* failed：A boolean that indicates if the task was failed or not.
* msg：A string with a generic message relayed to the user.
* rc：Some modules execute command line utilities or are geared for executing commands directly (raw, shell, command, and so on), this field contains ‘return code’ of these utilities.

利用返回信息，作判断处理逻辑。

```yaml
---
- hosts: all
  gather_facts: no
  remote_user: root
  
  tasks:
    - name: "limits.conf info"
      shell: grep 1024000 /etc/security/limits.conf
      ignore_errors: True
      register: returnmsg2  
    ...

    - name: config security-limits.conf
      shell:
        cmd: |
          cat << EOF >> /etc/security/limits.conf
          * hard nofile 1024000
          * soft nofile 1024000
          * soft nproc 1024000
          * hard nproc 1024000
          EOF
      when: returnmsg2.rc != 0

```

end

### role

Roles 的概念来自于这样的想法：通过 include 包含文件并将它们组合在一起，组织成一个简洁、可重用的抽象对象。这种方式可使你将注意力更多地放在大局上，只有在需要时才去深入了解细节。

laybook 适合中小项目，而大项目一定要是有 Roles。Roles 不仅支持 Tasks 的集合，同时包括 var_files、tasks、handlers、meta、templates等

最简单的role

```bash
root@Dodo:/tmp/task/roles/hello# tree /tmp/task/roles/
/tmp/task/roles/
└── hello
    └── tasks
        └── main.yaml

2 directories, 1 file

```

复杂结构的roles应用，目录结构。

```bash
$ tree ansible_playbooks/
ansible_playbooks/
└── roles  必须叫roles
    ├── dbsrvs -------------role1_name
    │   ├── defaults  ---------必须存在的目录，存放默认的变量，模板文件中的变量就是引用自这里。defaults中的变量优先级最低，通常我们可以临时指定变量来进行覆盖
    │   │   └── main.yml
    │   ├── files -------------ansible中unarchive、copy等模块会自动来这里找文件，从而我们不必写绝对路径，只需写文件名
    │   │   ├── mysql.tar.gz
    │   │   └── nginx.tar.gz
    │   ├── handlers -----------存放tasks中的notify指定的内容
    │   │   └── main.yml
    │   ├── meta
    │   ├── tasks --------------存放playbook的目录，其中main.yml是主入口文件，在main.yml中导入其他yml文件，要采用import_tasks关键字，include要弃用了
    │   │   ├── install.yml
    │   │   └── main.yml -------主入口文件
    │   ├── templates ----------存放模板文件。template模块会将模板文件中的变量替换为实际值，然后覆盖到客户机指定路径上
    │   │   └── nginx.conf.j2
    │   └── vars
    └── websrvs -------------role2名称
        ├── defaults
        │   └── main.yml
        ├── files
        │   ├── mysql.tar.gz
        │   └── nginx.tar.gz
        ├── handlers
        │   └── main.yml
        
        ├── meta
        ├── tasks
        │   ├── install.yml
        │   └── main.yml
        ├── templates
        │   └── nginx.conf.j2
        └── vars
```

end

#### role and tags

```yaml
## 简单
- hosts: "{{ run | default('all')}}"
  remote_user: root #以root身份执行
  gather_facts: no
  roles:
    - influxdb
  tags:
    - influxdb


## 复杂
- hosts: master
  gather_facts: yes
  roles:
    - { role: HA, tags: HA, when: "groups['master'] | length  > 1" }
```

end

### include and include_tasks

最初的`include`可以引用其他`yaml file`中的task。

```yaml
## main yaml
# cat t1.yaml
- hosts: all
  remote_user: root
  gather_facts: no
  vars:
    testlist:
    - aaaaaaaaaaaaaaaaaaaaaaaaaaaa
    - [b,c]
    - daaaaaaaaaaaaaaaaaaaaaa
  tasks:
  - include: t3.yaml

## task yaml
# cat t3.yaml
- debug:
    msg: "{{oute}}--{{item}}"
  loop: "{{testlist | flatten(levels=1)}}"
  loop_control:
    index_var: oute

```

当我们需要包含一个任务列表时，”include_tasks”关键字的用法与”include”完全相同。

```yaml
# cat intest.yml
---
- hosts: test70
  remote_user: root
  gather_facts: no
  tasks:
  - debug:
      msg: "test task1"
  - include_tasks: in.yml
  - debug:
      msg: "test task2"
 
# cat in.yml
- debug:
    msg: "task1 in in.yml"
- debug:
    msg: "task2 in in.yml"
```

end

### Block

block"关键字将多个任务整合成一个"块"，这个"块"将被当做一个整体，我们可以对这个"块"添加判断条件，当条件成立时，则执行这个块中的所有任务。

```yaml
- hosts: all
  gather_facts: no
  tasks:
  - debug:
      msg: "task1 not in block"
  - block:
      - debug:
          msg: "task2 in block1"
      - debug:
          msg: "task3 in block2"
    when: true

## output

TASK [debug] *****************************************************************************************************************
ok: [127.0.0.1] => {
    "msg": "task1 not in block"
}

TASK [debug] *****************************************************************************************************************
ok: [127.0.0.1] => {
    "msg": "task2 in block1"
}

TASK [debug] *****************************************************************************************************************
ok: [127.0.0.1] => {
    "msg": "task3 in block2"
}

```

end

#### rescue and always

```yaml
- hosts: all
  gather_facts: no
  tasks:
  - block:
      - debug:
          msg: "{{ test_var }}"
      # it can not echo because test_var is not define.
      - debug:
          msg: " test_var ? "
    rescue:
      - debug:
          msg: "not define test_var"
    always:
      - debug:
          msg: "byebye"
## output

TASK [debug] *******************************************************************
## 由于有fatal error，导致"test_var ?" 没有执行
fatal: [127.0.0.1]: FAILED! => {"msg": "The task includes an option with an undefined variable. The error was: 'test_var' is undefined\n\nThe error appears to have been in ...

TASK [debug] *******************************************************************
ok: [127.0.0.1] => {
    "msg": "not define test_var"
}

TASK [debug] *******************************************************************
ok: [127.0.0.1] => {
    "msg": "byebye"
}

```

end

### default

You can **provide default values** for variables directly in your templates using the Jinja2 ‘default’ filter. This is often a better approach than failing if a variable is not defined:

```yaml
{{ some_variable | default(5) }}
```

In the above example, if the variable ‘some_variable’ is not defined, Ansible uses the default value 5, rather than raising an “undefined variable” error and failing. If you are working within a role, you can also add a `defaults/main.yml` to define the default values for variables in your role



#### `|default(false) and undefined`

核心： 利用`|` 和 `false or 0`，判断变量是否定义。

There is a very special construction in Jinja2/Ansible: ‘`foo is defined`’, which allows you to check if foo is existing at all. It’s usually used in this form:

```yaml
...
when: foo is defined and foo == True
```

If we omit ‘is defined’, and foo is not defined, condition ‘`foo == True` ‘ will fail. It wouldn’t become just ‘False’, it will cause ansible to stop executing playbook with error message. So, most people use ‘is defined’ again and again. When conditions become **complicated**, it cause pain.

```yaml
when: foo is defined and foo == True or (bar is defined and bar != True)
```

After some suffering I realized, there is a better way.

Jinja2 have a very special filter: ‘default’, which allows to provide a value if variable is not defined.

```yaml
when: foo|default(False) == True or bar|default(False) == True 
```

**The use of ‘default’ filter allows to eliminate use of ‘is defined’ and simplifies playbooks a lot**. In roles you can use role/defaults/main.yaml to store default values for all variables, but for playbooks there is such option. Use ‘|default’ filter for that.

验证栗子：

因为变量没有定义且`|`的默认值也为`0 or false`，所以不满足`when`条件，无法输出。

```yaml
- hosts: all
  remote_user: root
  gather_facts: no
  vars:
    testlist:
    - aaaaaaaaaaaaaaaaaaaaaaaaaaaa
    - [b,c]
    - daaaaaaaaaaaaaaaaaaaaaa
  tasks:
  - include: t3.yaml
  
## cat t3.yaml
- debug:
    msg: "{{oute}}--{{item}}"
  loop: "{{testlist | flatten(levels=1)}}"
  loop_control:
    index_var: oute
    label: "test-label"
  # ( undefine_var | default(false) )
  when:  ( undefine_var | default(0) )

## output
skipping: [127.0.0.1] => (item=test-label)
skipping: [127.0.0.1] => (item=test-label)
skipping: [127.0.0.1] => (item=test-label)
skipping: [127.0.0.1] => (item=test-label)
```

end
