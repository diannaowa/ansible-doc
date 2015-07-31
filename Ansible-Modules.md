# Ansible Modules #
---

@author 刘振伟

@QQ 570962906

Ansible拥有大量的模块(module library)来操作远程主机节点或者通过playbook对节点操作。

用户也可以开发自己的模块，这些模块可以控制系统资源，如系统服务，包管理，文件操作或者执行系统命令等等。

Ansible中每个模块都会完成特定的任务。在每个playbook任务定义的任务都会依次得到执行，当然也可以指定值运行某个特定的操作。

    ansible webservers -m service -a "name=httpd state=started"
    ansible webservers -m ping
    ansible webservers -m command -a "/sbin/reboot -t now" 

如上边第一个命令是使用service模块在webservres组内的所有主机节点上启动httpd服务，-m指定模块名称，-a指定模块所需的参数，
`key=value`方式指定模块参数,多个参数以空格隔开。有些模块是不需要带参数的。command或者shell模块则需要指定一个字符串参数，表示要运行的命令。

Ansible通过playbook中所定义的操作以同样的方式执行每个模块：

    - name: reboot the servers
      action: command /sbin/reboot -t now

也可以以下边的方式定义：

    - name: reboot the servers
      command: /sbin/reboot -t now

给模块传递参数：

    - name: restart webserver
      service:
    	name: httpd
    	state: restarted


所有的模块包括自己开发的模块都会或者应该返回一个JSON格式的数据。使用playbook的时候，某一个任务可以触发另一个任务的执行。

查看某个某块的文档：

    ansible-doc yum

列出所有已经安装的模块：

    ansible-doc -l


**核心模块**

