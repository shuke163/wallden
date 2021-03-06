# ansible
-------------

一、Ansible小结：
Tip1： 
当需要代理连接网络时，比如使用get_url模块时，采用的关键字environment: “{{proxy}}”，proxy信息定义在vars中，注意environment后面的值必须是dictionary。
代码示例：
- hosts: wanhaha
  become: yes
  vars:
    proxy01:
      http_proxy: "http://192.168.30.5:8080"
  tasks:
  - name: install pgdg-centos92-9.2-2.noarch.rpm
    yum: name=http:……/pgdg-centos92-9.2-2.noarch.rpm
         state=present
    environment: "{{ proxy01 }}"• 1

Tip2： 
在使用lineinfile模块修改指定行内容时，要考虑到playbook重复执行时，可能会重复插入内容，使用regex时，要注意配合正则指定好需要修改的line，让playbook在重复执行时，也能找到指定line。
代码示例：
    - name: modify /etc/ntp.conf
      lineinfile: dest=/etc/ntp.conf 
                  regexp="{{item.regexp}}"
                  line="{{item.line}}"
      with_items:
        - { regexp: '^restrict',line: 'restrict default ignore' }
        - { regexp: 'restrict',line: 'restrict 127.0.0.1'}• 1

Tip3： 
使用when来指定该task执行时需要满足的一些条件，比如ansible_distribution_major_version、inventory_hostname、ansible_ssh_host等，这些都是ansible的内置变量，也可以自己定义变量。
代码示例：
when: ansible_ssh_host != "vagrant1" and ansible_ssh_host != "vagrant2"

when: ansible_ssh_host == "vagrant1" or ansible_ssh_host == "vagrant2"• 1

Tip4： 
使用shell启动多个service时，当task运行成功时，退出后实际没有启动成功，需要考虑使用nohup命令(或者命令最后加上&)，该命令可以在你退出帐户/关闭终端之后继续运行相应的进程。
代码示例：
- name: start PCMI service
      shell: "nohup /var/opt/something start && touch {{ target_dir }}/something.done"
      args:
        creates: "{{ target_dir }}/something.done"• 1

Tip5： 
因为task执行只会输出成功changed或者ok或者error详细，不会输出执行命令的输出内容，可以使用register关键字和debug模块，输出前一个task的结果。
代码示例：
      - name: check service ststus
        shell: systemctl status jobarg-monitor
        register: jobarg_monitor_status

      - debug: var=jobarg_monitor_status.stdout_lines• 1


二、serverspec的使用
这个spec主要检测playbook自动化执行的内容，是否都成功了。官网上对于语法讲解的很详细，比较简单。 
安装不讲了，直接看官网链接。 
配置在官网上没找到详细的内容，这边记录一下： 
（1）修改hosts.json，这个文件是根据Rakefile中的代码找到的，Rakefile根据hosts.json中的内容，获取需要spec的host信息。 
（2）在Rakefile指定不同的host需要执行的spec文件的路径。 
（3）在spec/spec_helper.rb中修改host对应的ip
serverspec不是很常用，需要的配置文件比较多，代码就不贴了。。。

