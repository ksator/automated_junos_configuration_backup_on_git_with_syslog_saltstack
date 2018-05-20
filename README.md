# Demo overview:  

At each junos commit, SaltStack automatically collects the new junos configuration file and archives it to a git server: 
- When a Junos commit is completed, the Junos device send a syslog message ```UI_COMMIT_COMPLETED```.  
- The junos devices are configured to send this syslog message to SaltStack.  
- Each time SaltStack receives this syslog message, SaltStack automatically collects the new junos configuration file from the
JUNOS device that send this commit syslog message, and SaltStack automatically archives the new junos configuration file to a git server  

[!continous_backup.png](continous_backup.png)  

# Demo building blocks: 


- Juniper devices
- SaltStack


# Building blocks role: 

## Junos devices: 
- They are configured to send the syslog message ```UI_COMMIT_COMPLETED``` to SaltStack when a commit is completed

## SaltStack: 
- In addition to the Salt master, Salt Junos proxy minions are required (one process per Junos device is required)  
- The Salt master listens to syslog messages sent by junos devices
- The Salt master generates a ZMQ messages to the event bus when a junos syslog message is received. The ZMQ message has a tag and data. The data structure is a dictionary, which contains information about the event.
- The Salt reactor binds sls files to event tags. The reactor has a list of event tags to be matched, and each event tag has a list of reactor SLS files to be run. So these sls files define the SaltStack reactions.
- The sls reactor file used in this content does the following: it parses the data from the ZMQ message to extract the network device name. It then ask to the Junos proxy minion that manages the "faulty" device to execute an sls file.
- The sls file executed by the Junos proxy minion collects the new junos configuration and archives the collected data to a git server  

## Gitlab  
- This SaltStack setup uses a gitlab server for external pillars (variables) and as a remote file server (templates, sls files, ...).  

# Gitlab

This SaltStack setup uses a gitlab server for external pillars and as a remote file server.  

## Install Gitlab

There is a Gitlab docker image available https://hub.docker.com/r/gitlab/gitlab-ce/

You first need to install docker. This step is not covered by this documentation.  

Then:  

Pull the image: 
```
# docker pull gitlab/gitlab-ce
```

Verify: 
```
# docker images
REPOSITORY                   TAG                 IMAGE ID            CREATED             SIZE
gitlab/gitlab-ce             latest              09b815498cc6        6 months ago        1.33GB
```

Instanciate a container: 
```
docker run -d --rm --name gitlab -p 9080:80 gitlab/gitlab-ce
```
Verify:
```
# docker ps
CONTAINER ID        IMAGE                        COMMAND                  CREATED             STATUS                PORTS                                                 NAMES
9e8330425d9c        gitlab/gitlab-ce             "/assets/wrapper"        5 months ago        Up 5 days (healthy)   443/tcp, 0.0.0.0:3022->22/tcp, 0.0.0.0:9080->80/tcp   gitlab
```
## Configure Gitlab

Create the organization ```organization```.    
Create the repositories ```network_parameters``` and ```network_model``` in the organization ```organization```.      
The repository ```network_parameters``` is used for SaltStack external pillars.    
The repository ```network_model``` is used as an external file server for SaltStack   


# SaltStack 

## Install SaltStack 

This is not covered by this documentation. 

You need a  master and a minion.  

The Salt Junos proxy has some requirements (```junos-eznc``` python library and other dependencies). Install on the master or on a minion the dependencies to use a SaltStack proxy for Junos. You need to install these dependencies on each node (master/minion) that will run a junos proxy daemon(s).  

You need one junos proxy daemon per device. Start one junos proxy daemon per device.  

## Run basic tests 

Run this command on the master to check the accepted keys: 
```
salt-key -L
```

Run this command on the master to make sure a minion is up and responding to the master. This is not an ICMP ping. 
Example for the minion ```svl-util-01```
```
salt svl-util-01 test.ping
```

Select a junos proxy daemon and run these additionnal tests.  
Example with the junos proxy ```core-rtr-p-02``` (it manages the network device ```core-rtr-p-02```)
```
salt core-rtr-p-02 test.ping
```
```
salt core-rtr-p-02 junos.cli "show version"
```
## Edit the SaltStack master configuration file 