Ansible会维护一些较为核心的模块，这些模块托管在github上 [ansible-modules-core](https://github.com/ansible/ansible-modules-core).


**附加模块**

这些模块目前随着Ansible安装使用，但以后可能也会独立安装。这些模块大多数由社区维护。非核心模块功能也是较为齐全的，但是其维护及相应效率有可能没有核心模块那么高。
附加模块也是托管在github上的 [ansible-modules-extras](http://github.com/ansible/ansible-modules-extras)。


**返回值**

Ansible模块通常会返回一个有一定格式的数据（JSON格式的数据），可以将其赋值给一个变量，也可以直接输出到标准输出上。

*行为(Facts)*

一些模块会给Ansible返回一个`facts`，通过`ansible_facts`，当前主机可以将其直接作为一个变量使用。

*状态(status)*

每个模块都必须返回一个状态，来说明模块是否执行成功，或者是说明操作对象是否发生变化（如上传文件到远程机，根据状态可以判断该文件是否修改过，只有修改过的文件Ansible才会重新上传覆盖）

*其他返回信息*

通常，模块还会返回成功或者失败的信息。一些模块会直接返回标准输出和标准错误信息，如command和shell模块。如果Ansible见到标准输出信息，会将其追加到stdou_lines列表中，并打印出来。

## 介绍几个较为常用的模块 ##

**Commands Modules(命令模块)**

*`command` --- 在远程节点上执行命令*

Command模块后边紧跟这要执行的命令，命令的参数以空格隔开。指定的命令会在所选的所有的节点上执行。命令并不是通过shell执行的，所以并不能使用`$HOME`等环境变量和一些操作符`(<,>,|,&)`。shell模块支持环境变量和操作符。

 选项：

`chdir` 在运行命令之前，先切换到指定的目录

    [root@web1 ~]# ansible webservers -m command -a "ls -l chdir=/tmp"
    192.168.1.65 | success | rc=0 >>
    total 68
    -rw-r--r--. 1 root   root   61530 Jul 13 15:59 5kcrm.sql
    srwx------. 1 mongod mongod 0 Jul 22 06:28 mongodb-27017.sock
    -rw-r--r--. 1 root   root   0 Jul 22 11:59 test
    drwx------. 2 root   root4096 Jul 21 22:49 vmware-root
上边的命令，先切换到/tmp目录下，然后执行`ls -l` 命令。

`creates` 后边可以直接指定一个文件名(目录名)，或者是以正则模式匹配一系列文件名。如果所指定的文件存在的话，则不执行指定的命令。

    [root@web1 ~]# ansible webservers -m command -a "ls -l /tmp creates=/tmp/test"
    192.168.1.65 | success | rc=0 >>
    skipped, since /tmp/test exists
    
    [root@web1 ~]# ansible webservers -m command -a "ls -l /tmp creates=/tmp/test2"
    192.168.1.65 | success | rc=0 >>
    total 68
    -rw-r--r--. 1 root   root   61530 Jul 13 15:59 5kcrm.sql
    srwx------. 1 mongod mongod 0 Jul 22 06:28 mongodb-27017.sock
    -rw-r--r--. 1 root   root   0 Jul 22 11:59 test
    drwx------. 2 root   root4096 Jul 21 22:49 vmware-root

上边的命令，如果`/tmp/test`存在的话，则不执行`ls -l /tmp`命令。`/tmp/test2`不存在，执行了`ls -l /tmp`命令。示例中的test和test2可以是目录也可以是文件。

`removes` 后边可以直接指定一个文件名(或目录名)，或者是以正则模式匹配一系列文件名。如果所指定的文件不存在的话，则不运行命令。

    [root@web1 ~]# ansible webservers -m command -a "ls -l /tmp removes=/tmp/testdir2"
    192.168.1.65 | success | rc=0 >>
    skipped, since /tmp/testdir2 does not exist
    
    [root@web1 ~]# ansible webservers -m command -a "ls -l /tmp removes=/tmp/testdir"
    192.168.1.65 | success | rc=0 >>
    total 72
    -rw-r--r--. 1 root   root   61530 Jul 13 15:59 5kcrm.sql
    srwx------. 1 mongod mongod 0 Jul 22 06:28 mongodb-27017.sock
    -rw-r--r--. 1 root   root   0 Jul 22 11:59 test
    drwxr-xr-x. 2 root   root4096 Jul 22 12:03 testdir
    drwx------. 2 root   root4096 Jul 21 22:49 vmware-root

/tmp/testdir2不存在则不执行指定的命令。/tmp/testdir存在则运行命令。

`warn` 这个是在1.8版本中新增的选项。如果再ansible.cfg配置文件中开启了警告信息，可以只用此选项对当前命令关闭警告，默认为True。

Playbooks示例:

    # Example from Ansible Playbooks.
    - command: /sbin/shutdown -t now
    
    # Run the command if the specified file does not exist.
    - command: /usr/bin/make_database.sh arg1 arg2 creates=/path/to/database
    
    # You can also use the 'args' form to provide the options. This command
    # will change the working directory to somedir/ and will only run when
    # /path/to/database doesn't exist.
    - command: /usr/bin/make_database.sh arg1 arg2
      args:
    chdir: somedir/
    creates: /path/to/database


    [root@web1 ~]# cat /etc/ansible/playbook.yml
    ---
    - hosts: webservers
      remote_user: root
      tasks:
      - name: ls -l /tmp
    command: /bin/ls -l /tmp removes=/tmp/testdir
    [root@web1 ~]# ansible-playbook /etc/ansible/playbook.yml 
    
    PLAY [webservers] ************************************************************* 
    
    GATHERING FACTS *************************************************************** 
    ok: [192.168.1.65]
    
    TASK: [ls -l /tmp] ************************************************************ 
    changed: [192.168.1.65]
    
    PLAY RECAP ******************************************************************** 
    192.168.1.65   : ok=2changed=1unreachable=0failed=0



*`script` 在远程机上运行本地脚本*

`script`模块的-a选项直接跟一个本地脚本的绝对路径，脚本的参数以空格隔开。该模块首先将指定的脚本传到远程节点上，然后在远程节点的shell环境下执行该脚本。

    [root@web1 ~]# cat /root/test.sh 
    #!/bin/bash
    
    echo $1
    [root@web1 ~]# ansible webservers -m script -a "/root/test.sh 12"
    192.168.1.65 | success >> {
    "changed": true, 
    "rc": 0, 
    "stderr": "", 
    "stdout": "12\n"
    }


creates和removes参数与command模块的参数类似：

    [root@web1 ~]# ansible webservers -m script -a "/root/test.sh 12 creates=/tmp/test"
    192.168.1.65 | success >> {
    "changed": false, 
    "msg": "skipped, since /tmp/test exists"
    }
    
    [root@web1 ~]# ansible webservers -m script -a "/root/test.sh 12 creates=/tmp/test2"
    192.168.1.65 | success >> {
    "changed": true, 
    "rc": 0, 
    "stderr": "", 
    "stdout": "12\n"
    }

    [root@web1 ~]# ansible webservers -m script -a "/root/test.sh 12 removes=/tmp/test2"
    192.168.1.65 | success >> {
    "changed": false, 
    "msg": "skipped, since /tmp/test2 does not exist"
    }
    
    [root@web1 ~]# ansible webservers -m script -a "/root/test.sh 12 removes=/tmp/test"
    192.168.1.65 | success >> {
    "changed": true, 
    "rc": 0, 
    "stderr": "", 
    "stdout": "12\n"
    }

Playbooks示例：-v参数显示详细信息

    [root@web1 ~]# cat /etc/ansible/playbook.yml 
    ---
    - hosts: webservers
      remote_user: root
    
      tasks:
    #  - name: ls -l /tmp
    #    command: /bin/ls -l /tmp removes=/tmp/testdir
       - name: test script module
         script: /root/test.sh test
    [root@web1 ~]# ansible-playbook /etc/ansible/playbook.yml -v
    
    PLAY [webservers] ************************************************************* 
    
    GATHERING FACTS *************************************************************** 
    ok: [192.168.1.65]
    
    TASK: [test script module] **************************************************** 
    changed: [192.168.1.65] => {"changed": true, "rc": 0, "stderr": "", "stdout": "test\n"}
    
    PLAY RECAP ******************************************************************** 
    192.168.1.65   : ok=2changed=1unreachable=0failed=0

通常情况都是将某个功能开发成一个Ansible的模块来使用，而并不是像上边那样在远程节点上运行本地的脚本。



*`shell` 在远程节点执行命令*

`shell`模块的参数为命令名称，命令本身的参数以空格隔开。像`command`模块那样在远程节点执行命令，但`shell`模块再远程节点是通过shell环境(/bin/bash)执行命令的,该模块也可以执行一个shell脚本，但该脚本必须在远程节点上存在。

`chdir`、`creates`、`removes`参数与`command`模块的参数一样。

    [root@web1 ~]# ansible webservers -m shell -a "echo $HOME"
    192.168.1.65 | success | rc=0 >>
    /root
    
    [root@web1 ~]# ansible webservers -m shell -a "echo $HOME removes=/tmp/test"
    192.168.1.65 | success | rc=0 >>
    /root
    
    [root@web1 ~]# ansible webservers -m shell -a "/root/remote.sh hello removes=/tmp/test"
    192.168.1.65 | success | rc=0 >>
    I am on remote server:hello

上边执行的`/root/remote.sh`要在远程节点上存在。


PlayBooks示例：


    [root@web1 ~]# cat /etc/ansible/shell.yml 
    ---
    - hosts: webservers
      remote_user: root
    
      tasks:
       - name: shell module 1
    #file on the remote
         shell: /root/remote.sh world
       - name: shell module 2
         shell: ls -l
         args:
          chdir: /tmp/testdir/
    [root@web1 ~]# ansible-playbook /etc/ansible/shell.yml  -v
    
    PLAY [webservers] ************************************************************* 
    
    GATHERING FACTS *************************************************************** 
    ok: [192.168.1.65]
    
    TASK: [shell module 1] ******************************************************** 
    changed: [192.168.1.65] => {"changed": true, "cmd": "/root/remote.sh world", "delta": "0:00:00.014310", "end": "2015-07-22 14:17:29.724289", "rc": 0, "start": "2015-07-22 14:17:29.709979", "stderr": "", "stdout": "I am on remote server:world", "warnings": []}
    
    TASK: [shell module 2] ******************************************************** 
    changed: [192.168.1.65] => {"changed": true, "cmd": "ls -l", "delta": "0:00:00.018538", "end": "2015-07-22 14:17:31.158544", "rc": 0, "start": "2015-07-22 14:17:31.140006", "stderr": "", "stdout": "total 0\n-rw-r--r--. 1 root root 0 Jul 22 14:15 testfile", "warnings": []}
    
    PLAY RECAP ******************************************************************** 
    192.168.1.65   : ok=3changed=2unreachable=0failed=0 


[官方文档](http://docs.ansible.com/ansible/list_of_commands_modules.html)

**文件相关模块(Files Modules)**

[官方文档](http://docs.ansible.com/ansible/list_of_files_modules.html)

*`copy` 复制本地文件到远程路径下*

`copy`模块将本地文件复制到远程路径下。`fetch`模块将远程文件复制到本地。

`copy`的选项：

`dest` 必选参数，为目标文件指定远程节点上的一个绝对路径。如果`src`是一个目录，那么该参数也必须是个目录。


`src` 本地文件的绝对路径，或者相对路径。如果是个路径则会递归复制，路径是以`/`结尾的话，只复制目录里面的内容，如果不以`/`几位的话会复制目录本身和里面的内容。类似Rsync那样。


`backup`  可选参数，为源文件创建一个备份文件，被给备份文件添加一个时间戳信息。值为：yes/no，默认为no。

    [root@web1 ~]# ansible webservers -m copy -a "src=/root/link/test.sh dest=/root/"
    192.168.1.65 | success >> {
    "changed": true, 
    "checksum": "946786e8a3f5f48367c117896960a3516a0d183c", 
    "dest": "/root/test.sh", 
    "gid": 0, 
    "group": "root", 
    "md5sum": "8d2fb7913d42f3e38ff61ea47bcbc8d0", 
    "mode": "0644", 
    "owner": "root", 
    "secontext": "system_u:object_r:admin_home_t:s0", 
    "size": 21, 
    "src": "/root/.ansible/tmp/ansible-tmp-1437548370.02-201274541353834/source", 
    "state": "file", 
    "uid": 0
    }
    
    [root@web1 ~]# echo "echo 1">>/root/link/test.sh 
    [root@web1 ~]# ansible webservers -m copy -a "src=/root/link/test.sh dest=/root/ backup=yes"
    192.168.1.65 | success >> {
    "backup_file": "/root/test.sh.2015-07-22@15:00:11~", 
    "changed": true, 
    "checksum": "578133b59b8576279740953cf87cea8d10404120", 
    "dest": "/root/test.sh", 
    "gid": 0, 
    "group": "root", 
    "md5sum": "a018eb673250b68bced72b681ee75128", 
    "mode": "0644", 
    "owner": "root", 
    "secontext": "system_u:object_r:admin_home_t:s0", 
    "size": 28, 
    "src": "/root/.ansible/tmp/ansible-tmp-1437548412.04-34909014140112/source", 
    "state": "file", 
    "uid": 0
    }

由最后一条命令看出，使用了`backup=yes`，会自动备份旧的文件。


`content` 可选参数，当使用该参数来代替`src`的时候，会将内容直接写入到目标文件中。但，要是要是涉及到较为复杂的内容替换，推荐使用`template`模块。

    [root@web1 ~]# ansible webservers -m copy -a "content='test test' dest=/root/content.txt"
    192.168.1.65 | success >> {
    "changed": true, 
    "checksum": "abedc47a5ede3fab13390898c5160ec9afbb6ec3", 
    "dest": "/root/content.txt", 
    "gid": 0, 
    "group": "root", 
    "md5sum": "4f4acc5d8c71f5fbf04dace00b5360c8", 
    "mode": "0644", 
    "owner": "root", 
    "secontext": "system_u:object_r:admin_home_t:s0", 
    "size": 9, 
    "src": "/root/.ansible/tmp/ansible-tmp-1437548581.81-7002576101851/source", 
    "state": "file", 
    "uid": 0
    }
    ##远程机
    [root@db2 ~]# cat content.txt 
    test test[root@db2 ~]# 

`directory_mode` 可选参数，当递归复制的时候，为所创建的目录设置权限，如果没有指定则使用系统的默认权限。该参数只影响新创建的目录，不会影响已经存在的目录。

    [root@web1 ~]# tree /root/test/
    /root/test/
    └── test1
    └── test2
    └── testfile
    
    2 directories, 1 file
    [root@web1 ~]# ansible webservers -m copy -a "src=/root/test/ dest=/root/ directory_mode=0777"
    192.168.1.65 | success >> {
    "changed": true, 
    "checksum": "4e1243bd22c66e76c2ba9eddc1f91394e57f9f83", 
    "dest": "/root/test1/test2/testfile", 
    "gid": 0, 
    "group": "root", 
    "md5sum": "d8e8fca2dc0f896fd7cb4cb0031ba249", 
    "mode": "0644", 
    "owner": "root", 
    "secontext": "system_u:object_r:admin_home_t:s0", 
    "size": 5, 
    "src": "/root/.ansible/tmp/ansible-tmp-1437548879.2-36869348375941/source", 
    "state": "file", 
    "uid": 0
    }
    #远程机
    [root@db2 ~]# ll /root/test1/
    total 4
    drwxrwxrwx. 2 root root 4096 Jul 22 15:07 test2

`force`,该参数默认值为yes,当远程文件与本地文件内容不一致的时候会替换远程文件。只有当远程目标文件不存在的时候才会传输文件。

`group` 文件或目录的所属组.

    [root@web1 ~]# ansible webservers -m copy -a "src=/root/test/testgroup dest=/root/test1/testgroup group=liuzhenwei"
    192.168.1.65 | success >> {
    "changed": true, 
    "checksum": "ce3af7760f1289d02bf6a7ad19f3214c4e5c7c2e", 
    "dest": "/root/test1/testgroup", 
    "gid": 500, 
    "group": "liuzhenwei", 
    "md5sum": "102f5037fe6474019fe947b4977bb2a5", 
    "mode": "0644", 
    "owner": "root", 
    "secontext": "system_u:object_r:admin_home_t:s0", 
    "size": 3, 
    "src": "/root/.ansible/tmp/ansible-tmp-1437549245.05-39690928376614/source", 
    "state": "file", 
    "uid": 0
    }
    #远程机
    [root@db2 ~]# ll /root/test1/testgroup 
    -rw-r--r--. 1 root liuzhenwei 3 Jul 22 15:14 /root/test1/testgroup
    [root@db2 ~]#


`mode` 文件或目录的权限，如0644。

    [root@web1 ~]# echo "xxxxx" >/root/test/testgroup
    [root@web1 ~]# ansible webservers -m copy -a "src=/root/test/testgroup dest=/root/test1/testgroup mode=0755"
    192.168.1.65 | success >> {
    "changed": true, 
    "checksum": "db83d64e545efe62240d994937f1d50308aad177", 
    "dest": "/root/test1/testgroup", 
    "gid": 500, 
    "group": "liuzhenwei", 
    "md5sum": "02de18d4addc78e6b963c729de8bc0b6", 
    "mode": "0755", 
    "owner": "root", 
    "secontext": "system_u:object_r:admin_home_t:s0", 
    "size": 6, 
    "src": "/root/.ansible/tmp/ansible-tmp-1437549375.37-6560761620894/source", 
    "state": "file", 
    "uid": 0
    }
    
    [root@db2 ~]# ll /root/test1/testgroup 
    -rwxr-xr-x. 1 root liuzhenwei 6 Jul 22 15:16 /root/test1/testgroup

上边命令可以看出，当文件已经存在的时候，内容不一致，则只会替换里面的内容而不会重新传输本文件。

`owner` 文件或目录的所有者。

Playbooks示例：


    [root@web1 ~]# ansible-playbook /etc/ansible/copy.yml 
    
    PLAY [webservers] ************************************************************* 
    
    GATHERING FACTS *************************************************************** 
    ok: [192.168.1.65]
    
    TASK: [copy file] ************************************************************* 
    changed: [192.168.1.65]
    
    PLAY RECAP ******************************************************************** 
    192.168.1.65   : ok=2changed=1unreachable=0failed=0   
    
    [root@web1 ~]# cat /etc/ansible/copy.yml
    ---
    - hosts: webservers
      remote_user: root
      tasks:
       - name: copy file
         copy: src=/tmp/testfile dest=/tmp/testfile mode=0644
    
	#远程节点
    [root@db2 ~]# ll /tmp/testfile 
    -rw-r--r--. 1 root root 9 Jul 22 15:22 /tmp/testfile

*返回值*

src 要复制到远程节点的源文件路径

backup_file 远程节点上的备份文件路径，backup=yes的时候才有

uid 文件的所有者ID

dest 目标文件在远程节点上的绝对路径,/root/file.txt

checksum 校验值

md5sum md5校验值

state 状态，如 file

gid 文件的所属组ID

mode 文件的权限

owner 文件所有者

group 文件所属组

size 文件大小

[本模块的官方文档](http://docs.ansible.com/ansible/copy_module.html)

注：当复制大量的文件的时候，`copy`模块并是最好的选择，这个时候可以使用`synchronize`模块


*`synchronize`  使用rsync来同步文件*

该模块事实上是封装了rsync。在实际使用中用户也可以直接使用rsync来传输文件，但是这样的话 就需要指定大量的参数和节点facts。也可以使用`command`和`shell`模块来执行rsync命令。`synchronize`模块使得调用`rsync`更加方便快捷，该模块并不支持rsync的所有功能。

选项：

`archive` 表示以归档模式传输，默认为yes,保持文件的所有属性，如权限，时间戳信息，所属组和所有者等信息。


`compress` 默认为yes,压缩传输文件。

`dest` 文件在远程节点上的存放路径，可以是绝对路径也可以是相对路径。

`src` 源文件的路径，可以是绝对路径也可以是相对路径。

`copy_links` 默认为no,当为yes的时候表示，如果源文件是传输的一个符号链接，则直接将符号链接所指向的目标文件的内容传输到远程节点，而不是传输符号链接本身。

    [root@web1 test]# ll
    total 8
    drwxr-xr-x. 3 root root 4096 Jul 22 15:05 test1
    -rw-r--r--. 1 root root6 Jul 22 15:16 testgroup
    lrwxrwxrwx. 1 root root9 Jul 23 10:02 testlink -> testgroup
    [root@web1 test]# ansible webservers -m synchronize -a "src=/root/test/testlink dest=/root/test copy_links=yes"
    192.168.1.65 | success >> {
    "changed": true, 
    "cmd": "rsync --delay-updates -F --compress --archive --copy-links --rsh 'ssh  -S none -o StrictHostKeyChecking=no' --out-format='<<CHANGED>>%i %n%L' \"/root/test/testlink\" \"root@192.168.1.65:/root/test\"", 
    "msg": "<f+++++++++ testlink\n", 
    "rc": 0, 
    "stdout_lines": [
    "<f+++++++++ testlink"
    ]
    }
    #远程节点
    [root@db2 ~]# ll test/
    total 4
    -rw-r--r--. 1 root root 6 Jul 22 15:16 testlink
    [root@db2 ~]# cat test/
    cat: test/: Is a directory
    [root@db2 ~]# cat test/testlink 
    xxxxx

    #默认情况下copy_links为no,表示复制符号链接本身
    
    [root@web1 test]# ansible webservers -m synchronize -a "src=/root/test/testlink dest=/root/test copy_links=no"
    192.168.1.65 | success >> {
    "changed": true, 
    "cmd": "rsync --delay-updates -F --compress --archive --rsh 'ssh  -S none -o StrictHostKeyChecking=no' --out-format='<<CHANGED>>%i %n%L' \"/root/test/testlink\" \"root@192.168.1.65:/root/test\"", 
    "msg": "cL+++++++++ testlink -> testgroup\n", 
    "rc": 0, 
    "stdout_lines": [
    "cL+++++++++ testlink -> testgroup"
    ]
    }
    #远程节点
    [root@db2 ~]# ll test
    total 0
    lrwxrwxrwx. 1 root root 9 Jul 23 10:02 testlink -> testgroup


`delete` 在传输文件之后，删除远程节点上在`src`中不存在的文件,该参数需要`recursive=yes`

`recursive` 是否递归传输。该参数的默认值就是`archive`的值。

    [root@web1 ~]# ansible webservers -m synchronize -a "src=/root/test/ dest=/root/test/ delete=yes"
    192.168.1.65 | success >> {
    "changed": true, 
    "cmd": "rsync --delay-updates -F --compress --delete-after --archive --rsh 'ssh  -S none -o StrictHostKeyChecking=no' --out-format='<<CHANGED>>%i %n%L' \"/root/test/\" \"root@192.168.1.65:/root/test/\"", 
    "msg": ".d..t...... ./\n<f+++++++++ testgroup\ncd+++++++++ test1/\ncd+++++++++ test1/test2/\n<f+++++++++ test1/test2/testfile\n*deleting   testlink\n", 
    "rc": 0, 
    "stdout_lines": [
    ".d..t...... ./", 
    "<f+++++++++ testgroup", 
    "cd+++++++++ test1/", 
    "cd+++++++++ test1/test2/", 
    "<f+++++++++ test1/test2/testfile", 
    "*deleting   testlink"
    ]
    }

`links` 是否复制符号链接信息，默认值与`archive`参数的值一致。

`mode` 传输的模式。默认为`push`,指将本地文件传输到远程节点上。`pull`是指将远程文件拉到本地。

`owner` 维护所有者信息，默认值与`archive`设定的值相同,即默认为yes。以root用户传输文件的时候该参数才有起作用。

    [root@web1 ~]# ansible webservers -m synchronize -a "src=/root/test/owner dest=/tmp" -u liuzhenwei
    liuzhenwei@192.168.1.65's password: 
    192.168.1.65 | success >> {
    "changed": true, 
    "cmd": "rsync --delay-updates -F --compress --archive --rsh 'ssh  -S none -o StrictHostKeyChecking=no' --out-format='<<CHANGED>>%i %n%L' \"/root/test/owner\" \"liuzhenwei@192.168.1.65:/tmp\"", 
    "msg": "<f+++++++++ owner\n", 
    "rc": 0, 
    "stdout_lines": [
    "<f+++++++++ owner"
    ]
    }
    
    #远程节点
    [root@db2 ~]# ll /tmp/owner 
    -rw-r--r--. 1 liuzhenwei liuzhenwei 0 Jul 23 10:24 /tmp/owner

    ####################################

    [root@web1 ~]# ansible webservers -m synchronize -a "src=/root/test/owner dest=/tmp" -u root
    192.168.1.65 | success >> {
    "changed": true, 
    "cmd": "rsync --delay-updates -F --compress --archive --rsh 'ssh  -S none -o StrictHostKeyChecking=no' --out-format='<<CHANGED>>%i %n%L' \"/root/test/owner\" \"root@192.168.1.65:/tmp\"", 
    "msg": ".f....og... owner\n", 
    "rc": 0, 
    "stdout_lines": [
    ".f....og... owner"
    ]
    }
    #远程节点
    [root@db2 ~]# ll /tmp/owner 
    -rw-r--r--. 1 root root 0 Jul 23 10:24 /tmp/owner



`times` 是否维护文件的修改时间戳信息(mtime),默认值与`archive`设定值相同。

`dest_port` 目标节点的ssh端口号，在主机资源文件中(inventory file)设置的`ansible_ssh_port`选项值会覆盖本选项值。

`group` 维护所属组信息，默认值与`archive`设定值相同。

`perms` 维护文件的权限信息。默认值与`archive`设定值相同。


Playbooks示例：

    [root@web1 ~]# ansible-playbook /etc/ansible/sync.yml 
    
    PLAY [webservers] ************************************************************* 
    
    GATHERING FACTS *************************************************************** 
    ok: [192.168.1.65]
    
    TASK: [sync file] ************************************************************* 
    changed: [192.168.1.65 -> 127.0.0.1]
    
    PLAY RECAP ******************************************************************** 
    192.168.1.65   : ok=2changed=1unreachable=0failed=0   
    
    [root@web1 ~]# cat /etc/ansible/sync.yml
    ---
    
    - hosts: webservers
      remote_user: root
      tasks:
    - name: sync file
      synchronize: src=/tmp/t dest=/tmp

官方文档上的示例：

    # Synchronization of src on the control machine to dest on the remote hosts
    synchronize: src=some/relative/path dest=/some/absolute/path
    
    # Synchronization without any --archive options enabled
    synchronize: src=some/relative/path dest=/some/absolute/path archive=no
    
    # Synchronization with --archive options enabled except for --recursive
    synchronize: src=some/relative/path dest=/some/absolute/path recursive=no
    
    # Synchronization with --archive options enabled except for --times, with --checksum option enabled
    synchronize: src=some/relative/path dest=/some/absolute/path checksum=yes times=no
    
    # Synchronization without --archive options enabled except use --links
    synchronize: src=some/relative/path dest=/some/absolute/path archive=no links=yes
    
    # Synchronization of two paths both on the control machine
    local_action: synchronize src=some/relative/path dest=/some/absolute/path
    
    # Synchronization of src on the inventory host to the dest on the localhost in
    pull mode
    synchronize: mode=pull src=some/relative/path dest=/some/absolute/path
    
    # Synchronization of src on delegate host to dest on the current inventory host.
    # If delegate_to is set to the current inventory host, this can be used to syncronize
    # two directories on that host.
    synchronize: >
    src=some/relative/path dest=/some/absolute/path
    delegate_to: delegate.host
    
    # Synchronize and delete files in dest on the remote host that are not found in src of localhost.
    synchronize: src=some/relative/path dest=/some/absolute/path delete=yes
    
    # Synchronize using an alternate rsync command
    synchronize: src=some/relative/path dest=/some/absolute/path rsync_path="sudo rsync"
    
    # Example .rsync-filter file in the source directory
    - var   # exclude any path whose last part is 'var'
    - /var  # exclude any path starting with 'var' starting at the source directory
    + /var/conf # include /var/conf even though it was previously excluded
    
    # Synchronize passing in extra rsync options
    synchronize: src=/tmp/helloworld dest=/var/www/helloword rsync_opts=--no-motd,--exclude=.git


注：必须在控制节点和远程节点上都安装了rsync.
[此模块官方文档](http://docs.ansible.com/ansible/synchronize_module.html)


*`ini_file` 配置ini文件*

管理（添加，删除，修改）ini文件中的某个项(section)。如果该sections不存在则会添加一个新的，源文件中的注释会被命令丢弃，所以目标文件中不会保存注释信息。

该模块需要python安装`ConfigParser`包。

选项：

`backup` 是否为目标文件创建一个备份文件，默认为no.

`dest` 目标ini文件，如果该文件不存在则会创建之。

`group`  文件或目录的所属组。

`mode` 文件或目录的权限，如`0644`.从1.8版本后还可以使用符号权限模式，如 `u+rwx`或`u=rw,g=r,o=r`

`owner` 文件或目录的所有者

`section` section名称，如果`state=present`，在设置某个值的时候该section会被自动添加进文件中。

`option` 要管理section中的哪个选项。

`value` 要给`option`设置的值

    [root@web1 ~]# ansible webservers -m ini_file -a "dest=/etc/test.ini section=Mongo option=Host value=127.0.0.1"
    192.168.1.65 | success >> {
    "changed": true, 
    "dest": "/etc/test.ini", 
    "gid": 0, 
    "group": "root", 
    "mode": "0644", 
    "msg": "OK", 
    "owner": "root", 
    "secontext": "unconfined_u:object_r:etc_t:s0", 
    "size": 26, 
    "state": "file", 
    "uid": 0
    }
    #远程节点
    [root@db2 ~]# cat /etc/test.ini 
    [Mongo]
    Host = 127.0.0.1

Playbooks示例：

    [root@web1 ~]# ansible-playbook /etc/ansible/ini.yml 
    
    PLAY [webservers] ************************************************************* 
    
    GATHERING FACTS *************************************************************** 
    ok: [192.168.1.65]
    
    TASK: [test ini_file] ********************************************************* 
    changed: [192.168.1.65]
    
    PLAY RECAP ******************************************************************** 
    192.168.1.65   : ok=2changed=1unreachable=0failed=0   
    
    [root@web1 ~]# vim /etc/ansible/ini.yml
    [root@web1 ~]# cat /etc/ansible/ini.yml
    ---
    - hosts: webservers
      remote_user: root
      tasks:
       - name: test ini_file
 
         ini_file: dest=/etc/test.ini
                   section=drinks
                   option=temperature
                   value=cold
                   backup=yes
    #远程节点
    [root@db2 ~]# cat /etc/test.ini 
    [Mongo]
    Host = 127.0.0.1
    
    [drinks]
    temperature = cold

[该模块官方文档](http://docs.ansible.com/ansible/ini_file_module.html)


*`stat` 获取文件或者文件系统的状态信息*

类似Linux命令`stat`，获取文件的属性信息。

`follow` 无默认值，是否获取符号链接所指向源文件的信息。默认情况下，当path是个符号链接的时候，只获取符号链接本身的信息。

`path` 文件的路径


    [root@web1 ~]# ansible webservers -m stat -a "path=/tmp/testfilelink"
    192.168.1.65 | success >> {
    "changed": false, 
    "stat": {
        "atime": 1437708044.3021781, 
        "ctime": 1437708042.1791775, 
        "dev": 2050, 
        "exists": true, 
        "gid": 0, 
        "gr_name": "root", 
        "inode": 1058342, 
        "isblk": false, 
        "ischr": false, 
        "isdir": false, 
        "isfifo": false, 
        "isgid": false, 
        "islnk": true, 
        "isreg": false, 
        "issock": false, 
        "isuid": false, 
        "lnk_source": "/tmp/testfile", 
        "mode": "0777", 
        "mtime": 1437708042.1791775, 
        "nlink": 1, 
        "path": "/tmp/testfilelink", 
        "pw_name": "root", 
        "rgrp": true, 
        "roth": true, 
        "rusr": true, 
        "size": 13, 
        "uid": 0, 
        "wgrp": true, 
        "woth": true, 
        "wusr": true, 
        "xgrp": true, 
        "xoth": true, 
        "xusr": true
    }
    }

    [root@web1 ~]# ansible webservers -m stat -a "path=/tmp/testfilelink follow=yes"
    192.168.1.65 | success >> {
    "changed": false, 
    "stat": {
        "atime": 1437707925.3743169, 
        "checksum": "fcff57d112692c8f1c3d330714358c59f1c268cc", 
        "ctime": 1437549773.34641, 
        "dev": 2050, 
        "exists": true, 
        "gid": 0, 
        "gr_name": "root", 
        "inode": 1186223, 
        "isblk": false, 
        "ischr": false, 
        "isdir": false, 
        "isfifo": false, 
        "isgid": false, 
        "islnk": false, 
        "isreg": true, 
        "issock": false, 
        "isuid": false, 
        "md5": "e9409172a4036cc688f169c72131e921", 
        "mode": "0644", 
        "mtime": 1437549773.0564165, 
        "nlink": 1, 
        "path": "/tmp/testfilelink", 
        "pw_name": "root", 
        "rgrp": true, 
        "roth": true, 
        "rusr": true, 
        "size": 9, 
        "uid": 0, 
        "wgrp": false, 
        "woth": false, 
        "wusr": true, 
        "xgrp": false, 
        "xoth": false, 
        "xusr": false
    }
    }


Playbooks示例：

	# Obtain the stats of /etc/foo.conf, and check that the file still belongs
	# to 'root'. Fail otherwise.
	- stat: path=/etc/foo.conf
	  register: st #此处表示，将stat返回值赋值给st变量
	- fail: msg="Whoops! file ownership has changed"
	  when: st.stat.pw_name != 'root' #引用st变量值做判断操作
	
	# Determine if a path exists and is a symlink. Note that if the path does
	# not exist, and we test sym.stat.islnk, it will fail with an error. So
	# therefore, we must test whether it is defined.
	# Run this to understand the structure, the skipped ones do not pass the
	# check performed by 'when'
	- stat: path=/path/to/something
	  register: sym
	- debug: msg="islnk isn't defined (path doesn't exist)"
	  when: sym.stat.islnk is not defined
	- debug: msg="islnk is defined (path must exist)"
	  when: sym.stat.islnk is defined
	- debug: msg="Path exists and is a symlink"
	  when: sym.stat.islnk is defined and sym.stat.islnk
	- debug: msg="Path exists and isn't a symlink"
	  when: sym.stat.islnk is defined and sym.stat.islnk == False
	
	
	# Determine if a path exists and is a directory.  Note that we need to test
	# both that p.stat.isdir actually exists, and also that it's set to true.
	- stat: path=/path/to/something
	  register: p
	- debug: msg="Path exists and is a directory"
	  when: p.stat.isdir is defined and p.stat.isdir
	
	# Don't do md5 checksum
	- stat: path=/path/to/myhugefile get_md5=no


	[root@web1 ~]# ansible-playbook /etc/ansible/stat.yml
	
	PLAY [webservers] ************************************************************* 
	
	GATHERING FACTS *************************************************************** 
	ok: [192.168.1.65]
	
	TASK: [stat] ****************************************************************** 
	ok: [192.168.1.65]
	
	TASK: [debug msg="test debug msg"] ******************************************** 
	ok: [192.168.1.65] => {
	    "msg": "test debug msg"
	}
	
	PLAY RECAP ******************************************************************** 
	192.168.1.65               : ok=3    changed=0    unreachable=0    failed=0 


	[root@web1 ~]# cat /etc/ansible/stat.yml
	---
	- hosts: webservers
	  remote_user: root
	  tasks:
	    - name: stat
	      stat: path=/tmp/testfile
	      register: st
	    - debug: msg="test debug msg"
	      when: st.stat.pw_name == 'root'

[该模块官方文档](http://docs.ansible.com/ansible/stat_module.html)


*`lineinfule` 文件中指定的行是否存在，使用正则的后向引用替换某一行内容*

该模块会搜索文件中的某一行是否存在，也可以用来修改单行内容。如果需要修改多行内容的话可以使用`replace`模块。

选项：

`backup` 默认为no,是否在修改之前为目标文件做一个备份。

`state` 默认为present,指定的行是否存在

`create` 默认为no,当`state=present`时，指定该参数为yes表示当文件不存在时自动创建。默认情况下当文件不存在时返回错误。

`dest` 要操作的目标文件路径。

`line` 需要指定`state=present`,将新的一行插入文件或替换其中的某行内容。如果指定了`backrefs=yes`，则可以使用正则表达式的后向引用功能。

`regexp` 正则表达式，若`state=present`,则为替换匹配到的行，只替换最后一次匹配到的行。若`state=absent`,会删除匹配到的行。

	[root@web1 ~]# ansible webservers -m lineinfile -a "dest=/etc/myapp.conf regexp=^HOST line=HOST=127.0.0.1"
	192.168.1.65 | success >> {
	    "backup": "", 
	    "changed": true, 
	    "msg": "line replaced"
	
	#远程节点
	[root@db2 ~]# cat /etc/myapp.conf
	[server]
	HOST=127.0.0.1
	USER=root
	PASSWD=123456
	
	#删除匹配到的行
	
	[root@web1 ~]# ansible webservers -m lineinfile -a "dest=/etc/myapp.conf regexp=^HOST state=absent"
	192.168.1.65 | success >> {
	    "backup": "", 
	    "changed": true, 
	    "found": 1, 
	    "msg": "1 line(s) removed"
	}

`backrefs` 默认为no,是否使用正则表达式的后向引用。



`insertafter` 默认为`EOF`，`state=present`，将会在最后一次匹配到内容的后边插入新的指定内容。`EOF`表示在文件默认插入一个新航。如果正则表达式没有匹配到任何内容则效果和使用`EOF`一样，表示在文件默认加入新行。

	#默认会在文件默认新加一行，当没有匹配到任何内容时

	[root@web1 ~]# ansible webservers -m lineinfile -a "dest=/etc/myapp.conf regexp=^HOST line=HOST=127.0.0.1"
	192.168.1.65 | success >> {
	    "backup": "", 
	    "changed": true, 
	    "msg": "line added"
	}
	
	#在匹配到行的后边新加一行,如果同时也指定了regexp，会替换掉该表达式匹配到的行
	[root@web1 ~]# ansible webservers -m lineinfile -a "dest=/etc/myapp.conf line=PORT=3306 insertafter=^USER"
	192.168.1.65 | success >> {
	    "backup": "", 
	    "changed": true, 
	    "msg": "line added"
	}
	
	#远程节点
	[root@db2 ~]# cat /etc/myapp.conf 
	[server]
	USER=root
	PORT=3306
	PASSWD=123456
	HOST=127.0.0.1


`insertbefore` 默认是不启用该选项,需要`state=present`，在最后一次匹配到的行之前添加新航。`BOF`表示在文件开头新加一行，如果正则表达式没有匹配到任何内容则在文件开头新加一行。

	[root@web1 ~]# ansible webservers -m lineinfile -a "dest=/etc/myapp.conf line='#comment:test' insertbefore='^\[server'"
	192.168.1.65 | success >> {
	    "backup": "", 
	    "changed": true, 
	    "msg": "line added"
	}

	#远程节点
	[root@db2 ~]# cat /etc/myapp.conf 
	#comment:test
	[server]
	USER=root
	PORT=3307
	PASSWD=123456
	HOST=127.0.0.1

官方的Playbooks示例:
	- lineinfile: dest=/etc/selinux/config regexp=^SELINUX= line=SELINUX=enforcing
	
	- lineinfile: dest=/etc/sudoers state=absent regexp="^%wheel"
	
	- lineinfile: dest=/etc/hosts regexp='^127\.0\.0\.1' line='127.0.0.1 localhost' owner=root group=root mode=0644
	
	- lineinfile: dest=/etc/httpd/conf/httpd.conf regexp="^Listen " insertafter="^#Listen " line="Listen 8080"
	
	- lineinfile: dest=/etc/services regexp="^# port for http" insertbefore="^www.*80/tcp" line="# port for http by default"
	
	# Add a line to a file if it does not exist, without passing regexp
	- lineinfile: dest=/tmp/testfile line="192.168.1.99 foo.lab.net foo"
	
	# Fully quoted because of the ': ' on the line. See the Gotchas in the YAML docs.
	- lineinfile: "dest=/etc/sudoers state=present regexp='^%wheel' line='%wheel ALL=(ALL) NOPASSWD: ALL'"
	
	- lineinfile: dest=/opt/jboss-as/bin/standalone.conf regexp='^(.*)Xms(\d+)m(.*)$' line='\1Xms${xms}m\3' backrefs=yes
	
	# Validate the sudoers file before saving
	- lineinfile: dest=/etc/sudoers state=present regexp='^%ADMIN ALL\=' line='%ADMIN ALL=(ALL) NOPASSWD:ALL' validate='visudo -cf %s'

[此模块官方文档](http://docs.ansible.com/ansible/lineinfile_module.html)

*`template` 将控制节点的模板文件做变量替换后，传到远程节点*

Ansible使用了Jinja2的模板解析引擎。
在模板中可使用到的附加变量：

1. `ansible_managed` 在配置文件`ansible.cfg`的`defaults` 中定义,包含模板名称，主机，模板文件的修改时间和所有者UID信息。
1. `template_host` 模板主机的节点名称。
1. `template_uid` 模板所有者UID。
1. `template_path`，`template_fullpath` 模板的绝对路径。
1. `template_run_date` 模板被渲染的日期.

选项：

`backup` 默认为no,是否备份目标文件

`dest` 要操作的文件在远程节点上的路径

`force` 默认为yes,当目标文件与源文件内容不一样的时候，目标文件会被替换。若改为no,该文件只能不存在的时候被传输一次。

`src` 模板文件的路径，可以是绝对路径也可以是相对路径。

	[root@web1 ~]# cat /tmp/mytemplate.j2
	[mysql]
	host={{ host_ip }}#变量引用，这些变量在playbooks中定义
	user={{ user_name }}
	passwd={{ password }}
	[root@web1 ~]# cat /etc/ansible/tem.yml 
	---
	- hosts: webservers
	  remote_user: root
	#定义变量，这些变量会替换模板里面相应的值
	  vars:
	    host_ip: 192.168.1.200
	    user_name: testname
	    password: 123456
	
	  tasks:
	    - name: Config file
	      template: src=/tmp/mytemplate.j2 dest=/etc/file.conf mode=0644
	[root@web1 ~]# ansible-playbook /etc/ansible/tem.yml 
	
	PLAY [webservers] ************************************************************* 
	
	GATHERING FACTS *************************************************************** 
	ok: [192.168.1.65]
	
	TASK: [Config file] *********************************************************** 
	changed: [192.168.1.65]
	
	PLAY RECAP ******************************************************************** 
	192.168.1.65               : ok=2    changed=1    unreachable=0    failed=0 
	
	#远程节点
	[root@db2 ~]# cat /etc/file.conf 
	[mysql]
	host=192.168.1.200
	user=testname
	passwd=123456

官方playbooks示例：

	# Example from Ansible Playbooks
	- template: src=/mytemplates/foo.j2 dest=/etc/file.conf owner=bin group=wheel mode=0644
	
	# The same example, but using symbolic modes equivalent to 0644
	- template: src=/mytemplates/foo.j2 dest=/etc/file.conf owner=bin group=wheel mode="u=rw,g=r,o=r"
	
	# Copy a new "sudoers" file into place, after passing validation with visudo
	- template: src=/mine/sudoers dest=/etc/sudoers validate='visudo -cf %s'


[该模块官方文档](http://docs.ansible.com/ansible/template_module.html)



## 系统相关模块(System Modules) ##


*`authorized_key` 添加或删除SSH认证key*

添加或删除指定用户的认证key.

选项：

`key` ssh公钥信息，一个字符串或者是url.

`user` 被管理ssh  key的用户账户。

官方playbooks示例：

	# Example using key data from a local file on the management machine
	- authorized_key: user=charlie key="{{ lookup('file', '/home/charlie/.ssh/id_rsa.pub') }}"
	
	# Using github url as key source
	- authorized_key: user=charlie key=https://github.com/charlie.keys
	
	# Using alternate directory locations:
	- authorized_key: user=charlie
	                  key="{{ lookup('file', '/home/charlie/.ssh/id_rsa.pub') }}"
	                  path='/etc/ssh/authorized_keys/charlie'
	                  manage_dir=no
	
	# Using with_file
	- name: Set up authorized_keys for the deploy user
	  authorized_key: user=deploy
	                  key="{{ item }}"
	  with_file:
	    - public_keys/doe-jane
	    - public_keys/doe-john
	
	# Using key_options:
	- authorized_key: user=charlie
	                  key="{{ lookup('file', '/home/charlie/.ssh/id_rsa.pub') }}"
	                  key_options='no-port-forwarding,host="10.0.1.1"'
	
	# Set up authorized_keys exclusively with one key
	- authorized_key: user=root key=public_keys/doe-jane state=present
	                   exclusive=yes


[该模块官方文档](http://docs.ansible.com/ansible/authorized_key_module.html)


*`cron` 管理cron.d和crontab计划任务*

该模块用来管理crontab计划任务，可以增加，更新，删除计划任务。

`backup` 默认不启用，在修改之前是否将原来的计划任务备份。

`cron_file` 默认不启用，如果指定，将使用指定的文件代替用户的crontab.该文件应该都在cron.d目录下。

`day` 默认为`*`,日(1-31,*,*/2)

`hour` 默认为`*`,时 (0-23,*,*/2)

`minute` 默认为`*`,分(0-59,*,*/2)

`month` 默认为`*`,月(1-12,*,*/2)

`weekday` 默认为`*`,周(0-6,*)

`job` 计划任务要执行的命令

`user` 要管理哪个用户的计划任务

`state` 默认为present,当为absent时，可以按照计划任务`name`删除指定条目

`special_time` 默认不启用。可以指定某个特殊时间运行的任务。`reboot`重启的时候执行，`yearly`每年执行一次,`annually` 每年执行一次,`monthly`每个月执行一次,`weekly`每周执行一次,`daily`每年执行一次,`hourly`每小时执行一次.


	[root@web1 ~]# ansible webservers -m cron -a "name='test cron' hour=5,2 job='ls -l >>/dev/null' user=liuzhenwei"
	192.168.1.65 | success >> {
	    "changed": true, 
	    "jobs": [
	        "test cron"
	    ]
	}
	
	#远程节点
	[liuzhenwei@db2 ~]$ crontab -l
	#Ansible: test cron
	* 5,2 * * * ls -l >>/dev/null

	#删除计划任务name为'test cron'
	
	[root@web1 ~]# ansible webservers -m cron -a "name='test cron' user=liuzhenwei state=absent"
	192.168.1.65 | success >> {
	    "changed": true, 
	    "jobs": []
	}

	#重启执行的任务
	[root@web1 ~]# ansible webservers -m cron -a "name='test cron' user=liuzhenwei special_time=reboot job='ls -l'"
	192.168.1.65 | success >> {
	    "changed": true, 
	    "jobs": [
	        "test cron"
	    ]
	}

	#远程节点
	
	[liuzhenwei@db2 ~]$ crontab -l
	#Ansible: test cron
	@reboot ls -l


Playbooks 示例：

	# Ensure a job that runs at 2 and 5 exists.
	# Creates an entry like "0 5,2 * * ls -alh > /dev/null"
	- cron: name="check dirs" minute="0" hour="5,2" job="ls -alh > /dev/null"
	
	# Ensure an old job is no longer present. Removes any job that is prefixed
	# by "#Ansible: an old job" from the crontab
	- cron: name="an old job" state=absent
	
	# Creates an entry like "@reboot /some/job.sh"
	- cron: name="a job for reboot" special_time=reboot job="/some/job.sh"
	
	# Creates a cron file under /etc/cron.d
	- cron: name="yum autoupdate" weekday="2" minute=0 hour=12
	        user="root" job="YUMINTERACTIVE=0 /usr/sbin/yum-autoupdate"
	        cron_file=ansible_yum-autoupdate
	
	# Removes a cron file from under /etc/cron.d
	- cron: name="yum autoupdate" cron_file=ansible_yum-autoupdate state=absent

[该模块官方文档](http://docs.ansible.com/ansible/cron_module.html)


*`service` 管理服务*

管理远程节点的服务，支持这几种服务模式：BSD init, OpenRC, SysV, Solaris SMF, systemd, upstart

选项：

`enabled` 服务是否开机启动

`must_exist` 默认为True,避免当某个服务不存在的时候，模块执行终止。2.0版本中才开始支持该选项。

`name` 服务名称。

`runlevel` 默认值是服务本身的默认设置。该服务的运行级别。

`state` 对服务要执行的操作，`started`,`stopped`,`restarted`,`reloaded`

`args` 需要给命令提供的附加参数

	[root@web1 ~]# ansible webservers -m service -a "name=httpd state=started"
	192.168.1.65 | success >> {
	    "changed": true, 
	    "name": "httpd", 
	    "state": "started"
	}
	
	#远程节点
	[root@db2 ~]# lsof -i:80
	COMMAND   PID   USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
	httpd   15910   root    5u  IPv6 446993      0t0  TCP *:http (LISTEN)
	httpd   15913 apache    5u  IPv6 446993      0t0  TCP *:http (LISTEN)
	httpd   15914 apache    5u  IPv6 446993      0t0  TCP *:http (LISTEN)
	httpd   15915 apache    5u  IPv6 446993      0t0  TCP *:http (LISTEN)
	httpd   15916 apache    5u  IPv6 446993      0t0  TCP *:http (LISTEN)
	httpd   15917 apache    5u  IPv6 446993      0t0  TCP *:http (LISTEN)
	httpd   15918 apache    5u  IPv6 446993      0t0  TCP *:http (LISTEN)
	httpd   15919 apache    5u  IPv6 446993      0t0  TCP *:http (LISTEN)
	httpd   15920 apache    5u  IPv6 446993      0t0  TCP *:http (LISTEN)

	#设置开机启动
	[root@web1 ~]# ansible webservers -m service -a "name=httpd enabled=yes"
	192.168.1.65 | success >> {
	    "changed": true, 
	    "enabled": true, 
	    "name": "httpd"
	}
	

	#远程节点
	[root@db2 ~]# chkconfig --list httpd
	httpd          	0:off	1:off	2:on	3:on	4:on	5:on	6:off


Playbooks示例：

	# Example action to start service httpd, if not running
	- service: name=httpd state=started
	
	# Example action to stop service httpd, if running
	- service: name=httpd state=stopped
	
	# Example action to restart service httpd, in all cases
	- service: name=httpd state=restarted
	
	# Example action to reload service httpd, in all cases
	- service: name=httpd state=reloaded
	
	# Example action to enable service httpd, and not touch the running state
	- service: name=httpd enabled=yes
	
	# Example action to start service foo, based on running process /usr/bin/foo
	- service: name=foo pattern=/usr/bin/foo state=started
	
	# Example action to restart network service for interface eth0
	- service: name=network state=restarted args=eth0
	
	# Example action to restart nova-compute if it exists
	- service: name=nova-compute state=restarted must_exist=no

[该模块官方文档](http://docs.ansible.com/ansible/service_module.html)

*`setup` 获取远程节点的facts信息*

该模块会自动调用playbooks获取远程节点的信息

`fact_path` fact文件路径，默认在`/etc/ansible/facts.d`，利用该文件可以自定义本节点的facts信息，默认是不存在的。

`filter` 过滤返回的facts信息

	[root@web1 ~]# ansible webservers -m setup
	192.168.1.65 | success >> {
	    "ansible_facts": {
	        "ansible_all_ipv4_addresses": [
	            "192.168.1.65"
	        ], 
	        "ansible_all_ipv6_addresses": [
	            "fe80::250:56ff:fe93:a885"
	        ], 
	        "ansible_architecture": "x86_64", 
	        "ansible_bios_date": "01/07/2011", 
	        "ansible_bios_version": "6.00", 
	        "ansible_cmdline": {
	            "KEYBOARDTYPE": "pc", 
	            "KEYTABLE": "us", 
	            "LANG": "en_US.UTF-8", 
	            "SYSFONT": "latarcyrheb-sun16", 
	            "crashkernel": "129M@0M", 
	            "quiet": true, 
	            "rd_NO_DM": true, 
	            "rd_NO_LUKS": true, 
	            "rd_NO_LVM": true, 
	            "rd_NO_MD": true, 
	            "rhgb": true, 
	            "ro": true, 
	            "root": "UUID=c3e6eda7-d67d-4787-8f9f-6534941f11fd"
	        }, 
	        "ansible_date_time": {
	            "date": "2015-07-24", 
	            "day": "24", 
	            "epoch": "1437727501", 
	            "hour": "16", 
	            "iso8601": "2015-07-24T08:45:01Z", 
	            "iso8601_micro": "2015-07-24T08:45:01.852906Z", 
	            "minute": "45", 
	            "month": "07", 
	            "second": "01", 
	            "time": "16:45:01", 
	            "tz": "CST", 
	            "tz_offset": "+0800", 
	            "weekday": "Friday", 
	            "year": "2015"
	        },

	#下边输出信息省略.......


官方的示例：

	# Display facts from all hosts and store them indexed by I(hostname) at C(/tmp/facts).
	ansible all -m setup --tree /tmp/facts
	
	# Display only facts regarding memory found by ansible on all hosts and output them.
	ansible all -m setup -a 'filter=ansible_*_mb'#可以根据正则匹配过滤出所需的信息
	
	# Display only facts returned by facter.
	ansible all -m setup -a 'filter=facter_*'
	
	# Display only facts about certain interfaces.
	ansible all -m setup -a 'filter=ansible_eth[0-2]'

[该模块官方文档](http://docs.ansible.com/ansible/setup_module.html)


还其他很多与系统操作相关的模块,如管理用户，管理用户组等，具体参考[官方文档](http://docs.ansible.com/ansible/list_of_system_modules.html)


## 包管理相关模块(Packaging Modules) ##

该系列包含了大部分包管理相关的模块，具体参考[官方文档](http://docs.ansible.com/ansible/list_of_packaging_modules.html)

介绍几个较为常用的模块。

*`yum` 使用yum包管理器管理软件包*

该模块使用yum包管理器可以对软件包进行安装，更新，删除以及查看包和组的列表信息。

选项：

`conf_file` 远程节点的yum配置文件，没有默认值

`disabled_gpg_check` 默认值为no,安装软件包的时候是否进行GPG检查，该选项只有在`state=present`或`state=latest`起作用。

`disablerepo` 在安装或更新的时候禁用这些yum仓库，当有多个仓库需要禁用的时候以','隔开。默认不启用该选项。

`enablerepo` 在安装或更新软件包的时候启用这些yum仓库，多个仓库以','隔开。默认不启用该选项。

`exclude` 在2.0版本新加的选项，当`state=present`或`state=latest`排除指定的软件包。

`name` 软件包的名称，同时也可以指定包的版本,如`name-1.0`。当指定`state=latest`,该选项指定参数`*`,这表示要更新所有的软件包，和运行命令`yum -y update`作用一致。该选项也可以指定一个rpm包的url链接，或者一个本地的rpm路径。如果需要指定多个软件包以','隔开即可。

`state` 指定对软件包的操作行为，删除或者安装。默认为 `present`表示安装包。其他可以指定的值,`present`,`latest`都是安装包，`absent`表示卸载包。

`update_cache` 在`state=present`或`state=latest`是否强制更新缓存。在1.9版中开始支持该选项。

安装vsftp

	[root@web1 ~]# ansible webservers -m yum -a "name=vsftpd state=latest"
	192.168.1.65 | success >> {
	    "changed": true, 
	    "msg": "", 
	    "rc": 0, 
	    "results": [
	        "Loaded plugins: fastestmirror, security\nLoading mirror speeds from cached hostfile\nSetting up Install Process\nResolving Dependencies\n--> Running transaction check\n---> Package vsftpd.x86_64 0:2.2.2-13.el6_6.1 will be installed\n--> Finished Dependency Resolution\n\nDependencies Resolved\n\n================================================================================\n Package        Arch           Version                    Repository       Size\n================================================================================\nInstalling:\n vsftpd         x86_64         2.2.2-13.el6_6.1           updates         151 k\n\nTransaction Summary\n================================================================================\nInstall       1 Package(s)\n\nTotal download size: 151 k\nInstalled size: 332 k\nDownloading Packages:\nRunning rpm_check_debug\nRunning Transaction Test\nTransaction Test Succeeded\nRunning Transaction\n\r  Installing : vsftpd-2.2.2-13.el6_6.1.x86_64                               1/1 \n\nInstalled:\n  vsftpd.x86_64 0:2.2.2-13.el6_6.1                                              \n\nComplete!\n"
	    ]
	}


卸载软件包

	[root@web1 ~]# ansible webservers -m yum -a "name=vsftpd state=absent"
	192.168.1.65 | success >> {
	    "changed": true, 
	    "msg": "", 
	    "rc": 0, 
	    "results": [
	        "Loaded plugins: fastestmirror, security\nSetting up Remove Process\nResolving Dependencies\n--> Running transaction check\n---> Package vsftpd.x86_64 0:2.2.2-13.el6_6.1 will be erased\n--> Finished Dependency Resolution\n\nDependencies Resolved\n\n================================================================================\n Package        Arch           Version                   Repository        Size\n================================================================================\nRemoving:\n vsftpd         x86_64         2.2.2-13.el6_6.1          @updates         332 k\n\nTransaction Summary\n================================================================================\nRemove        1 Package(s)\n\nInstalled size: 332 k\nDownloading Packages:\nRunning rpm_check_debug\nRunning Transaction Test\nTransaction Test Succeeded\nRunning Transaction\n\r  Erasing    : vsftpd-2.2.2-13.el6_6.1.x86_64                               1/1 \n\nRemoved:\n  vsftpd.x86_64 0:2.2.2-13.el6_6.1                                              \n\nComplete!\n"
	    ]


官方Playbooks示例:

	- name: install the latest version of Apache
	  yum: name=httpd state=latest
	
	- name: remove the Apache package
	  yum: name=httpd state=absent
	
	- name: install the latest version of Apache from the testing repo
	  yum: name=httpd enablerepo=testing state=present
	
	- name: install one specific version of Apache
	  yum: name=httpd-2.2.29-1.4.amzn1 state=present
	
	- name: upgrade all packages
	  yum: name=* state=latest
	
	- name: install the nginx rpm from a remote repo
	  yum: name=http://nginx.org/packages/centos/6/noarch/RPMS/nginx-release-centos-6-0.el6.ngx.noarch.rpm state=present
	
	- name: install nginx rpm from a local file
	  yum: name=/usr/local/src/nginx-release-centos-6-0.el6.ngx.noarch.rpm state=present
	
	- name: install the 'Development tools' package group
	  yum: name="@Development tools" state=present

注：

当在playbooks循环操作多个包的时候，ansible会对所有的包调用一次yum模块，而不是每个包都调用一次该模块。


[该模块官方文档](http://docs.ansible.com/ansible/yum_module.html)


*`rpm_key` 添加或删除rpm的gpg key*

为rpm DB添加（类似`rpm --import`）或删除gpg key.

选项：

`key` 要被操作的key,可以指定key的url,key文件，或者是key在数据库中的key id。

`state` 对key进行的操作，导入或者去除。默认值为`present`表示导入key,`absent`表示去除指定的key.

`validate_certs` 如果为`no`,而且`key`指定了一个https的url，此时不对证书做验证。该选项默认值为`yes`。


示例：

	# Example action to import a key from a url
	- rpm_key: state=present key=http://apt.sw.be/RPM-GPG-KEY.dag.txt
	
	# Example action to import a key from a file
	- rpm_key: state=present key=/path/to/key.gpg
	
	# Example action to ensure a key is not present in the db
	- rpm_key: state=absent key=DEADB33F

[该模块官方文档](http://docs.ansible.com/ansible/rpm_key_module.html)


*`package` 一个通用的包管理模块*

该模块在2.0版中才开始支持,利用目标系统下的包管理器，安装、更新或者卸载软件包。


选项：

`name` 软件包的名称，同时也可以指定包的版本,如`name-1.0`。当指定`state=latest`,该选项指定参数`*`,这表示要更新所有的软件包，和运行命令`yum -y update`作用一致。该选项也可以指定一个rpm包的url链接，或者一个本地的rpm路径。如果需要指定多个软件包以','隔开即可。

`state` 指定对软件包的操作行为，删除或者安装。默认为 `present`表示安装包。其他可以指定的值,`present`,`latest`都是安装包，`absent`表示卸载包。

`use` 默认值为`auto`,指定需要使用哪种包管理器(yum,apt等等)，默认为`auto`，表示根据已有的`facts`或者自动检测目标节点，以确定使用哪种包管理器。

示例：
	
	- name: install the latest version of ntpdate
	  package: name=ntpdate state=latest
	
	# This uses a variable as this changes per distribution.
	- name: remove the apache package
	  package : name={{apache}} state=absent


注：
该模块实际是根据莫标节点系统调用相应的包管理模块(apt,yum等等)。


[该模块官方文档](http://docs.ansible.com/ansible/package_module.html)

*`easy_install` 安装Python相关库或包*

在Python的virtualenv中安装Python库。

`name` Python库名称

`state` 2.0版中新加特性，默认值为`present`,表示安装制定库即可，若值为`latest`则为表示要安装最新版。

`virtualenv` 默认不启用该选项，制定一个特定的虚拟环境目录，若指定的虚拟环境目录不存在则会自动创建。

`virtualenv_command` 默认为`virtualenv`，创建虚拟环境所使用的命令，可选值有`pyenv`,`virtualenv`,`virtualenv2`。

`executable` 默认不启用该选项，指定要运行的特定版本的`easy_install `。如果再系统中同时存在Python2.7和3.3的版本,此时想为Python3.3版本安装库或者包，该选项指定值为`easy_install-3.3`即可。

示例：

	# Examples from Ansible Playbooks
	- easy_install: name=pip state=latest
	
	# Install Bottle into the specified virtualenv.
	- easy_install: name=bottle virtualenv=/webapps/myapp/venv

注：

easy_install 模块只能用来安装Python的库，而且该模块不具备卸载库的功能。

[该模块官方文档](http://docs.ansible.com/ansible/easy_install_module.html)


