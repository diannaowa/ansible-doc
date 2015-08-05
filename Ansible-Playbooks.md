# Ansible Playbooks #
---

@author 刘振伟

@qq 570962906


Playbooks可以称为是Ansible的配置，部署，编排语言。在Playbooks中，你可以定义远程节点要执行的策略，或者是要执行的某组动作。如：希望远程节点要先安装httpd,创建一个vhost,最后启动httpd。


如果在你的工作中Ansible是一个工具模块的话，那么Playbooks就是你的设计方案。

最基本的，Playbooks可以配置或者部署远程节点主机；高级些的应用是，可以按照一定的次序执行特定的操作，可以对执行节点的主机进行监控，同时也可监控其负载均衡情况。

Playbooks具有较好的可读性，其是基于纯文本语言进行编排设计的。有多种方式来组织Playbooks以及文件包含。


官方提供了一些比较好的Playbooks示例，托管在[github](https://github.com/ansible/ansible-examples)上.


**Playbooks简单示例**

Playbooks使用了YAML格式，具有较少的语法格式。其特意的避免将自己设计为一门语言或者脚本，而是一个纯粹的配置和过程模型。


每个playbooks都有一个或者多个`plays`组成。

`play`简单来说就是一些主机节点要执行一些任务。最基本的，一个任务也就是调用一个Ansible模块。

通过为Playbooks编写多个`plays`,可以协调多主机的部署，按照一定的步骤在`webservers`组内的所有主机上执行，然后按照一定的步骤在`databaseserver`组内的主机上执行，然后再有其他更多的命令在`webservers`组内主机上执行，等等。

可以为主机上不同的操作定义多个`plays`,并在可以在不同的时间运行不同的`plays`。

下边是个只包含一个`play`的playbooks:

	[root@web1 ~]# cat /etc/ansible/http.yml
	---
	- hosts: webservers
	  vars:
	    http_port: 80 #定义变量
	    max_clients: 200 #定义变量
	  remote_user: root
	  tasks:
	  - name: ensure apache is at the latest version
	    yum: pkg=httpd state=latest
	  - name: write the apache config file
	    template: src=/etc/ansible/httpd.j2 dest=/etc/httpd.conf
	    notify:
	    - restart apache
	  - name: ensure apache is running (and enable it at boot)
	    service: name=httpd state=started enabled=yes
	  handlers:
	    - name: restart apache
	      service: name=httpd state=restarted


	[root@web1 ~]# ansible-playbook /etc/ansible/http.yml 
	
	PLAY [webservers] ************************************************************* 
	
	GATHERING FACTS *************************************************************** 
	ok: [192.168.1.65]
	
	TASK: [ensure apache is at the latest version] ******************************** 
	changed: [192.168.1.65]
	
	TASK: [write the apache config file] ****************************************** 
	changed: [192.168.1.65]
	
	TASK: [ensure apache is running (and enable it at boot)] ********************** 
	changed: [192.168.1.65]
	
	NOTIFIED: [restart apache] **************************************************** 
	changed: [192.168.1.65]
	
	PLAY RECAP ******************************************************************** 
	192.168.1.65               : ok=5    changed=4    unreachable=0    failed=0 

 在模板httpd.j2中会引用上边定义的两个变量，如{{ max_clients }}
`notify`表示当配置文件发生变化时会触发`restart apache`动作
`handlers`定义一些要被触发的动作。

我们也可以将每个任务定义为多行模式，使用YAML字典类型来描述模块的参数。

	---
	- hosts: webservers
	  vars:
	    http_port: 80
	    max_clients: 200
	  remote_user: root
	  tasks:
	  - name: ensure apache is at the latest version
	    yum:
	      pkg: httpd
	      state: latest
	  - name: write the apache config file
	    template:
	      src: /srv/httpd.j2
	      dest: /etc/httpd.conf
	    notify:
	    - restart apache
	  - name: ensure apache is running
	    service:
	      name: httpd
	      state: started
	  handlers:
	    - name: restart apache
	      service:
	        name: httpd
	        state: restarted


playbooks也可以包含多个`plays`,如下边的示例，在同一个playbooks中既可以对`web servers`组操作，也可以对`datavase servers`组操作：

	---
	- hosts: webservers
	  remote_user: root
	
	  tasks:
	  - name: ensure apache is at the latest version
	    yum: pkg=httpd state=latest
	  - name: write the apache config file
	    template: src=/srv/httpd.j2 dest=/etc/httpd.conf
	
	- hosts: databases
	  remote_user: root
	
	  tasks:
	  - name: ensure postgresql is at the latest version
	    yum: name=postgresql state=latest
	  - name: ensure that postgresql is started
	    service: name=postgresql state=running

`plays`就像任务一样按照在playbooks中的定义顺序从上到下依次执行。


## playbooks基础知识点 ##

**主机和用户(Hosts and Users)**

Playbooks中的每个`play`,都需要指定一个远程主机节点和要以哪个远程用户执行任务。

`hosts`指定主机组的列表，或者是一个主机的匹配正则表达式。多个主机组或者多个匹配模式之间以冒号隔开。`remote_user`指定一个用户账户。

	---
	- hosts: webservers
	  remote_user: root


远程用户也可以在每个任务中指定：

	---
	- hosts: webservers
	  remote_user: root
	  tasks:
	    - name: test connection
	      ping:
	      remote_user: yourname


也可以以其他用户的身份来执行任务(sudo):

	---
	- hosts: webservers
	  remote_user: yourname
	  sudo: yes

示例：

	[root@web1 ~]# cat /etc/ansible/sudo.yml 
	---
	- hosts: webservers
	  remote_user: liuzhenwei
	  sudo: yes
	  tasks:
	    - name: create file as root
	      file: path=/tmp/testfile2 state=touch

	[root@web1 ~]# ansible-playbook /etc/ansible/sudo.yml -K -k
	SSH password: 
	SUDO password[defaults to SSH password]: 
	
	PLAY [webservers] ************************************************************* 
	
	GATHERING FACTS *************************************************************** 
	ok: [192.168.1.65]
	
	TASK: [create file as root] *************************************************** 
	changed: [192.168.1.65]
	
	PLAY RECAP ******************************************************************** 
	192.168.1.65               : ok=2    changed=1    unreachable=0    failed=0
	
	#远程节点
	[root@db2 ~]# ll /tmp/testfile2
	-rw-r--r--. 1 root root 0 Jul 27 15:01 /tmp/testfile2

也可以为每个任务指定是否使用sudo,而不是应用于整个`play`中：


	---
	- hosts: webservers
	  remote_user: yourname
	  tasks:
	    - service: name=nginx state=started
	      become: yes
	      become_method: sudo


也可以先以自己的账户登录，然后切换为其他非root用户：

	---
	- hosts: webservers
	  remote_user: yourname
	  become: yes
	  become_user: postgres

示例：

	[root@web1 ~]# cat /etc/ansible/become.yml
	---
	- hosts: webservers
	  remote_user: liuzhenwei
	  become: yes
	  become_user: tom
	  tasks:
	   - name: touch file 
	     file: path=/tmp/sufile state=touch

	[root@web1 ~]# ansible-playbook /etc/ansible/become.yml -k --ask-become-pass
	SSH password: 
	SU password[defaults to SSH password]: 
	
	PLAY [webservers] ************************************************************* 
	
	GATHERING FACTS *************************************************************** 
	
	ok: [192.168.1.65]
	
	TASK: [touch file] ************************************************************ 
	changed: [192.168.1.65]
	
	PLAY RECAP ******************************************************************** 
	192.168.1.65               : ok=2    changed=1    unreachable=0    failed=0

	#远程节点
	[root@db2 ~]# ll /tmp/sufile 
	-rw-r--r--. 1 tom tom 0 Jul 27 15:21 /tmp/sufile


也可以使用su来提升用户权限：

	---
	- hosts: webservers
	  remote_user: yourname
	  become: yes
	  become_method: su

示例：


	[root@web1 ~]# cat /etc/ansible/become.yml 
	---
	- hosts: webservers
	  remote_user: liuzhenwei
	  become: yes
	  become_method: su
	  tasks:
	   - name: touch file 
	     file: path=/tmp/sufiles state=touch

	[root@web1 ~]# ansible-playbook /etc/ansible/become.yml -k --ask-su-pass
	SSH password: 
	SU password[defaults to SSH password]: 
	
	PLAY [webservers] ************************************************************* 
	
	GATHERING FACTS *************************************************************** 
	ok: [192.168.1.65]
	
	TASK: [touch file] ************************************************************ 
	changed: [192.168.1.65]
	
	PLAY RECAP ******************************************************************** 
	192.168.1.65               : ok=2    changed=1    unreachable=0    failed=0

	#远程节点
	[root@db2 ~]# ll /tmp/sufiles
	-rw-rw-r--. 1 root root 0 Jul 27 15:23 /tmp/sufiles


**任务列表**


每个`play`都包含了一个任务列表，任务是按照顺序执行的，且每个任务会都在指定的所有远程节点上执行，然后再执行下一个任务。

playbooks从上到下执行，如果某个主机的某个任务执行失败则推出。

每个任务实际上都是执行了一个模块，并且为该模块指定了特定的参数。文件上边定义的变量，可以用在模块的参数内。

模块都是等幂的，意思就是当你再次运行的 时候，他们只做他们必须要做的操作（改变某些文件状态等），以使系统达到所需的状态。这样的话，当你重复运行多次相同的playbooks时，也是非常安全的。

每个任务都应该有一个名称，这个名称会在运行playbook的时候输出，如果没有指定名称，那么对主机做的操作行为将会被用来输出到屏幕上。

任务可以以这种格式来定义：`action:module options`。但是建议使用较为常用的一种格式: `module:options`来定义你的任务。

下边是一个简单的任务示例：


	tasks:
	  - name: make sure apache is running
	    service: name=httpd state=running


`command`和`shell`模块只需要一个参数列表既可：

	tasks:
	  - name: disable selinux
	    command: /sbin/setenforce 0


`command`和`shell`会关注它的返回码，如果有一个自定义的脚本或者命令，成功执行后的退出码为非0值，可以进行以下操作：

	tasks:
	  - name: run this command and ignore the result
	    shell: /usr/bin/somecommand || /bin/true

或者是这样的方式：

	tasks:
	  - name: run this command and ignore the result
	    shell: /usr/bin/somecommand
	    ignore_errors: True


如果参数较多的话也可以将其换行书写：

	tasks:
	  - name: Copy ansible inventory file to client
	    copy: src=/etc/ansible/hosts dest=/etc/ansible/hosts
	            owner=root group=root mode=0644

同样也可以在tasks中引用变量:

	tasks:
	  - name: create a virtual host file for {{ vhost }}
	    template: src=somefile.j2 dest=/etc/httpd/conf.d/{{ vhost }}

这些变量都会在`vars`中定义。

同样也可以在模板中使用变量。稍后会介绍。

**Handlers:当某些操作发生改变时触发执行的任务**

正如前面提到的，Ansible模块都是幂等的，而且当在远程主机上做了改变之后也可以传递`change`这个事件。Playbooks有一个基本的事件系统来对‘改变’做出响应。

`notify`做指定的行为会在每个任务的最后触发，如果再多个不同的任务里指定了相同的触发事件，那么只会触发一次。


例如，多个资源都表名需要重启Apache，因为他们都修改了配置文件。但是Apache只会重启一次，这样就避免了一些不必要的重启动作。


下边的例子，表名当文件内容发生变化的时候，重启两个服务:

	- name: template configuration file
	  template: src=template.j2 dest=/etc/foo.conf
	  notify:
	     - restart memcached
	     - restart apache

`notify`中的任务称为`handlers`。

`handlers`也是一个任务列表，与普通`tasks`不同的是hanlders下边的每个操作行为都需要指定一个全局唯一的名称。如果没有任务操作触发该hanlders，那么它将不会运行。如论有多少个操作都触发了同一个handlers，在所有的任务执行完成后，也只会触发执行一次该handlders。

`handlers`定义格式如下：

	handlers:
	    - name: restart memcached
	      service: name=memcached state=restarted
	    - name: restart apache
	      service: name=apache state=restarted


示例：

	[root@web1 ~]# cat /etc/ansible/notify.yml
	---
	- hosts: webservers
	  remote_user: root
	  vars:
	    http_port: 8083
	    max_clients: 220
	  tasks:
	   - name: template config file
	     template: src=/etc/ansible/httpd.j2 dest=/etc/httpd.conf
	     notify:
	       - restart apache
	  handlers:
	    - name: restart apache
	      service: name=httpd state=restarted

	[root@web1 ~]# ansible-playbook /etc/ansible/notify.yml
	
	PLAY [webservers] ************************************************************* 
	
	GATHERING FACTS *************************************************************** 
	ok: [192.168.1.65]
	
	TASK: [template config file] ************************************************** 
	changed: [192.168.1.65]
	
	NOTIFIED: [restart apache] **************************************************** 
	changed: [192.168.1.65]
	
	PLAY RECAP ******************************************************************** 
	192.168.1.65               : ok=3    changed=2    unreachable=0    failed=0 


注：

- `notify hanlders`都会按照书写的循序运行。
- `hanlders`名称必须是全局唯一。
- 如果有两个`handlers`定义了相同的名称，只会有一个`handlers`会运行。


**执行Playbooks**

	ansible-playbook playbook.yml -f 10

-f|--forks 指定使用的并发进程数

**Ansible-Pull**

`ansible-pull`可以从git仓库中检索出最新的配置信息，然后运行`ansible-playbook`。


[该部分内容的官方文档](http://docs.ansible.com/ansible/playbooks_intro.html)


## Playbook角色和include语法 ##

Playbooks有时候很可能是一个比较大的文件，里面包含了各种各样的对主机的操作，当你想复用某些功能的时候，不得不重新考虑怎么组织文件。

最基本的，可以将`tasks`分隔到一个小文件里，然后包含任务文件。因为`handlers`也是任务，所以也可以在`handlers`区段包含handler文件。

playbooks也可以包含其他playbook文件。

当你开始思考任务、`handlers`,变量等等的时候，会发现是一个很大的概念。这个时候就需要考虑到建模，而不是单纯的让某个事物看起来像某个事物。不再是将某些操作应用到某些主机上。在编程的时候使用封装思想来组织这些，如，一个人可以驾驶汽车，但他并不需要了解发动机是怎么工作的。

Ansible中的角色就是利用包含文件这个思想来使自己组织的更加清晰，可复用。它使得你更加专注于大局面，只在必要的时候考虑细节实现即可。


**任务包含文件和重用**

如果想在多个playbooks之间重用任务列表，那么文件包含将会是个好方法。


下边是一个简单的任务列表文件：

	---
	# possibly saved as tasks/foo.yml
	
	- name: placeholder foo
	  command: /bin/foo
	
	- name: placeholder bar
	  command: /bin/bar


以方式包含文件：

	tasks:
	
	  - include: tasks/foo.yml

也可以向包含文件内传递参数，称之为"参数化包含"。

例如，想要部署多个wordpress，可以将部署的任务写到一个文件中，wordpress.yml,然后想下边这样包含文件：

	tasks:
	  - include: wordpress.yml wp_user=timmy
	  - include: wordpress.yml wp_user=alice
	  - include: wordpress.yml wp_user=bob

还可以支持一下包含文件方式，传递一个参数列表：

	tasks:
	 - { include: wordpress.yml, wp_user: timmy, ssh_keys: [ 'keys/one.txt', 'keys/two.txt' ] }

无论使用哪种包含文件的方式，传递的变量都可以在被包含的文件里调用：

	{{ wp_user }}

这种称之为显式传递参数，在`vars`中定义的参数也可以在被包含文件中引用。

从1.0版本开始，也可以这样传递参数：

	tasks:
	
	  - include: wordpress.yml
	    vars:
	        wp_user: timmy
	        ssh_keys:
	          - keys/one.txt
	          - keys/two.txt


也可以在`handlers`中使用文件包含。例如，想要定义重启apache的hanlder，为所有的playbooks只能定义一次，可以以下方式定义一个handlers.yml文件:

	---
	# this might be in a file like handlers/handlers.yml
	- name: restart apache
	  service: name=httpd state=restarted

在主playbook文件中包含该文件：

	handlers:
	  - include: handlers/handlers.yml

包含还可以将一个playbook导入到另一个playbook中。这样的话就可以定义一个顶级的较为通用的playbooks来包含其他文件：

	- name: this is a play at the top level of a file
	  hosts: all
	  remote_user: root
	
	  tasks:
	
	  - name: say hi
	    tags: foo
	    shell: echo "hi..."
	
	- include: load_balancers.yml
	- include: webservers.yml
	- include: dbservers.yml

示例：

	[root@web1 ansible]# tree /etc/ansible/
	/etc/ansible/
	├── handlers
	│   └── restart.yml
	├── hosts
	├── main.yml
	├── tasks
	│   └── apache.yml
	└── templates
	    └── httpd.j2

	[root@web1 ansible]# cat tasks/apache.yml 
	---
	- name: config httpd.file
	  template: src=httpd.j2 dest=/etc/httpd.conf
	[root@web1 ansible]# cat handlers/restart.yml 
	---
	- name: restart apache
	  service: name=httpd state=restarted
	[root@web1 ansible]# cat main.yml 
	---
	- hosts: webservers
	  remote_user: root
	  vars:
	    http_port: 8085
	    max_clients: 123
	  
	  tasks:
	    - include: tasks/apache.yml
	    
	  handlers:
	    - include: handlers/restart.yml

	[root@web1 ansible]# ansible-playbook /etc/ansible/main.yml 
	
	PLAY [webservers] ************************************************************* 
	
	GATHERING FACTS *************************************************************** 
	ok: [192.168.1.65]
	
	TASK: [config httpd.file] ***************************************************** 
	changed: [192.168.1.65]
	
	PLAY RECAP ******************************************************************** 
	192.168.1.65               : ok=2    changed=1    unreachable=0    failed=0


**角色**

现在已经了解了`tasks`,`handlers`,但是有什么更好方法来组织自己的playbooks呢？很简单，那就是角色，角色会基于已知的文件结构自动加载特定的变量文件，任务文件以及`handlers`文件。使用角色来组织内容可以更方便的共享。

一个简单的使用角色组织文件的示例：

	site.yml
	webservers.yml
	fooservers.yml
	roles/
	   common/
	     files/
	     templates/
	     tasks/
	     handlers/
	     vars/
	     defaults/
	     meta/
	   webservers/
	     files/
	     templates/
	     tasks/
	     handlers/
	     vars/
	     defaults/
	     meta/

在playbooks中这样定义：

	---
	- hosts: webservers
	  roles:
	     - common
	     - webservers


每一个角色都会遵循以下的行为：

- 如果`roles/x/tasks/main.yml`存在，里面的任务列表会被添加到`play`中。
- 如果`roles/x/handlers/main.yml`存在，里面的`handlers`会被添加到`play`中。
- 如果`roles/x/vars/main.yml`存在，里面的变量会被添加到`play`中。
- 如果`roles/x/meta/main.yml`存在，里面的角色依赖会被添加到角色列表中。
- 在`roles/x/files`任务所需要被复制的文件，无需绝对路径或者相对路径都可以引用该文件。
- 在`roles/x/files`中的任务脚本都可以直接使用该文件，无需指定绝对路径或者是相对路径。
- 在`roles/x/templates`中的模板，无需指定绝对路径或者相对路径，都可以直接使用文件名引用该文件。
- 需要包含在`roles/x/tasks`中的任务文件时，无需指定绝对路径或者相对路径，可以直接使用文件名包含。

`tasks,handlers,vars,meta`目录下`main.yml`中的内容都会被自动调用。

在Ansible1.4或之后的版本，可以在配置文件中指定`roles_path`指定一个路径，用来查找角色。

如果某些文件不存在的话，将会自动跳过，所以`vars/`之类的子目录如果用不到的话也可以不创建。

也可以给角色传递参数：

	---
	
	- hosts: webservers
	  roles:
	    - common
	    - { role: foo_app_instance, dir: '/opt/a',  port: 5000 }
	    - { role: foo_app_instance, dir: '/opt/b',  port: 5001 }

大多数情况，也可以给角色添加执行条件：

	---
	
	- hosts: webservers
	  roles:
	    - { role: some_role, when: "ansible_os_family == 'RedHat'" }

在特定的条件下才会执行角色下的任务。

也可以给角色指定一个标签：

	---
	
	- hosts: webservers
	  roles:
	    - { role: foo, tags: ["bar", "baz"] }

如果在`playbooks`中有一个`tasks`块存在，它将会在角色执行完成后执行。

定义在角色执行之前或者执行后需要执行的任务：

	---
	
	- hosts: webservers
	
	  pre_tasks:
	    - shell: echo 'hello'
	
	  roles:
	    - { role: some_role }
	
	  tasks:
	    - shell: echo 'still busy'
	
	  post_tasks:
	    - shell: echo 'goodbye'


示例：

	[root@web1 ~]# tree /etc/ansible/
	/etc/ansible/
	├── hosts
	├── roles
	│   └── webservers
	│       ├── defaults
	│       ├── files
	│       ├── handlers
	│       │   ├── main.yml
	│       │   └── restart.yml
	│       ├── meta
	│       ├── tasks
	│       │   ├── apache.yml
	│       │   └── main.yml
	│       ├── templates
	│       │   └── httpd.j2
	│       └── vars
	│           └── main.yml
	└── webservers.yml


	[root@web1 ~]# cat /etc/ansible/roles/webservers/tasks/main.yml 
	---
	- include: apache.yml
	[root@web1 ~]# cat /etc/ansible/roles/webservers/tasks/apache.yml 
	---
	- name: config httpd.file
	  template: src=httpd.j2 dest=/etc/httpd.conf
	  notify:
	    - restart apache
	[root@web1 ~]# cat /etc/ansible/roles/webservers/handlers/main.yml 
	---
	- include: restart.yml
	[root@web1 ~]# cat /etc/ansible/roles/webservers/handlers/restart.yml 
	---
	- name: restart apache
	  service: name=httpd state=restarted
	[root@web1 ~]# cat /etc/ansible/roles/webservers/vars/main.yml 
	---
	http_port: 8099
	max_clients: 321

	[root@web1 ansible]# ansible-playbook /etc/ansible/webservers.yml
	
	PLAY [webservers] ************************************************************* 
	
	GATHERING FACTS *************************************************************** 
	ok: [192.168.1.65]
	
	TASK: [webservers | config httpd.file] **************************************** 
	changed: [192.168.1.65]
	
	NOTIFIED: [webservers | restart apache] *************************************** 
	changed: [192.168.1.65]
	
	PLAY RECAP ******************************************************************** 
	192.168.1.65               : ok=3    changed=2    unreachable=0    failed=0


有些子目录并没有用到 ，如meta,可以不用创建。


**角色默认变量**

要设置默认变量，只需要再角色目录下创建`defaults/main.yml`文件即可。默认变量的优先级最低，会被其他地方定义的变量所覆盖，`inventory`文件中定义 的变量也可以覆盖默认变量。

**角色依赖**

角色依赖使得当使用某个角色的时候自动将其他角色导入。角色依赖保存在角色目录下的`meta/main.yml`文件中，该文件包含一个角色和参数的列表，这些内容会在某个特定的角色之前插入。即在某个特定的角色执行之前会先执行其依赖。

	---
	dependencies:
	  - { role: common, some_parameter: 3 }
	  - { role: apache, port: 80 }
	  - { role: postgres, dbname: blarg, other_parameter: 12 }

也可以以全路径的方式定义：

	---
	dependencies:
	   - { role: '/path/to/common/roles/foo', x: 1 }


角色依赖总在包含它们的角色执行之前而执行，而且是递归执行。默认情况下，角色也只能作为一个依赖关系添加一次，如果另一个角色也将它列为一个依赖关系，它将不再运行。但是在`meta/main.yml`设置`allow_duplicates: yes`,可以突破这个限制。如一个称为`car`的角色需要添加一个名为`wheel`的角色作为它的依赖：

	---
	dependencies:
	- { role: wheel, n: 1 }
	- { role: wheel, n: 2 }
	- { role: wheel, n: 3 }
	- { role: wheel, n: 4 }

`meta/main.yml`内容：

	---
	allow_duplicates: yes
	dependencies:
	- { role: tire }
	- { role: brake }

执行的结果会是类似下边这样：

	tire(n=1)
	brake(n=1)
	wheel(n=1)
	tire(n=2)
	brake(n=2)
	wheel(n=2)
	...
	car

[更多角色相关示例文件](https://github.com/ansible/ansible-examples)

[该部分内容官方文档](http://docs.ansible.com/ansible/playbooks_roles.html)



## 变量 ##

Ansible使得运维可以实现平时工作的自动化，但是在日常运维工作中，所涉及到的系统并不都是完全一样的。

Ansible在运行的时候会自动收集远程节点的属性信息，如节点主机的IP地址，操作系统版本等信息。有时候需要根据获取到的不同的主机IP来设置不同的配置文件。

使用“变量”可以很方便的同时对每个主机做不同的操作。

**Playbooks中定义变量**

在playbooks文件中直接定义变量：

	- hosts: webservers
	  vars:
	    http_port: 80

定义了一个变量名为http_port的变量,值为80.

**通过文件包含和角色定义变量**

	[root@web1 ~]# cat /etc/ansible/roles/webservers/vars/main.yml 
	---
	http_port: 8099
	max_clients: 321

在角色目录下的`vars/main.yml`直接定义变量即可。


**Jinja2的语法格式来使用变量**

Ansible中可以使用Jinja2模板引擎来使用定义好的变量。

在模板文件中使用变量：

	My amp goes to {{ max_amp_value }}

也可以直接在playbooks文件中使用变量：

	template: src=foo.cfg.j2 dest={{ remote_install_path }}/foo.cfg

使用变量值来确定文件在远程节点上的存储位置。

在模板中可以访问到所有主机范围内的变量。事实上，也有方法可以访问到其他主机的变量。


**一个YAML问题**

在定义变量中引用其他变量

	- hosts: app_servers
	  vars:
	       app_path: "{{ base_path }}/22"

要是双引号将变量的值(如app_path的值)包括起来才行。

**系统的信息：facts**

还有另外一个地方可以获取一些变量，但这些变量的值是ansible自己搜集来的。

从远程节点搜集到的系统信息称为`facts`。

`facts`包括远程主机的IP地址，和操作系统类型，磁盘相关信息等等。

执行下边的命令来查看都有哪些信息：

	ansible hostname -m setup
返回内容类似下边这样：

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
	            "date": "2015-07-30", 
	            "day": "30", 
	            "epoch": "1438241766", 
	            "hour": "15", 
	            "iso8601": "2015-07-30T07:36:06Z", 
	            "iso8601_micro": "2015-07-30T07:36:06.373646Z", 
	            "minute": "36", 
	            "month": "07", 
	            "second": "06", 
	            "time": "15:36:06", 
	            "tz": "CST", 
	            "tz_offset": "+0800", 
	            "weekday": "Thursday", 
	            "year": "2015"
	        }, 
	        "ansible_default_ipv4": {
	            "address": "192.168.1.65", 
	            "alias": "eth1", 
	            "gateway": "192.168.1.254", 
	            "interface": "eth1", 
	            "macaddress": "00:50:56:93:a8:85", 
	            "mtu": 1500, 
	            "netmask": "255.255.255.0", 
	            "network": "192.168.1.0", 
	            "type": "ether"
	        }, 
	        "ansible_default_ipv6": {}, 
	        "ansible_devices": {
	            "sda": {
	                "holders": [], 
	                "host": "Serial Attached SCSI controller: VMware PVSCSI SCSI Controller (rev 02)", 
	                "model": "Virtual disk", 
	                "partitions": {
	                    "sda1": {
	                        "sectors": "409600", 
	                        "sectorsize": 512, 
	                        "size": "200.00 MB", 
	                        "start": "2048"
	                    }, 
	                    "sda2": {
	                        "sectors": "41943040", 
	                        "sectorsize": 512, 
	                        "size": "20.00 GB", 
	                        "start": "411648"
	                    }, 
	                    "sda3": {
	                        "sectors": "12288000", 
	                        "sectorsize": 512, 
	                        "size": "5.86 GB", 
	                        "start": "42354688"
	                    }, 
	                    "sda4": {
	                        "sectors": "2", 
	                        "sectorsize": 512, 
	                        "size": "1.00 KB", 
	                        "start": "54642688"
	                    }, 
	                    "sda5": {
	                        "sectors": "113124044", 
	                        "sectorsize": 512, 
	                        "size": "53.94 GB", 
	                        "start": "54642751"
	                    }
	                }, 
	                "removable": "0", 
	                "rotational": "1", 
	                "scheduler_mode": "cfq", 
	                "sectors": "167772160", 
	                "sectorsize": "512", 
	                "size": "80.00 GB", 
	                "support_discard": "0", 
	                "vendor": "VMware"
	            }, 
	            "sr0": {
	                "holders": [], 
	                "host": "IDE interface: Intel Corporation 82371AB/EB/MB PIIX4 IDE (rev 01)", 
	                "model": "VMware IDE CDR10", 
	                "partitions": {}, 
	                "removable": "1", 
	                "rotational": "1", 
	                "scheduler_mode": "cfq", 
	                "sectors": "2097151", 
	                "sectorsize": "512", 
	                "size": "1024.00 MB", 
	                "support_discard": "0", 
	                "vendor": "NECVMWar"
	            }
	        }, 
	        "ansible_distribution": "CentOS", 
	        "ansible_distribution_major_version": "6", 
	        "ansible_distribution_release": "Final", 
	        "ansible_distribution_version": "6.2", 
	        "ansible_domain": "", 
	        "ansible_env": {
	            "G_BROKEN_FILENAMES": "1", 
	            "HOME": "/root", 
	            "LANG": "en_US.UTF-8", 
	            "LC_CTYPE": "en_US.UTF-8", 
	            "LESSOPEN": "|/usr/bin/lesspipe.sh %s", 
	            "LOGNAME": "root", 
	            "MAIL": "/var/mail/root", 
	            "PATH": "/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin", 
	            "PWD": "/root", 
	            "SELINUX_LEVEL_REQUESTED": "", 
	            "SELINUX_ROLE_REQUESTED": "", 
	            "SELINUX_USE_CURRENT_RANGE": "", 
	            "SHELL": "/bin/bash", 
	            "SHLVL": "2", 
	            "SSH_CLIENT": "192.168.1.80 53565 22", 
	            "SSH_CONNECTION": "192.168.1.80 53565 192.168.1.65 22", 
	            "USER": "root", 
	            "_": "/usr/bin/python"
	        }, 
	        "ansible_eth1": {
	            "active": true, 
	            "device": "eth1", 
	            "ipv4": {
	                "address": "192.168.1.65", 
	                "netmask": "255.255.255.0", 
	                "network": "192.168.1.0"
	            }, 
	            "ipv6": [
	                {
	                    "address": "fe80::250:56ff:fe93:a885", 
	                    "prefix": "64", 
	                    "scope": "link"
	                }
	            ], 
	            "macaddress": "00:50:56:93:a8:85", 
	            "module": "vmxnet3", 
	            "mtu": 1500, 
	            "promisc": false, 
	            "type": "ether"
	        }, 
	        "ansible_fips": false, 
	        "ansible_form_factor": "Other", 
	        "ansible_fqdn": "db2", 
	        "ansible_hostname": "db2", 
	        "ansible_interfaces": [
	            "lo", 
	            "eth1"
	        ], 
	        "ansible_kernel": "2.6.32-220.el6.x86_64", 
	        "ansible_lo": {
	            "active": true, 
	            "device": "lo", 
	            "ipv4": {
	                "address": "127.0.0.1", 
	                "netmask": "255.0.0.0", 
	                "network": "127.0.0.0"
	            }, 
	            "ipv6": [
	                {
	                    "address": "::1", 
	                    "prefix": "128", 
	                    "scope": "host"
	                }
	            ], 
	            "mtu": 16436, 
	            "promisc": false, 
	            "type": "loopback"
	        }, 
	        "ansible_machine": "x86_64", 
	        "ansible_machine_id": "283cb3834e9c505bb450a80c0000000c", 
	        "ansible_memfree_mb": 1948, 
	        "ansible_memory_mb": {
	            "nocache": {
	                "free": 3445, 
	                "used": 388
	            }, 
	            "real": {
	                "free": 1948, 
	                "total": 3833, 
	                "used": 1885
	            }, 
	            "swap": {
	                "cached": 0, 
	                "free": 5999, 
	                "total": 5999, 
	                "used": 0
	            }
	        }, 
	        "ansible_memtotal_mb": 3833, 
	        "ansible_mounts": [
	            {
	                "device": "/dev/sda2", 
	                "fstype": "ext4", 
	                "mount": "/", 
	                "options": "rw", 
	                "size_available": 7911301120, 
	                "size_total": 21137846272, 
	                "uuid": "c3e6eda7-d67d-4787-8f9f-6534941f11fd"
	            }, 
	            {
	                "device": "/dev/sda1", 
	                "fstype": "ext4", 
	                "mount": "/boot", 
	                "options": "rw", 
	                "size_available": 160615424, 
	                "size_total": 203097088, 
	                "uuid": "a4ee8c3b-6c4d-4769-8928-d8c464261b95"
	            }
	        ], 
	        "ansible_nodename": "db2", 
	        "ansible_os_family": "RedHat", 
	        "ansible_pkg_mgr": "yum", 
	        "ansible_processor": [
	            "GenuineIntel", 
	            "Intel(R) Xeon(R) CPU           E5405  @ 2.00GHz", 
	            "GenuineIntel", 
	            "Intel(R) Xeon(R) CPU           E5405  @ 2.00GHz"
	        ], 
	        "ansible_processor_cores": 2, 
	        "ansible_processor_count": 1, 
	        "ansible_processor_threads_per_core": 1, 
	        "ansible_processor_vcpus": 2, 
	        "ansible_product_name": "VMware Virtual Platform", 
	        "ansible_product_serial": "VMware-56 4d b9 1f 28 12 95 da-bb 9f d9 92 bd 8a 4f cc", 
	        "ansible_product_uuid": "564DB91F-2812-95DA-BB9F-D992BD8A4FCC", 
	        "ansible_product_version": "None", 
	        "ansible_python_version": "2.6.6", 
	        "ansible_selinux": {
	            "config_mode": "enforcing", 
	            "mode": "permissive", 
	            "policyvers": 24, 
	            "status": "enabled", 
	            "type": "targeted"
	        }, 
	        "ansible_ssh_host_key_dsa_public": "AAAAB3NzaC1kc3MAAACBANrQNWsIo72zmhfUIqnV2iMT6LFKLmJAA224ThUSwmqsMpX5j3yfbrHrXjvxTxlJDfjtLkf65aggpju+aBu2dMLINi+KrUOSwzF+WDMhqXAsXxG478GwX5Jelvk6P2K1mzngtGudmezABzzHa//t/JwzeVhj/zKIEE0RZIJaSW1dAAAAFQC6Ys9dszeDIrDC81YZ/1msrJLpFwAAAIA93v+yt4lk2FV348UDLG1gFvdd7POxVMrbkeUq4twpAyeS7PxvkHs1p3BHXA2VyvICix5BULEuchVxkSK/hdXBPj7F3OCpQJhkNDKzE3ac3xAnXom1AQwIpmUEEzQ1nPIPAdnlfcgFx3PPoyUzbIwUEgvPENcPa/zzXrbdpWmz9wAAAIAQdOyJ6Mj5LD1YffMY5r4wZogCFPh2a/4Sji4N4nOFVwMBQBZcZQkhbdKYMk6K/xMYUdgWJI+MxO8o9d8aQbOA1Pqgq5YV0Vmnbbgaciu4gqbttS2np+xNsqkeRvTLxtZdYzqSE2yzhcojTqVXHNMAkiauwrIjNAkKIHLCFhAaww==", 
	        "ansible_ssh_host_key_rsa_public": "AAAAB3NzaC1yc2EAAAABIwAAAQEArOD1mR+s8FmMhMPODLMQ5fHiaVKXhwbrXk3mU090Dyx+Dw8UZDZl6mxULJGNR27n4N0PHWvn7L9F3zlLSg+r1hKAuG26XqOcO9TvXLBLQVVh9z5PnxOxH5UWjRD79unoi6NlObfbF2SPL3PRog/iWQRe2+gWikUy209APh1ZUvDo0uJHqdHhhUG6K3B/3hNkv/EZMdbMaqvX75SDeA8fdSS601VE0W1NCDPTJIYXfAWD4/cVzU+vGM7n7t7pTegRiQKI41AR/8Tr0TD3Fu0xljmNz3nni32xyouf6gi7WRcpb3gRIC8vITvAs/oTRHuSgn4hiL8gKTPxj9+Q/EyBWQ==", 
	        "ansible_swapfree_mb": 5999, 
	        "ansible_swaptotal_mb": 5999, 
	        "ansible_system": "Linux", 
	        "ansible_system_vendor": "VMware, Inc.", 
	        "ansible_user_dir": "/root", 
	        "ansible_user_gecos": "root", 
	        "ansible_user_gid": 0, 
	        "ansible_user_id": "root", 
	        "ansible_user_shell": "/bin/bash", 
	        "ansible_user_uid": 0, 
	        "ansible_userspace_architecture": "x86_64", 
	        "ansible_userspace_bits": "64", 
	        "ansible_virtualization_role": "guest", 
	        "ansible_virtualization_type": "VMware", 
	        "module_setup": true
	    }, 
	    "changed": false
	}


可以以一下方式使用上边返回的值：

	{{ ansible_devices.sda.model }}

值为Virtual disk。

要获取系统的主机名：

	{{ ansible_hostname }}


`facts`会经常在条件语句及模板中使用。

`facts`可以动态的创建符合特定模式的组和主机。

**关闭facts**

如果你确信不需要主机的任何facts信息，而且对远程节点主机都了解的很清楚，那么可以将其关闭。远程操作节点较多的时候，关闭facts会提升ansible的性能。

只需要在`play`中设置如下：

	- hosts: whatever
	  gather_facts: no


**本地facts(facts.d)**


通常情况下facts都是ansible的setup模块自动获取的。用户也可以根据API来自定义自己的facts模块。也可以以一种更简便的方法来自定义节点的facts相关信息,使用`facts.d`机制。

如果远程节点系统上存在`etc/ansible/facts.d`目录，这个目录下的以`.fact`为后缀的文件，里面的内容可以是JSON格式，或者ini格式书写；或者是一个可以返回json格式数据的可执行文件，都可以用来提供本地facts信息。


在远程节点创建一个`/etc/ansible/facts.d/preferences.fact`的文件，内容如下：

	[general]
	asdf=1
	bar=2

在控制节点获取自定义的信息：

	[root@web1 ~]# ansible webservers -m setup -a "filter=ansible_local"
	192.168.1.65 | success >> {
	    "ansible_facts": {
	        "ansible_local": {
	            "preferences": {
	                "general": {
	                    "asdf": "1", 
	                    "bar": "2"
	                }
	            }
	        }
	    }, 
	    "changed": false
	}

可以在playbooks或者模板中这样使用获取到的信息：

	{{ ansible_local.preferences.general.asdf }}


**facts缓存**

Ansible目前支持两种缓存方式：redis和json文件。

在配置文件ansible.cfg中设置如下：

	[defaults]
	gathering = smart #其值还可以设置为explicit
	fact_caching = redis
	fact_caching_timeout = 86400

目前来说还不支持设置 redis访问密码和为redis设置访问接口的功能。在不久的将来可能会加入支持。


缓存到json文件中，也是在ansible.cfg中配置：

	[defaults]
	gathering = smart
	fact_caching = jsonfile
	fact_caching_location = /path/to/cachedir
	fact_caching_timeout = 86400 #单位为妙

`fact_caching_location`指定一个可写的目录，若不存在的话ansible会尝试创建它。


**注册变量**

一个任务的运行结果都可以保存到一个变量中，供稍后使用，有时候需要将运行的命令的角色保存起来，并作为下一个人任务的执行条件。在运行playbooks的时候可以使用`-v`参数来显示执行过程中的结果信息。

具体格式如下：

	- hosts: web_servers
	
	  tasks:
	
	     - shell: /usr/bin/foo
	       register: foo_result
	       ignore_errors: True
	
	     - shell: /usr/bin/bar
	       when: foo_result.rc == 5

使用`register`将shell 模块执行的结果保存到变量foo_result中，并在之后引用该变量。

示例：

	#远程节点上创建一个shell脚本
	[root@db2 ~]# cat /tmp/test.sh 
	#!/bin/bash
	
	echo 123
	
	#控制节点
	[root@web1 ~]# cat /etc/ansible/register.yml 
	---
	- hosts: webservers
	  remote_user: root
	  tasks:
	   - shell: /tmp/test.sh
	     register: rst_value
	     ignore_errors: True
	   - name: touch file in tmp named register.log
	     file: state=touch dest=/tmp/register.log
	     when: rst_value.stdout == "123"

	#远程节点
	[root@db2 ~]# ll /tmp/register.log 
	-rw-r--r--. 1 root root 0 Jul 30 16:42 /tmp/register.log

当远程节点脚本/tmp/test.sh运行结束并输出123的时候，在远程节点上创建文件/tmp/register.log。


**访问复杂的变量**

在facts信息中，网络相关的信息是一个嵌套的数据结构，想获取IP地址：

	{{ ansible_eth0["ipv4"]["address"] }}

或者

	{{ ansible_eth0.ipv4.address }}

访问数组的第一个元素：

	{{ foo[0] }}


**将变量定义到单独的文件中**

也可以将变量定义到一个单独的文件中，然后在playbooks中使用`var_files`导入文件即可：

	---
	
	- hosts: all
	  remote_user: root
	  vars:
	    favcolor: blue
	  vars_files:
	    - /vars/external_vars.yml
	
	  tasks:
	
	  - name: this is just a placeholder
	    command: /bin/echo foo

变量文件也是YMAL格式：

	---
	# in the above example, this would be vars/external_vars.yml
	somevar: somevalue
	password: magic

当你在自动化工作中使用了ansible的角色，就不需要想上边那样定义变量文件了，在角色目录下的vars/main.yml中定义的变量，会自动导入到playbooks中，不需要显式包含变量文件。


**命令行传递变量**

除了上边介绍的变量定义形式，还可以通过命令行传递变量，如，有一个较为通用的节点部署playbooks,在每次部署的时候传递一个版本号进去：

	ansible-playbook release.yml --extra-vars "version=1.23.45 other_variable=foo"

也可以向playbooks传递主机组和远程用户信息：

	---
	
	- hosts: '{{ hosts }}'
	  remote_user: '{{ user }}'
	
	  tasks:
	     - ...
	
	ansible-playbook release.yml --extra-vars "hosts=vipers user=starbuck"

在ansible1.2中，可以以JSON格式传递变量：

	--extra-vars '{"pacman":"mrs","ghosts":["inky","pinky","clyde","sue"]}'

在ansible1.3中也可以使用JSON文件传递变量：

	--extra-vars "@some_file.json"

以上边的方式也可以传递一个YAML格式的变量文件。

[该部分内容官方文档](http://docs.ansible.com/ansible/playbooks_variables.html)


## Jinja2的过滤器 ##

将数据在模板中渲染成不同的格式：


	{{ some_variable | to_json }}
	{{ some_variable | to_yaml }}

更容易阅读的输出：

	{{ some_variable | to_nice_json }}
	{{ some_variable | to_nice_yaml }}


从已有的数据格式中读取数据：
	
	{{ some_variable | from_json }}
	{{ some_variable | from_yaml }}

示例：

	tasks:
	  - shell: cat /some/path/to/file.json
	    register: result
	
	  - set_fact: myvar="{{ result.stdout | from_json }}"

**在条件语句中使用过滤**

示例：


	tasks:
	
	  - shell: /usr/bin/foo
	    register: result
	    ignore_errors: True
	
	#当返回结果result为failed的时候输出显示该debug信息
	  - debug: msg="it failed"
	    when: result|failed
	
	  # in most cases you'll want a handler, but if you want to do something right now, this is nice
	  - debug: msg="it changed"
	    when: result|changed
	
	  - debug: msg="it succeeded"
	    when: result|success
	
	  - debug: msg="it was skipped"
	    when: result|skipped

**强制变量定义**

默认情况下，如果一个变量未被定义，ansible会报错，但是使用下边的方法可以关闭这个特性：

	{{ variable | mandatory }}

该变量可以正常使用，但是若在变量未定义，在模板中估计还是会抛出一个错误。

**未定义变量设置默认值**

Jinja2提供了一个`default`的过滤器，当某个变量未定义的时候会设置一个默认值：

	{{ some_variable | default(5) }}

当变量`some_variable`未被定义的时候，其值被设置为5，而不是抛出一个错误。


**忽略未定义变量和参数**

示例：

	[root@web1 ~]# cat /etc/ansible/va.yml
	---
	- hosts: webservers
	  remote_user: root
	  tasks:
	   - name: touch files with an optional mode
	     file: dest={{item.path}} state=touch mode={{item.mode|default(omit)}}
	     with_items:
	      - path: /tmp/foo
	      - path: /tmp/bar
	      - path: /tmp/baz
	        mode: "0444"

	[root@web1 ~]# ansible-playbook /etc/ansible/va.yml
	
	PLAY [webservers] ************************************************************* 
	
	GATHERING FACTS *************************************************************** 
	ok: [192.168.1.65]
	
	TASK: [touch files with an optional mode] ************************************* 
	changed: [192.168.1.65] => (item={'path': '/tmp/foo'})
	changed: [192.168.1.65] => (item={'path': '/tmp/bar'})
	changed: [192.168.1.65] => (item={'path': '/tmp/baz', 'mode': '0444'})
	
	PLAY RECAP ******************************************************************** 
	192.168.1.65               : ok=2    changed=1    unreachable=0    failed=0 

创建前两个文件的时候并没有传递mode参数，最后一个文件才会传递mode=0444参数。


**列表过滤器**

从列表中获取最小值：

	{{ list1 | min }}

获取最大值：

	{{ [3, 4, 2] | max }}


**集合过滤器**

从一个列表中获取一个唯一集合：

	{{ list1 | unique }}

两个列表的并集：

	{{ list1 | union(list2) }}

两个列表的交集：

	{{ list1 | intersect(list2) }}

两个列表的差集，在列表1中存在，但不在列表2中存在：

	{{ list1 | difference(list2) }}

两个列表的对称差（只存在于某一个列表中的元素）：

	{{ list1 | symmetric_difference(list2) }}


**随机数**

从一个列表中获取一个随机元素：

	{{ ['a','b','c']|random }} => 'c'

获取一个0到59之间的数：

	{{ 59 |random}} * * * * root /script/from/cron

获取0到100之间的数，但步长为10：

	{{ 100 |random(step=10) }}  => 70

获取从1到100之间的一个随机数，但步长为10：

	{{ 100 |random(1, 10) }}    => 31
	{{ 100 |random(start=1, step=10) }}    => 51


获取一个列表的随机排序结果：

	{{ ['a','b','c']|shuffle }} => ['c','a','b']
	{{ ['a','b','c']|shuffle }} => ['b','c','a']


**Math**

判断一个变量是否数字:

	{{ myvar | isnan }}

取对数：

	{{ myvar | log }}


基于10的对数：

	{{ myvar | log(10) }}


2次方或5次方：

	{{ myvar | pow(2) }}
	{{ myvar | pow(5) }}

平方根或者是5次方根：

	{{ myvar | root }}
	{{ myvar | root(5) }}

**IP地址过滤器**

判断一个字符串是否是有效IP：

	{{ myvar | ipaddr }}

也可以指定一个特定的IP版本：

	{{ myvar | ipv4 }}
	{{ myvar | ipv6 }}

从CIDR中获取IP地址：

	{{ '192.0.2.1/24' | ipaddr('address') }}

**Hashing过滤器**

获取一个字符串的sha1 hash:

	{{ 'test1'|hash('sha1') }}

获取MD5值：

	{{ 'test1'|hash('md5') }}

获取checksum:

	{{ 'test2'|checksum }}

**其他一些过滤器**

为shell添加引号：

	- shell: echo={{ string_value | quote }}

三元表达式，当前边的表达式为真时取后边的第一个值，为假时取第二个值：

	{{ (name == "John") | ternary('Mr','Ms') }}

将列表转换成字符串：

	{{ list | join(" ") }}

获取路径的最后一项的值，如`etc/asdf/foo.txt`最后一项值为foo.txt:

	{{ path | basename }}

获取路径的目录：

	{{ path | dirname }}


扩展路径，包含字符`~`:

	{{ path | expanduser }}


获取链接(link)的真实路径：

	{{ path | realpath }}

从指定的起点，获取一个链接的相对路径：

	{{ path | relpath('/etc') }}

从一个路径或者文件名中获取名称和扩展名：

	# with path == 'nginx.conf' the return would be ('nginx', '.conf')
	{{ path | splitext }}

获取字符串的UUID值：

	{{ hostname | to_uuid }}

使用正则表达式匹配某个字符串，使用`match`或`search`:

	vars:
	  url: "http://example.com/users/foo/resources/bar"
	
	tasks:
	    - shell: "msg='matched pattern 1'"
	      when: url | match("http://example.com/users/.*/resources/.*")
	
	    - debug: "msg='matched pattern 2'"
	      when: url | search("/users/.*/resources/.*")

`match`是必须匹配整个字符串，`search`是搜到指定的子串即可。

正则表达式替换字符串：

	# convert "ansible" to "able"
	{{ 'ansible' | regex_replace('^a.*i(.*)$', 'a\\1') }}
	
	# convert "foobar" to "bar"
	{{ 'foobar' | regex_replace('^f.*o(.*)$', '\\1') }}
`\\1`为后向引用。

[该部分内容官方文档](http://docs.ansible.com/ansible/playbooks_filters.html#id13)


## 条件语句 ##

一般情况下，一个任务的结果有可能要依赖一个变量的值，或者是`facts`的信息，或者是一个任务的结果会决定下一个任务是否运行。在一些情况，一个变量的值有可能需要取决于另外一个变量的值。这就需要一些控制语句，来控制ansible的执行流程。

**`when`语句**

有时候需要根据一定的条件在特定的主机节点上跳过执行某些操作，如根据需要发行版来决定安装软件包的方式，或者是跳过某些命令的执行等。

Ansible中`when`语句的简单示例：

	tasks:
	  - name: "shutdown Debian flavored systems"
	    command: /sbin/shutdown -t now
	    when: ansible_os_family == "Debian"

当节点主机的系统为Debian时，执行关机操作。


也可以用小括号来确定符合语句的优先级：

	tasks:
	  - name: "shutdown CentOS 6 and 7 systems"
	    command: /sbin/shutdown -t now
	    when: ansible_distribution == "CentOS" and
	          (ansible_distribution_major_version == "6" or ansible_distribution_major_version == "7")


在`when`语句中也可以使用过滤器。如，我们想跳过一个语句执行中的错误，但是后续的任务的执行需要由该任务是否成功执行决定：

	tasks:
	  - command: /bin/false
	    register: result
	    ignore_errors: True
	  - command: /bin/something
	    when: result|failed
	  - command: /bin/something_else
	    when: result|success
	  - command: /bin/still/something_else
	    when: result|skipped

有时候需要将一个字符串的变量转换为整数来进行数字比较：


	tasks:
	  - shell: echo "only on Red Hat 6, derivatives, and later"
	    when: ansible_os_family == "RedHat" and ansible_lsb.major_release|int >= 6


在playbooks和inventory中定义的变量都可以使用，如，需要根据一个变量的bool值决定是否执行该任务：

	vars:
	  epic: true

条件语句：

	tasks:
	    - shell: echo "This certainly is epic!"
	      when: epic

或：

	tasks:
	    - shell: echo "This certainly isn't epic!"
	      when: not epic

如果引用的变量没有被定义，使用Jinja2的`defined`测试，可以跳过或者是抛出错误：

	tasks:
	    - shell: echo "I've got '{{ foo }}' and am not afraid to use it!"
	      when: foo is defined
	
	    - fail: msg="Bailing out. this play requires 'bar'"
	      when: bar is not defined

当`when`和`with_items`一起使用的时候，每个项都会单独被`when`语句处理：

	tasks:
	    - command: echo {{ item }}
	      with_items: [ 0, 2, 4, 6, 8, 10 ]
	      when: item > 5

示例：

	[root@web1 ~]# cat /etc/ansible/when.yml
	---
	- hosts: webservers
	  remote_user: root
	  tasks:
	   - command: echo {{ item }}
	     with_items: [ 1,2,3,4,5,6,8,10]
	     when: item > 5

	[root@web1 ~]# ansible-playbook /etc/ansible/when.yml
	
	PLAY [webservers] ************************************************************* 
	
	GATHERING FACTS *************************************************************** 
	ok: [192.168.1.65]
	
	TASK: [command echo {{ item }}] *********************************************** 
	skipping: [192.168.1.65] => (item=1)
	skipping: [192.168.1.65] => (item=2)
	skipping: [192.168.1.65] => (item=3)
	skipping: [192.168.1.65] => (item=4)
	skipping: [192.168.1.65] => (item=5)
	changed: [192.168.1.65] => (item=6)
	changed: [192.168.1.65] => (item=8)
	changed: [192.168.1.65] => (item=10)
	
	PLAY RECAP ******************************************************************** 
	192.168.1.65               : ok=2    changed=1    unreachable=0    failed=0


**加载自定义facts**

如果需要的话，也可以返回自定义的facts给控制节点。返回的自定义的facts变量也可以用作下个任务的执行条件：

	tasks:
	    - name: gather site specific fact data
	      action: site_facts
	    - command: /usr/bin/thingy
	      when: my_custom_fact_just_retrieved_from_the_remote_system == '1234'

**在角色和包含中使用`when`**


如果有多个任务都需要使用同一个条件语句控制。可以将这些任务打包到一个单独的任务文件中，然后使用`include`包含和`when`条件语句。条件语句只对包含任务文件起作用，对包含playbook文件不起作用。指定的条件语句会作用到所包含的每个任务上：

	- include: tasks/sometasks.yml
	  when: "'reticulating splines' in output"

角色中使用`when`:

	- hosts: webservers
	  roles:
	     - { role: debian_stock_config, when: ansible_os_family == 'Debian' }


**注册变量**

在playbook中将某个命令运行的结果保存起来，提供给后续任务使用。如，通过command模块来判断远程节点上某个文件是否存在或者通过执行某个命令的获取其返回结果，并保存起来，下个任务根据获取的变量值来决定执行的具体操作。

`register`关键字可以将任务执行结果保存到一个变量中，该变量可以在模板或者playbooks文件中使用：


	- name: test play
	  hosts: all
	
	  tasks:
	
	      - shell: cat /etc/motd
	        register: motd_contents
	
	      - shell: echo "motd contains the word hi"
	        when: motd_contents.stdout.find('hi') != -1

上边中的例子中，通过注册变量访问返回的内容，`stdout`里面保存了命令的标准输出内容。注册变量还可以使用在`with_items`中，如果其保存的内容可以转换为列表，或者内容本身就是个列表。如果命令的输出本身就是列表，可以通过`stdout_lines`访问：

	- name: registered variable usage as a with_items list
	  hosts: all
	
	  tasks:
	
	      - name: retrieve the list of home directories
	        command: ls /home
	        register: home_dirs
	
	      - name: add home dirs to the backup spooler
	        file: path=/mnt/bkspool/{{ item }} src=/home/{{ item }} state=link
	        with_items: home_dirs.stdout_lines
	        # same as with_items: home_dirs.stdout.split()

示例：

	[root@web1 ~]# cat /etc/ansible/rewith.yml
	---
	- hosts: webservers
	  remote_user: root
	  tasks:
	   - name: list of home dir
	     command: ls /home
	     register: home_dirs
	   - name: add home dirs to the backup
	     file: path=/tmp/back/{{ item }} src=/home/{{ item }} state=link
	     with_items: home_dirs.stdout_lines

	[root@web1 ~]# ansible-playbook /etc/ansible/rewith.yml
	
	PLAY [webservers] ************************************************************* 
	
	GATHERING FACTS *************************************************************** 
	ok: [192.168.1.65]
	
	TASK: [list of home dir] ****************************************************** 
	changed: [192.168.1.65]
	
	TASK: [add home dirs to the backup] ******************************************* 
	changed: [192.168.1.65] => (item=1.sql)
	changed: [192.168.1.65] => (item=1youku.sql)
	changed: [192.168.1.65] => (item=liuzhenwei)
	changed: [192.168.1.65] => (item=tom)
	
	PLAY RECAP ******************************************************************** 
	192.168.1.65               : ok=3    changed=2    unreachable=0    failed=0 

	###远程节点
	[root@db2 ~]# ll /tmp/back
	total 0
	lrwxrwxrwx. 1 root root 11 Aug  4 14:37 1.sql -> /home/1.sql
	lrwxrwxrwx. 1 root root 16 Aug  4 14:37 1youku.sql -> /home/1youku.sql
	lrwxrwxrwx. 1 root root 16 Aug  4 14:37 liuzhenwei -> /home/liuzhenwei
	lrwxrwxrwx. 1 root root  9 Aug  4 14:37 tom -> /home/tom


[该部分内容官方文档](http://docs.ansible.com/ansible/playbooks_conditionals.html)


## Loops ##

`with_items`

**标准循环**

循环的添加几个用户：

	- name: add several users
	  user: name={{ item }} state=present groups=wheel
	  with_items:
	     - testuser1
	     - testuser2

如果再事先已经定义好了一个列表变量：

	with_items: "{{somelist}}"


上边的循环语句执行方式与下边的任务执行结果相同：

	- name: add user testuser1
	  user: name=testuser1 state=present groups=wheel
	- name: add user testuser2
	  user: name=testuser2 state=present groups=wheel


使用`with_items`迭代循环的变量可以是个单纯的列表，也可以是一个包含字典的列表：

	- name: add several users
	  user: name={{ item.name }} state=present groups={{ item.groups }}
	  with_items:
	    - { name: 'testuser1', groups: 'wheel' }
	    - { name: 'testuser2', groups: 'root' }

**嵌套循环**

	- name: give users access to multiple databases
	  mysql_user: name={{ item[0] }} priv={{ item[1] }}.*:ALL append_privs=yes password=foo
	  with_nested:
	    - [ 'alice', 'bob' ]
	    - [ 'clientdb', 'employeedb', 'providerdb' ]

item[0]是循环的第一个列表的值['alice','bob']。item[1]是第二个列表的值。表示循环创建alice和bob两个用户，并且为其赋予在三个数据库上的所有权限。

也可以将用户列表事先赋值给一个变量：

	- name: here, 'users' contains the above list of employees
	  mysql_user: name={{ item[0] }} priv={{ item[1] }}.*:ALL append_privs=yes password=foo
	  with_nested:
	    - "{{users}}"
	    - [ 'clientdb', 'employeedb', 'providerdb' ]


**遍历Hashes**

如，有以下变量数据：

	---
	users:
	  alice:
	    name: Alice Appleworth
	    telephone: 123-456-7890
	  bob:
	    name: Bob Bananarama
	    telephone: 987-654-3210

现在需要输出每个用户的用户名和手机号。使用`with_dict`:

	tasks:
	  - name: Print phone records
	    debug: msg="User {{ item.key }} is {{ item.value.name }} ({{ item.value.telephone }})"
	    with_dict: "{{users}}"

**文件匹配遍历**

`with_fileglob`匹配单个目录下所有文件。

	---
	- hosts: all
	
	  tasks:
	
	    # first ensure our target directory exists
	    - file: dest=/etc/fooapp state=directory
	
	    # copy each file over that matches the given pattern
	    - copy: src={{ item }} dest=/etc/fooapp/ owner=root mode=600
	      with_fileglob:
	        - /playbooks/files/fooapp/*

注：在角色中以相对路径使用`with_fileglob`,ansible会将路径解析为`role/<rolename>/files`。

**遍历数据并行集合**


	[root@web1 ~]# cat /etc/ansible/loop.yml
	---
	- hosts: webservers
	  remote_user: root
	  vars:
	    alpha: [ 'a','b','c','d']
	    numbers: [ 1,2,3,4 ]
	  tasks:
	    - debug: msg="{{ item.0 }} and {{ item.1 }}"
	      with_together:
	         - "{{ alpha }}"
	         - "{{ numbers }}"

	[root@web1 ~]# ansible-playbook /etc/ansible/loop.yml
	
	PLAY [webservers] ************************************************************* 
	
	GATHERING FACTS *************************************************************** 
	ok: [192.168.1.65]
	
	TASK: [debug msg="{{ item.0 }} and {{ item.1 }}"] ***************************** 
	ok: [192.168.1.65] => (item=['a', 1]) => {
	    "item": [
	        "a", 
	        1
	    ], 
	    "msg": "a and 1"
	}
	ok: [192.168.1.65] => (item=['b', 2]) => {
	    "item": [
	        "b", 
	        2
	    ], 
	    "msg": "b and 2"
	}
	ok: [192.168.1.65] => (item=['c', 3]) => {
	    "item": [
	        "c", 
	        3
	    ], 
	    "msg": "c and 3"
	}
	ok: [192.168.1.65] => (item=['d', 4]) => {
	    "item": [
	        "d", 
	        4
	    ], 
	    "msg": "d and 4"
	}
	
	PLAY RECAP ******************************************************************** 
	192.168.1.65               : ok=2    changed=0    unreachable=0    failed=0


**遍历子元素**

假如现在需要遍历一个用户列表，并创建每个用户，而且还需要为每个用户配置以特定的SSH key登录。变量文件内容如下：

	---
	users:
	  - name: alice
	    authorized:
	      - /tmp/alice/onekey.pub
	      - /tmp/alice/twokey.pub
	    mysql:
	        password: mysql-password
	        hosts:
	          - "%"
	          - "127.0.0.1"
	          - "::1"
	          - "localhost"
	        privs:
	          - "*.*:SELECT"
	          - "DB1.*:ALL"
	  - name: bob
	    authorized:
	      - /tmp/bob/id_rsa.pub
	    mysql:
	        password: other-mysql-password
	        hosts:
	          - "db1"
	        privs:
	          - "*.*:SELECT"
	          - "DB2.*:ALL"


playbooks中定义如下：

	- user: name={{ item.name }} state=present generate_ssh_key=yes
	  with_items: "{{users}}"
	
	- authorized_key: "user={{ item.0.name }} key='{{ lookup('file', item.1) }}'"
	  with_subelements:
	     - users
	     - authorized

也可以遍历嵌套的子列表：

	- name: Setup MySQL users
	  mysql_user: name={{ item.0.user }} password={{ item.0.mysql.password }} host={{ item.1 }} priv={{ item.0.mysql.privs | join('/') }}
	  with_subelements:
	    - users
	    - mysql.hosts

**循环整数序列**

`with_sequence`可以生成一个自增的整数序列，可以指定起始值和结束值，也可以指定增长步长。

参数以`key=value`的形式指定，`format`指定输出的格式。

数字可以是十进制，十六进制，八进制。


	---
	- hosts: all
	
	  tasks:
	
	    # create groups
	    - group: name=evens state=present
	    - group: name=odds state=present
	
	    # create some test users
	    - user: name={{ item }} state=present groups=evens
	      with_sequence: start=0 end=32 format=testuser%02x
	
	    # create a series of directories with even numbers for some reason
	    - file: dest=/var/stuff/{{ item }} state=directory
	      with_sequence: start=4 end=16 stride=2
	
	    # a simpler way to use the sequence plugin
	    # create 4 groups
	    - group: name=group{{ item }} state=present
	      with_sequence: count=4


**随机选择**

`random_choice`可以从列表中随机取一个值。

	- debug: msg={{ item }}
	  with_random_choice:
	     - "go through the door"
	     - "drink from the goblet"
	     - "press the red button"
	     - "do nothing"


**Do-Until 循环**

	- action: shell /usr/bin/foo
	  register: result
	  until: result.stdout.find("all systems go") != -1
	  retries: 5
	  delay: 10

上边的例子重复执行shell模块，当标准输出中"all systems go"不等于-1的时候停止执行。重试5次，延迟时间10秒。`retries`默认值为3，`delay`默认值为5。

任务的返回值为最后一次循环的返回结果。


**循环注册变量**

在循环中使用`register`时，索保存的结果中包含`results`关键字，该关键字保存模块执行结果的列表

	- shell: echo "{{ item }}"
	  with_items:
	    - one
	    - two
	  register: echo

变量echo内容如下：

	{
	    "changed": true,
	    "msg": "All items completed",
	    "results": [
	        {
	            "changed": true,
	            "cmd": "echo \"one\" ",
	            "delta": "0:00:00.003110",
	            "end": "2013-12-19 12:00:05.187153",
	            "invocation": {
	                "module_args": "echo \"one\"",
	                "module_name": "shell"
	            },
	            "item": "one",
	            "rc": 0,
	            "start": "2013-12-19 12:00:05.184043",
	            "stderr": "",
	            "stdout": "one"
	        },
	        {
	            "changed": true,
	            "cmd": "echo \"two\" ",
	            "delta": "0:00:00.002920",
	            "end": "2013-12-19 12:00:05.245502",
	            "invocation": {
	                "module_args": "echo \"two\"",
	                "module_name": "shell"
	            },
	            "item": "two",
	            "rc": 0,
	            "start": "2013-12-19 12:00:05.242582",
	            "stderr": "",
	            "stdout": "two"
	        }
	    ]
	}

遍历注册变量的结果：

	- name: Fail if return code is not 0
	  fail:
	    msg: "The command ({{ item.cmd }}) did not have a 0 return code"
	  when: item.rc != 0
	  with_items: "{{echo.results}}"

示例：

	[root@web1 ~]# cat /etc/ansible/reloop.yml
	---
	- hosts: webservers
	  remote_user: root
	  tasks:
	   - shell: echo "{{ item }}"
	     with_items:
	      - one
	      - two
	     register: echo
	   - debug: msg="{{ echo }}"
	   - name: Fail if return code is not 0
	     fail: msg=" The command {{ item.cmd }} has a 0 return code"
	     when: item.rc != 0
	     with_items: "{{ echo.results }}"


	[root@web1 ~]# ansible-playbook /etc/ansible/reloop.yml
	
	PLAY [webservers] ************************************************************* 
	
	GATHERING FACTS *************************************************************** 
	ok: [192.168.1.65]
	
	TASK: [shell echo "{{ item }}"] *********************************************** 
	changed: [192.168.1.65] => (item=one)
	changed: [192.168.1.65] => (item=two)
	
	TASK: [debug msg="{{ echo }}"] ************************************************ 
	ok: [192.168.1.65] => {
	    "msg": "{'msg': 'All items completed', 'changed': True, 'results': [{u'stdout': u'one', u'changed': True, u'end': u'2015-08-05 11:30:42.560652', u'start': u'2015-08-05 11:30:42.549056', u'cmd': u'echo \"one\"', u'rc': 0, 'item': 'one', u'stderr': u'', u'delta': u'0:00:00.011596', 'invocation': {'module_name': u'shell', 'module_args': u'echo \"one\"'}, 'stdout_lines': [u'one'], u'warnings': []}, {u'stdout': u'two', u'changed': True, u'end': u'2015-08-05 11:30:44.105387', u'start': u'2015-08-05 11:30:44.093500', u'cmd': u'echo \"two\"', u'rc': 0, 'item': 'two', u'stderr': u'', u'delta': u'0:00:00.011887', 'invocation': {'module_name': u'shell', 'module_args': u'echo \"two\"'}, 'stdout_lines': [u'two'], u'warnings': []}]}"
	}
	
	TASK: [Fail if return code is not 0] ****************************************** 
	skipping: [192.168.1.65] => (item={u'cmd': u'echo "one"', u'end': u'2015-08-05 11:30:42.560652', u'stderr': u'', u'stdout': u'one', u'changed': True, u'rc': 0, 'item': 'one', u'warnings': [], u'delta': u'0:00:00.011596', 'invocation': {'module_name': u'shell', 'module_args': u'echo "one"'}, 'stdout_lines': [u'one'], u'start': u'2015-08-05 11:30:42.549056'})
	skipping: [192.168.1.65] => (item={u'cmd': u'echo "two"', u'end': u'2015-08-05 11:30:44.105387', u'stderr': u'', u'stdout': u'two', u'changed': True, u'rc': 0, 'item': 'two', u'warnings': [], u'delta': u'0:00:00.011887', 'invocation': {'module_name': u'shell', 'module_args': u'echo "two"'}, 'stdout_lines': [u'two'], u'start': u'2015-08-05 11:30:44.093500'})
	
	PLAY RECAP ******************************************************************** 
	192.168.1.65               : ok=4    changed=1    unreachable=0    failed=0

[该部分内容官方文档](http://docs.ansible.com/ansible/playbooks_loops.html)


## 最佳实践 ##


**目录布局**

目录和文件结构：

	production                # 生产环境的inventory文件
	staging                   # 测试环境的inventory文件
	
	group_vars/
	   group1                 # 为主机组定义的变量
	   group2                 # ""
	host_vars/
	   hostname1              # 主机变量
	   hostname2              # ""
	
	library/                  # 自定义的模块（可选）
	filter_plugins/           # 自定义的过滤器插件(可选)
	
	site.yml                  # 主playbooks
	webservers.yml            # webservers组的playbooks文件
	dbservers.yml             # dbserver的playbooks
	
	roles/
	    common/               # 角色，common下可以包含一些角色通用的操作
	        tasks/            #
	            main.yml      #  <-- 任务文件，也可以包含其他的任务文件
	        handlers/         #
	            main.yml      #  <-- handlers file
	        templates/        #  <-- 模板文件目录
	            ntp.conf.j2   #  <------- 模板文件以j2结尾
	        files/            #
	            bar.txt       #  <-- 用来复制的源文件
	            foo.sh        #  <-- 脚本文件
	        vars/             #
	            main.yml      #  <-- 该角色的变量文件，也可以包含其他变量文件
	        defaults/         #
	            main.yml      #  <-- 该角色的默认变量文件，优先级最低
	        meta/             #
	            main.yml      #  <-- 角色依赖关系
	
	    webtier/              # 另一个角色名称
	    monitoring/           # ""
	    fooapp/               # ""


**如何区分测试和生产环境**

在使用静态inventory文件的时候，要注意将不同环境内的主机区分开。为主机定义组名的时候，要做到见名知义.

	# file: production
	
	[atlanta-webservers]
	www-atl-1.example.com
	www-atl-2.example.com
	
	[boston-webservers]
	www-bos-1.example.com
	www-bos-2.example.com
	
	[atlanta-dbservers]
	db-atl-1.example.com
	db-atl-2.example.com
	
	[boston-dbservers]
	db-bos-1.example.com
	
	# webservers in all geos
	[webservers:children]
	atlanta-webservers
	boston-webservers
	
	# dbservers in all geos
	[dbservers:children]
	atlanta-dbservers
	boston-dbservers
	
	# everything in the atlanta geo
	[atlanta:children]
	atlanta-webservers
	atlanta-dbservers
	
	# everything in the boston geo
	[boston:children]
	boston-webservers
	boston-dbservers

**组和主机变量**

有时候只是使用组来组织主机也是不够的。可以将某些主机赋值给变量，如`atlanta`组有它自己的NTP服务器，当设置ntp.conf的时候，就可以使用这些变量了。

	---
	# file: group_vars/atlanta
	ntp: ntp-atlanta.example.com
	backup: backup-atlanta.example.com

还可以是以下的方式定义组变量：

	---
	# file: group_vars/webservers
	apacheMaxRequestsPerChild: 3000
	apacheMaxClients: 900

如果有些值是默认值或者是较为通用的，可以定义在`group_vars/all`文件内：

	---
	# file: group_vars/all
	ntp: ntp-boston.example.com
	backup: backup-boston.example.com

也可以在`host_vars`目录下为某个主机定义特定的变量，但是一般不建议这么做，除非是非常必要：

	---
	# file: host_vars/db-bos-1.example.com
	foo_agent_port: 86
	bar_agent_port: 99

[该部分官方文档](http://docs.ansible.com/ansible/playbooks_best_practices.html)