Edit the salt master configuration file:  
```
vi /etc/salt/master
```
Make sure the master configuration file has these details:  

```
engines:
  - junos_syslog:
      port: 516
```
```
ext_pillar:
  - git:
    - master git@gitlab:organization/network_parameters.git
```
```
fileserver_backend:
  - git
  - roots
```
```
gitfs_remotes:
  - ssh://git@gitlab/organization/network_model.git
```
```
file_roots:
  base:
    - /srv/salt
    - /srv/local
```

So: 
- the Salt master is listening junos syslog messages on port 516. For each junos syslog message received, it generates an equivalent ZMQ message and publish it to the event bus
- external pillars (variables) are in the gitlab repository ```organization/network_parameters``` (master branch)
- Salt uses the gitlab repository ```organization/network_model``` as a remote file server.  

## SaltStack Git execution module basic demo

ssh to the Salt master.

On the Salt master, list all the keys.
```
salt-key -L
```
The below commands are run from the master.  
Most of these commands are using the Git execution module.  
So the master is asking to the minion ```core-rtr-p-01``` to execute these commands.    
```
# salt core-rtr-p-01 git.clone /tmp/local_copy git@github.com:JNPRAutomate/automated_junos_configuration_backup_with_syslog_saltstack.git identity="/root/.ssh/id_rsa"
core-rtr-p-01:
    True

# salt core-rtr-p-01 cmd.run "ls /tmp/local_copy"
core-rtr-p-01:
    README.md
    ...
    ...

# salt core-rtr-p-01 git.config_set user.email me@example.com cwd=/tmp/local_copy
core-rtr-p-01:
    - me@example.com

# salt core-rtr-p-01 git.config_set user.name ksator cwd=/tmp/local_copy
core-rtr-p-01:
    - ksator
    
# salt core-rtr-p-01 git.config_get user.name cwd=/tmp/local_copy
core-rtr-p-01:
    ksator

# salt core-rtr-p-01 git.pull /tmp/local_copy
core-rtr-p-01:
    Already up-to-date.

# salt core-rtr-p-01 file.touch "/tmp/local_copy/test.txt"
core-rtr-p-01:
    True

# salt core-rtr-p-01 file.write "/tmp/local_copy/test.txt" "hello from SaltStack using git executiom module"
core-rtr-p-01:
    Wrote 1 lines to "/tmp/local_copy/test.txt"

# salt core-rtr-p-01 cmd.run "more /tmp/local_copy/test.txt"
core-rtr-p-01:
    ::::::::::::::
    /tmp/local_copy/test.txt
    ::::::::::::::
    hello from SaltStack using git executiom module

# salt core-rtr-p-01 git.status /tmp/local_copy
core-rtr-p-01:
    ----------
    untracked:
        - test.txt

# salt core-rtr-p-01 git.add /tmp/local_copy /tmp/local_copy/test.txt
core-rtr-p-01:
    add 'test.txt'

# salt core-rtr-p-01 git.status /tmp/local_copy
core-rtr-p-01:
    ----------
    new:
        - test.txt

# salt core-rtr-p-01 git.commit /tmp/local_copy 'The commit message'
core-rtr-p-01:
    [master 60f5943] The commit message
     1 file changed, 1 insertion(+)
     create mode 100644 test.txt

# salt core-rtr-p-01 git.status /tmp/local_copy
core-rtr-p-01:
    ----------

# salt core-rtr-p-01 git.push /tmp/local_copy origin master identity="/root/.ssh/id_rsa"
core-rtr-p-01:
```
The above commands pushed the file [test.txt](test.txt) to this repository  

## Test your Junos proxy daemons

### Run these basic tests

ssh to the Salt master.

