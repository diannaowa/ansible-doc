# Ansible开发者相关 #

@author 刘振伟

@qq 570962906

当业务比较大比较复杂的时候，单纯的使用Ansible有时候不会很好的完成相关的运维工作，这个时候就需要开发针对自己业务的一些模块或者ansible插件来完成这些工作。ansible还提供了python接口，可以使用python开发更为自动化的运维管理系统。


## Python API ##

Ansible的Python API使用起来还是比较简单方便的，下边是一个较为简单的实现：

	import ansible.runner
	
	runner = ansible.runner.Runner(
	   module_name='ping',
	   module_args='',
	   pattern='web*',
	   forks=10
	)
	datastructure = runner.run()

`run()`方法会返回执行的结果。返回的数据是一个JSON格式的：

	{
	    "dark" : {
	       "web1.example.com" : "failure message"
	    },
	    "contacted" : {
	       "web2.example.com" : 1
	    }
	}
示例：

	[root@web1 ~]# cat an.py
	#!/usr/bin/env python
	
	import ansible.runner
	
	runner = ansible.runner.Runner(
		module_name='ping',#调用的模块
	    module_args='',#模块参数
	    pattern='webservers',#主机组，可以是正则表达式如web*
	    forks=10
	)
	
	data = runner.run()
	print data

	[root@web1 ~]# python an.py 
	{'dark': {}, 'contacted': {'192.168.1.65': {'invocation': {'module_name': 'ping', 'module_args': ''}, u'changed': False, u'ping': u'pong'}}}

示例2：

	#!/usr/bin/python
	
	import ansible.runner
	import sys
	
	# construct the ansible runner and execute on all hosts
	results = ansible.runner.Runner(
	    pattern='*', forks=10,
	    module_name='command', module_args='/usr/bin/uptime',
	).run()
	
	if results is None:
	   print "No hosts found"
	   sys.exit(1)
	
	print "UP ***********"
	for (hostname, result) in results['contacted'].items():
	    if not 'failed' in result:
	        print "%s >>> %s" % (hostname, result['stdout'])
	
	print "FAILED *******"
	for (hostname, result) in results['contacted'].items():
	    if 'failed' in result:
	        print "%s >>> %s" % (hostname, result['msg'])
	
	print "DOWN *********"
	for (hostname, result) in results['dark'].items():
	    print "%s >>> %s" % (hostname, result)

默认使用的主机资源文件位置为/etc/ansible/hosts,也可以指定`host_list`参数，来指定一个inventory文件的路径。

## 动态主机文件 ##

当是用`--list`参数访问动态主机脚本的时候，脚本必须以JSON格式返回所有的主机组，每个组都应该包含字典形式的主机列表，子组列表，如果需要的话还应该组变量，最简单的信息是只包含主机列表：

	{
	    "databases"   : {
	        "hosts"   : [ "host1.example.com", "host2.example.com" ],
	        "vars"    : {
	            "a"   : true
	        }
	    },
	    "webservers"  : [ "host2.example.com", "host3.example.com" ],
	    "atlanta"     : {
	        "hosts"   : [ "host1.example.com", "host4.example.com", "host5.example.com" ],
	        "vars"    : {
	            "b"   : false
	        },
	        "children": [ "marietta", "5points" ]
	    },
	    "marietta"    : [ "host6.example.com" ],
	    "5points"     : [ "host7.example.com" ]
	}

database组包含了hosts列表，组变量的信息。webservers组只有主机列表。atlanta组有主机列表，组变量和子组列表。


当使用`--host <hostname>`访问脚本的时候，应该返回该主机的变量列表，或者是返回一个空的字典。

	{
	    "favcolor"   : "red",
	    "ntpserver"  : "wolf.example.com",
	    "monitoring" : "pack.example.com"
	}

但是如果针对每个主机都调用一次`--host`的话，将会产生大量的资源开销。在Ansible1.3或之后的版本中，如果脚本返回了一个称为`_meta`的顶级元素，该元素中如果包含着`hostvars`关键字，那么就可以将所有主机的变量都返回。脚本也不会再为每个主机都调用一次`--host`,这讲节省大量的资源开销，同时也方便的客户端的缓存。
返回的`_meta`类似如下格式：

	{
	
	    # results of inventory script as above go here
	    # ...
	
	    "_meta" : {
	       "hostvars" : {
	          "moocow.example.com"     : { "asdf" : 1234 },
	          "llama.example.com"      : { "asdf" : 5678 },
	       }
	    }
	
	}

所返回的变量可以在模板中引用。

示例：

动态脚本内容

	[root@web1 ~]# cat /etc/ansible/getHosts.py 
	#!/usr/bin/python
	import argparse
	try:
		import json
	except ImportError:
		import simplejson as json
	
	'''这里是模拟数据，工作上一般该数据都是从数据库或者缓存中读取的'''
	mockData = {
		"webservers":{
			"hosts": ["192.168.1.65"],
			"vars":{
				"http_port":8888,
				"max_clients":789
			}
		},
	
		"databases":{
			"hosts":["192.168.1.65"],
			"vars":{
				"action":"Restart MySQL server."
			}
		}
	}
	'''模拟数据结束'''

	def getList():
		'''get list hosts group'''
		print json.dumps(mockData)
	
	
	def getVars(host):
		'''Get variables about a specific host'''
		print json.dumps(mockData[host]["vars"])
	
	
	if __name__ == "__main__":
	
		parser = argparse.ArgumentParser()
		parser.add_argument('--list',action='store_true',dest='list',help='get all hosts')
		parser.add_argument('--host',action='store',dest='host',help='get all hosts')
		args = parser.parse_args()
	
		if args.list:
			getList()
	
		if args.host:
			getVars(args.host)

playbook内容：

	[root@web1 ~]# cat /etc/ansible/test.yml 
	---
	- hosts: webservers
	  remote_user: root
	  tasks:
	    - name: config httpd.file
	      template: src=/etc/ansible/httpd.j2 dest=/etc/httpd.conf

执行：

	[root@web1 ansible]# ansible-playbook -i /etc/ansible/getHosts.py /etc/ansible/test.yml
	
	PLAY [webservers] ************************************************************* 
	
	GATHERING FACTS *************************************************************** 
	ok: [192.168.1.65]
	
	TASK: [config httpd.file] ***************************************************** 
	changed: [192.168.1.65]
	
	PLAY RECAP ******************************************************************** 
	192.168.1.65               : ok=2    changed=1    unreachable=0    failed=0   
	
	[root@web1 ansible]# ansible -i /etc/ansible/getHosts.py databases -m shell -a "echo {{ action }}"192.168.1.65 | success | rc=0 >>
	Restart MySQL server.

Python API调用动态inventory:

	[root@web1 ~]# cat an.py 
	#!/usr/bin/env python
	
	import ansible.runner
	
	runner = ansible.runner.Runner(
		module_name='shell',
	    module_args='echo {{ action }}',
	    pattern='databases',
		host_list='/etc/ansible/getHosts.py',
	    forks=10
	)
	
	data = runner.run()
	print data

	[root@web1 ~]# python an.py 
	{'dark': {}, 'contacted': {u'192.168.1.65': {u'cmd': u'echo Restart MySQL server.', u'end': u'2015-08-07 15:26:02.141350', u'stdout': u'Restart MySQL server.', u'changed': True, u'start': u'2015-08-07 15:26:02.128109', u'delta': u'0:00:00.013241', u'stderr': u'', u'rc': 0, 'invocation': {'module_name': 'shell', 'module_args': u'echo Restart MySQL server.'}, u'warnings': []}}}



## 自定义模块开发 ##