On the Salt master, list all the keys. 
```
# salt-key -L
```
Run this command on the master to check if a minion is up and responding to the master. This is not an ICMP ping.  
Example with the junos proxy ```core-rtr-p-01``` (it manages the network device ```core-rtr-p-01```)  
```
# salt core-rtr-p-01 test.ping
core-rtr-p-01:
    True
```
List the grains: 
```
# salt core-rtr-p-01 grains.ls
...
```
Get the value of the grain ```nodename```. 
```
# salt core-rtr-p-01 grains.item nodename
core-rtr-p-01:
    ----------
    nodename:
        svl-util-01
```
So, the junos proxy daemon ```core-rtr-p-01``` is running on the minion ```svl-util-01```  

The Salt Junos proxy has some requirements (```junos-eznc``` python library and other dependencies).
```
# salt svl-util-01 cmd.run "pip list | grep junos"
svl-util-01:
    junos-eznc (2.1.7)
```

### Junos execution modules

Run this command on the master to ask to a proxy to use a Junos execution module:   
```
# salt core-rtr-p-01 junos.cli "show chassis hardware"
core-rtr-p-01:
    ----------
    message:

        Hardware inventory:
        Item             Version  Part number  Serial number     Description
        Chassis                                VM5AA80D5BB2      VMX
        Midplane
        Routing Engine 0                                         RE-VMX
        CB 0                                                     VMX SCB
        FPC 0                                                    Virtual FPC
          CPU            Rev. 1.0 RIOT-LITE    BUILTIN
          MIC 0                                                  Virtual
            PIC 0                 BUILTIN      BUILTIN           Virtual
    out:
        True
```
### Junos state modules 

  The files [collect_show_commands_example_1.sls](collect_show_commands_example_1.sls) and [collect_show_commands_example_2.sls](collect_show_commands_example_2.sls) use a diff syntax but they are equivalents.  

#### Syntax 1

```
# more /srv/salt/collect_junos_show_commands_example_1.sls
show version:
  junos:
    - cli
    - dest: /tmp/show_version.txt
    - format: text
show chassis hardware:
  junos:
    - cli
    - dest: /tmp/show_chassis_hardware.txt
    - format: text
```
Run this command on the master to ask to the proxy core-rtr-p-01 to execute the sls file  [collect_show_commands_example_1.sls](collect_show_commands_example_1.sls).
```
# salt core-rtr-p-01 state.apply collect_show_commands_example_1
```

#### Syntax 2
```
# more /srv/salt/collect_show_commands_example_2.sls
show_version:
  junos.cli:
    - name: show version
    - dest: /tmp/show_version.txt
    - format: text
show_chassis_hardware:
  junos.cli:
    - name: show chassis hardware
    - dest: /tmp/show_chassis_hardware.txt
    - format: text
```
Run this command on the master to ask to the proxy ```core-rtr-p-01``` to execute the sls file  [collect_show_commands_example_2.sls](collect_show_commands_example_2.sls).
```
# salt core-rtr-p-01 state.apply collect_show_commands_example_2
```
## sls file to collect junos show commands and to archive the output to git

The file [collect_data_and_archive_to_git.sls](collect_data_and_archive_to_git.sls) collectes data from junos devices (show commands) and archive the data collected on a git server  

Add the file [collect_data_and_archive_to_git.sls](collect_data_and_archive_to_git.sls) in the ```junos``` directory of the ```organization/network_model``` gitlab repository (```gitfs_remotes```) .  

## Pillars 

Here's an example for the ```top.sls``` file at the root of the gitlab repository ```organization/network_parameters``` (```ext_pillar```)  
```
{% set id = salt['grains.get']('id') %} 
{% set host = salt['grains.get']('host') %} 

base:
  '*':
    - production
   
{% if host == '' %}
  '{{ id }}':
    - {{ id }}
{% endif %}
```


The pillar ```data_collection``` is used by the file [collect_data_and_archive_to_git.sls](collect_data_and_archive_to_git.sls)  
The pillar ```data_collection``` is used to maintain the list of show commands we want SaltStack to run automatically based on events.  
Update the file ```production.sls``` in the repository ```organization/network_parameters``` (```ext_pillar```) to define the pillar ```data_collection``` 
```
data_collection:  
   - command: show configuration
   - command: show system commit
   - command: show configuration | compare rollback 1
```

## Test your automation content manually from the master

Example with the proxy ```core-rtr-p-02``` (it manages the network device ```core-rtr-p-02```).   
Run this command on the master to ask to the proxy ```core-rtr-p-02``` to execute it.  
```
salt core-rtr-p-02 state.apply junos.collect_data_and_archive_to_git
```

The data collected by the proxy ```core-rtr-p-02``` is archived in the directory [core-rtr-p-02](core-rtr-p-02)  

##  Update the Salt reactor

The reactor binds sls files to event tags. The reactor has a list of event tags to be matched, and each event tag has a list of reactor SLS files to be run. So these sls files define the SaltStack reactions.  
Update the reactor.  
```
# more /etc/salt/master.d/reactor.conf
reactor: 
    - 'jnpr/syslog/*/UI_COMMIT_COMPLETED':
       - /srv/reactor/automate_show_commands.sls
```
This reactor binds ```jnpr/syslog/*/UI_COMMIT_COMPLETED``` to ```/srv/reactor/automate_show_commands.sls```  

Restart the Salt master:
```
service salt-master stop
service salt-master start
```

The command ```salt-run reactor.list``` lists currently configured reactors:  
```
salt-run reactor.list
```

Create the sls file ```/srv/reactor/automate_show_commands.sls```.  
```
# more /srv/reactor/automate_show_commands.sls
{% if data['data'] is defined %}
{% set d = data['data'] %}
{% else %}
{% set d = data %}
{% endif %}

automate_show_commands:
  local.state.apply:
    - tgt: "{{ d['hostname'] }}"
    - arg:
      - junos.collect_data_and_archive_to_git
```
The file [automate_show_commands.sls](automate_show_commands.sls) parses the data from the ZMQ message that has the tags ```jnpr/syslog/*/UI_COMMIT_COMPLETED``` and extracts the network device name and asks to the Junos proxy minion that manages the device that send this syslog message to apply the file ```junos/collect_data_and_archive_to_git.sls```.  

The file [collect_data_and_archive_to_git.sls](collect_data_and_archive_to_git.sls) executed by the Junos proxy minion is located in the ```junos``` directory of the ```organization/network_model``` gitlab repository (```gitfs_remotes```). It collects show commands and archives the data collected to a git server.  

The list of junos commands to collect is maintained with the variable ```data_collection```.  The file [collect_data_and_archive_to_git.sls](collect_data_and_archive_to_git.sls) use the pillar ```data_collection```.  This variable is in the gitlab repository ```organization/network_parameters``` (```ext_pillar```) 

# Junos devices 

The Salt master is listening junos syslog messages on port 516.  

Configure your junos devices to send the sylog message ```UI_COMMIT_COMPLETED``` to the SaltStack master ip address on port 516.  
Example for the device ```core-rtr-p-02```  
```
# salt core-rtr-p-02 junos.cli "show configuration system syslog host 192.168.128.174 | display set"
core-rtr-p-02:
    ----------
    message:

        set system syslog host 192.168.128.174 any any
        set system syslog host 192.168.128.174 match UI_COMMIT_COMPLETED
        set system syslog host 192.168.128.174 port 516
    out:
        True

```

# Run the demo 

## Watch syslog messages and ZMQ messages  

Run this command on the master to see the syslog messages sent by junos devices:  
```
# tcpdump port 516 -XX 
```

Salt provides a runner that displays events in real-time as they are received on the Salt master.  
Run this command on the master:   
```
# salt-run state.event pretty=True
```

## Trigger a syslog message from a junos device 

```
root@core-rtr-p-02# set system login message "welcome to this demo"

[edit]
root@core-rtr-p-02# commit and-quit
commit complete
Exiting configuration mode

root@core-rtr-p-02>

```
The junos device sent a syslog message ```UI_COMMIT_COMPLETED``` to SaltStack. SaltStack rans show commands on this device to collect the new junos configuration and other dettails and archived the data collected on a git server.   

## Verify on the git server 

The data collected by the proxy ```core-rtr-p-02```  is archived in the directory [core-rtr-p-02](core-rtr-p-02)
