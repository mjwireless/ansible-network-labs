# Network Lab Immersion Day Lab Outline
* Part 1: Getting setup
  * Fork Git Repo 
  * SSH to jump station 
  * Downloading forked Git Repo 
* Part 2: Explore the environment 
  * Explore the environment with shell and Ansible ad-hoc commands 
  * Investigate Ansible’s configuration and Inventory 
  * Connect to the Routers using Ansible 
* Part 3: Backup, Configure, and Explore the Routers facts 
  * Lab 1 
    * Directory structure review 
    * Understand and use facts 
    * Run initial playbook to put basic configuration on Router 
  * Lab 2 
    * Gather banner information and reconfigure the banner  
    * Backup router configuration  
    * Restore router configuration from files 
  * Lab 3 
    * Configure DNS and loopback interface  
    * Create GRE tunnel and setup routing 
    * Secure router by pushing secure configuration file to router  
    * Ping another student router from loopback interface 
  * Lab 4 
    * Access Ansible Automation Controller 
    * Build a backup job with scheduling 
    * Interact with a job containing a Survey 

## Part 1: Initial Setup

## Part 2: Explore the environment
### Overview
* Review the topology
* Modify your inventory file
* Review Ansible's configuration and Inventory files
* Use ansible ad-hoc commands to use the ping, ios_command and cli_command modules to connect to your router

### Installed tools:
  * `git` - command line tool to interact with git source control repositories.
  * `podman` - a name-spaced drop in replacement for the docker cli.
  * `nano` - a simple text editor.
  * `vim` - a powerful text editor.
  * `ansible-builder` - a command line wrapper to podman to build execution environments.
  * `ansible-navigator` - a powerful replacement to the old ansible-playbook command.
  * `ansible-vault` - a command line encryption tool to protect secrets for ansible-playbooks.
  * `tree` - a command line utilitiy to show recursive directory listings in a visual format.

    We will be editing text files throught the course of this lab.  Both `nano` and `vim` are available.  If you are inexperienced with `vim`,  `nano` will be the better choice, as it behaves more like a basic text editor.

### Topology
The topology is simple for the sake of learning some ansible basics. Traffic will traverse the R0 router and by the end you should be able to ping from your loopback interface to other loopback interfaces in the lab.  There will be GRE (Generic Routing Encapsulation) tunnel between every student router and R0 by the end which will allow traffic to be routed between devices.  The diagram below is an example, it shows the instructor router R0 and 2 student routers but there is a router for every student.

![lab-topology.png should be visible here](images/lab-topology.png)

### Connect to your jump host.
Connect to your jump host where ### is replaced by your user number.  
> **Note**: The password will be provided by the lab instructor.
```
>>  ssh siduser<###>@siduser<###>.jump.mysidlabs.com
Password: <<< password >>>
```
You should then see the following:
```bash
Welcome to the CDW/Sirius Red Hat Immersion Day lab environment
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
Cloning into 'ansible-network-labs'...
remote: Enumerating objects: 125, done.
remote: Counting objects: 100% (125/125), done.
remote: Compressing objects: 100% (69/69), done.
remote: Total 125 (delta 44), reused 111 (delta 32), pack-reused 0
Receiving objects: 100% (125/125), 22.19 KiB | 11.09 MiB/s, done.
Resolving deltas: 100% (44/44), done. 
siduser101@jump:~$ 
```

## Lab 1 - `ansible-navigator` and execution environments
1. Review the project structure. 
    ```bash
    siduser101@jump:~$ cd ansible-network-labs/
    siduser101@jump:~/ansible-network-labs$ ls
    1.0-router-facts.yml        2.1-banner.yml          32-secure-config.cfg   encrypt.yml
    1.1-facts2.yml              2.2-backup.yml          3.2-secure-config.yml  hosts
    1.2.1-adv-facts.yml         2.3-cli-backup.yml      4.0-tower-backup.yml   misc
    1.2.2-adv-facts-filter.yml  2.4-adv-cli-backup.yml  4.1-tower-snmp.yml     README.md
    1.3-init.yml                2.5-restore-backup.yml  all-hosts              tower-hosts
    1.4-user-setup.yml          3.0-dns-cfg.yml         ansible.cfg            vars.yml
    2.0-snmp.yml                3.1-gre.yml             ansible-navigator.yml
    siduser101@jump:~/ansible-network-labs$ 
    ```
2. Review your ansible.cfg
    ```bash
    siduser101@jump:~/ansible-network-labs$ vim ansible.cfg
    ```
    ```vim
    1 [defaults]
    2 deprecation_warnings         = False
    3 gathering                    = explicit
    4 retry_files_enabled          = False
    5 inventory                    = ~/ansible-network-labs/hosts
    6 timeout                      = 60
    7 forks                        = 50
    8 host_key_checking            = False
    9 
    10 [ssh_connection]
    11 ssh_args                     = -o ControlMaster=auto -o ControlPersist=30m
    12 ansible_ssh_common_args      = ' -oKexAlgorithms=+diffie-hellman-group1-sha1 -caes128-cbc'
    13 scp_if_ssh                   = True
    14 
    15 [paramiko_connection]
    16 host_key_auto_add            = True
    17 
    18 [persistent_connection]
    19 connect_timeout              = 60
    20 command_timeout              = 60
    
    ```
    > **INFO**: This is the configuration that ansible will use if you are in the ~/ansible-network-labs directory. If the inventory setting was missing this would tell you that ansible will default to /etc/ansible/hosts.
3. Explore your inventory file
    ```bash
    siduser101@jump:~/ansible-network-labs$ vim hosts
    ```
    ```vim
    1 [all: vars]
    2 ansible_user=ec2-user
    3 ansible_ssh_private_key_file=~/.ssh/network-key.pem
    4 ansible_port=22
    5 
    6 [routers: children]
    7 cisco
    8 juniper
    9 
    10 [cisco]
    11 R<studentID> ansible_host=10.1.<studentID>.10
    12 
    13 [cisco: vars]
    14 ansible_network_os=ios
    15 
    16 [juniper]
    17 
    18 [juniper: vars]
    19 ansible_network_os=junos
    20 
    21 [arista]
    22 
    23 [arista: vars]
    24 ansible_network_os=eos
    ~                                                                                                         
    ~                                                                                                         
    ~                                                                                                         
    ~                                                                                                         
    ~                                                                                                         
    <ansible-network-labs/hosts [FORMAT=unix] [TYPE=] [POS=1,1][4%] [BUFFER=1] Fri 26 Aug 2022 05:17:18 PM UTC
    :

    ```
    > **Note**:  Ansible uses ansible_network_os to inform Ansible which network platform this hosts corresponds too. This is required when using connection plugins network_cli or netconf. We will be using ansible_connection: network_cli later which is automatically combined with this.    
    
    Here for example ansible_network_os=ios tells Ansible the Networking Operating System is Cisco’s ios. Platforms supported by ansible_network_os include:    

    | Networking Platform | Ansible Network OS |
    |---|---|
    |Cisco IOS|ios|
    |Cisco IOS-XR|iosxr|
    |Cisco NX-OS|nxos|
    |Arista EOS|eos|
    |Juniper JUNOS|junos|
    |F5 Big IP|f5|

1. Modify your inventory file by updating your student router name to use your student id.
    > **Note**: The lab guide will be using vim to show these edits. But you may use *nano* or edit the file directly in github and then use `git pull` to update your local files.

    In vim this can be done by using a global search and replace:  Type `:` to enter the expression: `%s/<studentID>/101/g` and press enter.
    ```vim
    1 [all:vars]
    2 ansible_user=ec2-user
    3 ansible_ssh_private_key_file=~/.ssh/network-key.pem
    4 ansible_port=22
    5 
    6 [routers:children]
    7 cisco
    8 juniper
    9 
    10 [cisco]
    11 R101 ansible_host=10.1.101.10
    12 
    13 [cisco:vars]
    14 ansible_network_os=ios
    15 
    16 [juniper]
    17 
    18 [juniper:vars]
    19 ansible_network_os=junos
    20 
    21 [arista]
    22 
    23 [arista:vars]
    24 ansible_network_os=eos
    ~                                                                                                         
    ~                                                                                                         
    ~                                                                                                         
    ~                                                                                                         
    ~                                                                                                         
    <le-network-labs/hosts[+] [FORMAT=unix] [TYPE=] [POS=11,1][45%] [BUFFER=1] Fri 26 Aug 2022 05:40:14 PM UTC
    :%s/<studentID>/101/g

    ```
    Save the file  `:wq`

1. Validate that Ansible can see the hosts in your inventory.
    ```bash
    siduser101@jump:~/ansible-network-labs$ ansible all --list-hosts
    hosts (1):
        R101
    siduser101@jump:~/ansible-network-labs$
    ```
    > *TIP:* The ***all*** field in the previous command refers to the automatically created group ‘all’. You can retry the command using another group name or hostname instead. For example, ***cisco***,  ***routers*** or ***R101***.  Using Groups liberally in inventory files gives flexibility in the long run and is considered a Best Practice.

## Part 3: Investigate the Network Infrastructure
1. Check if the jump host can connect to the router.
    ```bash
    siduser101@jump:~/ansible-network-labs$ ping 10.1.101.10
    PING 10.1.101.10 (10.1.101.10) 56(84) bytes of data.
    64 bytes from 10.1.101.10: icmp_seq=1 ttl=255 time=0.585 ms
    64 bytes from 10.1.101.10: icmp_seq=2 ttl=255 time=0.576 ms
    64 bytes from 10.1.101.10: icmp_seq=3 ttl=255 time=0.558 ms
    ^C
    --- 10.1.101.10 ping statistics ---
    3 packets transmitted, 3 received, 0% packet loss, time 2041ms
    rtt min/avg/max/mdev = 0.558/0.573/0.585/0.011 ms
    ```
    > **Tip**:  Mac users:  control-c to stop the ping.  Windows user: ctrl+c to stop the ping.

1. Check if you can connect to your student router using an ansible ad-hoc command:
   > **Note**:  Type "yes" if/when prompted to continue connecting.

    ```bash
    siduser101@jump:~/ansible-network-labs$ ansible routers -m ping
    [WARNING]: Unhandled error in Python interpreter discovery for host R101: unexpected output from Python
    interpreter discovery
    [WARNING]: Platform unknown on host R101 is using the discovered Python interpreter at /usr/bin/python,
    but future installation of another Python interpreter could change the meaning of that path. See
    https://docs.ansible.com/ansible-core/2.13/reference_appendices/interpreter_discovery.html for more
    information.
    R101 | FAILED! => {
        "ansible_facts": {
            "discovered_interpreter_python": "/usr/bin/python"
        },
        "changed": false,
        "module_stderr": "Shared connection to 10.1.101.10 closed.\r\n",
        "module_stdout": "\r\n\r\n\r\nLine has invalid autocommand \"/bin/sh -c '/usr/bin/python '\"'\"'Line has invalid autocommand \"/bin/sh -c '\"'\"'\"'\"'\"'\"'\"'\"'( umask 77 && mkdir -p \"` echo Line has invalid autocommand \"/bin/sh -c '\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'echo ~ec2-user && sleep 0'\"'\"'\"'\"'\"'\"'\"\"",
        "msg": "MODULE FAILURE\nSee stdout/stderr for the exact error",
        "rc": 0
    }
    ```
    > *TIP:*  Ansible supports calling modules directly from the command line via the ansible command. These are called Ad-Hoc commands and are often used to establish connectivity as in here with the ping module. Another common use case is to inspect a host or group of hosts with fact gathering modules like ***setup***. You can optionally pass parameters to Ad-Hoc commands with the -a option: `ansible localhost -m debug -a "msg='passing a parameter'"`

    > *NOTE*:  The ping module, without additional parameters, is unable to successfully communicate with the Routers. By default, Ansible connections are handled transparently by ssh and no connection type needs to be set. This works well in a typical server environment but typically not for network devices. However, Ansible has a pluggable connection architecture which allows it to be extended to connect to these and other devices.  Other examples of connection: types include local when connecting to the localhost and winrm which allows Ansible to connect to Microsoft Windows platforms.

1. Retry the ansible ping again selecting the group routers and this time setting the connection type to network_cli
   ```bash
   siduser101@jump:~/ansible-network-labs$ ansible routers -m ping -c network_cli
   R101 | SUCCESS => {
        "changed": false,
        "ping": "pong"
    }
    siduser101@jump:~/ansible-network-labs$
    ```
    > **Note**: Here Ansible combines the connection type **-c network_cli** with the **ansible_network_os=ios** inventory variable and now knows how to successfully communicate with the network device.
    
    > ***TIP:***  Ansible allows you to pass variables like ansible_network_os on the command line with the -e option and this has the highest precedence, i.e. will override any variable you have set.   
    You could try setting this to an illegal value, for example:   
    ***ansible routers -m ping -c network_cli -e ansible_network_os=linux***    
    You can also try setting it to a different vendors network operating system such as junos, the result may at first be surprising. However, ‘network_cli’ uses similar mechanisms to connect to network devices so a misconfiguration here may not result in failure.

1. Modify hosts files (see previous lesson on editing host file in github and pulling the changes for reference if needed)
    1. Add ***ansible_connection=network_cli*** to hosts file in repository and commit the change.
        ```vim
        ...
        10 [cisco]
        11 R101 ansible_host=10.1.101.10
        12 
        13 [cisco:vars]
        14 ansible_network_os=ios
        15 ansible_connection=network_cli
        ...
        ```
1. Retry the ansible ping command again.
    ```bash
    siduser101@jump:~/ansible-network-labs$ ansible routers -m ping
    R101 | SUCCESS => {
        "changed": false,
        "ping": "pong"
    }
    siduser101@jump:~/ansible-network-labs$ 

    ```

<!---
### Using Ansible Vault with ansible-navigator
[https://github.com/ansible/ansible-navigator/blob/main/docs/faq.md]

There are two methods to expose a vault password for use with ansible-navigator:
1. Store the vault password securely on the local files system:
    ```bash
    $ touch ~/.vault_password
    $ chmod 600 ~/.vault_password
    # The leading space here is necessary to keep the command out of the command history
    $  echo my_password >> ~/.vault_password
    # Link the password file into the current working directory
    $ ln ~/.vault_password .
    # Set the environment variable to the location of the file
    $ export ANSIBLE_VAULT_PASSWORD_FILE=.vault_password
    ```
2. Store the vault password in an environment variable
    ```bash
    $ touch ~/.vault_password.sh
    $ chmod 700 ~/.vault_password.sh
    $ echo -e '#!/bin/sh\necho ${ANSIBLE_VAULT_PASSWORD}' >> ~/.vault_password.sh
    # Link the password file into the current working directory
    $ ln ~/.vault_password.sh .
    # The leading space here is necessary to keep the command out of the command history
    # by using an environment variable prefixed with ANSIBLE it will automatically get passed
    # into the execution environment
    $  export ANSIBLE_VAULT_SECRET=my_password
    # Set the environment variable to the location of the file
    $ ANSIBLE_VAULT_PASSWORD_FILE=.vault_password.sh
    ```

Execute the following command to decrypt the group_vars/all.yml file:
```bash
    siduser101@jump:~/sid-ansible-security$ ansible-vault decrypt group_vars/all.yml 
    [DEPRECATION WARNING]: Ansible will require Python 3.8 or newer on the controller starting with Ansible 2.12. Current 
    version: 3.6.8 (default, Mar 18 2021, 08:58:41) [GCC 8.4.1 20200928 (Red Hat 8.4.1-1)]. This feature will be removed 
    from ansible-core in version 2.12. Deprecation warnings can be disabled by setting deprecation_warnings=False in 
    ansible.cfg.
    Vault password: Ger1974!
    Decryption successful
    siduser101@jump:~/sid-ansible-security$
```
-->




## Part 3 - Explore, configure and backup router using playbooks
### Lab 1 - Overview
* Direcory structure review
* ***Execution Environments*** and the new `ansible-navigator` command line tool.
* Understand and use facts
* Run initial playbook to load basic configuration to router
  
### Use `podman`, `ansible-builder` and `ansible-navigator` to work with execution environments
The Red Hat Ansible Automation Platform version 2.0 and above used *execution environment* containers to package and distribute automation dependencies in a portable way.:
Type the following at the command line:
```bash
siduser101@jump:~/ansible-network-labs$ podman images
REPOSITORY  TAG         IMAGE ID    CREATED     SIZE
siduser270@jump:~/sid-ansible-security$ 
```
Notice that there are currently no local images.
Type the `ansible-navigator` command, first making sure that you are in the root of the ansible-network-labs project:
```bash
siduser101@jump:~/ansible-network-labs$ ansible-navigator
--------------------------------------------------------------------------------------------------------------
Execution environment image and pull policy overview
--------------------------------------------------------------------------------------------------------------
Execution environment image name:     registry.redhat.io/ansible-automation-platform-22/ee-supported-rhel8:latest
Execution environment image tag:      latest
Execution environment pull arguments: None
Execution environment pull policy:    missing
Execution environment pull needed:    True
--------------------------------------------------------------------------------------------------------------
Updating the execution environment
--------------------------------------------------------------------------------------------------------------
Running the command: podman pull registry.redhat.io/ansible-automation-platform-22/ee-supported-rhel8:latest
Trying to pull registry.redhat.io/ansible-automation-platform-22/ee-supported-rhel8:latest...
Getting image source signatures
Checking if image destination supports signatures
Copying blob 50223de3f59a done  
Copying blob a96e4e55e78a done  
Copying blob 67d8ef478732 done  
Copying blob bddb6822d87e done  
Copying blob a9dacd757072 done  
Copying config d2595109e4 done  
Writing manifest to image destination
Storing signatures
d2595109e44f42bd917121949c925a64b50bf8fddfa482ed65f47be4adaee770
```
Since this is our first execution of ansible-navigator it has pulled the execution environment specified in the `ansible-navigator.yml` file which is the configuration file for this tool.  Once the pull has completed the text-based UI for ansible navigator will launch.
```bash
 0│Welcome                                                                                             ▒
 1│————————————————————————————————————————————————————————————————————————————————————————————————————▒
 2│                                                                                                    ▒
 3│Some things you can try from here:                                                                  ▒
 4│- :collections                                    Explore available collections                     ▒
 5│- :config                                         Explore the current ansible configuration         ▒
 6│- :doc <plugin>                                   Review documentation for a module or plugin       ▒
 7│- :help                                           Show the main help page                           ▒
 8│- :images                                         Explore execution environment images              ▒
 9│- :inventory -i <inventory>                       Explore an inventory                              ▒
10│- :log                                            Review the application log                        ▒
11│- :lint <file or directory>                       Lint Ansible/YAML files (experimental)            ▒
12│- :open                                           Open current page in the editor                   ▒
13│- :replay                                         Explore a previous run using a playbook artifact  ▒
14│- :run <playbook> -i <inventory>                  Run a playbook in interactive mode                ▒
15│- :settings                                       Review the current ansible-navigator settings     ▒
16│- :quit                                           Quit the application                              ▒
17│                                                                                                    ▒
18│happy automating,                                                                                   ▒
19│                                                                                                    ▒
^b/PgUp page up         ^f/PgDn page down         ↑↓ scroll         esc back         :help help

```

> **Note**: Follow along with the instructor as he reviews the `ansible-navigator.yml` config and also looks at some of the sub-commands of the tool.


Use `ansible-navigator` to inspect new image:
```bash
siduser101@jump:~/sid-ansible-security$ ansible-navigator images
```

```
  NAME                             TAG      EXECUTION ENVIRONMENT	CREATED         SIZE
0│sid-security-ee (primary)        latest                    True	2 months ago    681 MB






^f/PgUp page up    ^b/PgDn page down    ↑↓ scroll    esc back    [0-9] goto    :help help

```

> **Note**: Again, follow along with the instructor as he explores `sid-security-ee` execution environment.

### Review execution environments
1. Review ee project files
1. Review [`podman`](https://https://podman.io/) vs. `docker`
1. Review [`ansible-buider`](https://www.ansible.com/blog/introduction-to-ansible-builder)
1. Build sec-sid-mitigation-ee

> **OPTIONAL**: building this execution environment is optional.  It will take a couple of minutes to complete.
```bash
    ansible-builder build --tag sec_sid_ee
```
### Use `podman` to list the local images:
```bash
siduser101@jump:~/sid-ansible-security$ podman images
REPOSITORY                                 TAG         IMAGE ID      CREATED            SIZE
<none>                                     <none>      cc8028f10813  About an hour ago  907 MB
<none>                                     <none>      a04cc3797cca  About an hour ago  913 MB
<none>                                     <none>      774fabe4d536  About an hour ago  881 MB
<none>                                     <none>      0643cb081bc6  About an hour ago  896 MB
localhost/sid-security-ee                  latest      9ad85776c89f  About an hour ago  908 MB
<none>                                     <none>      ee55b78317df  About an hour ago  791 MB
<none>                                     <none>      c5809904a915  About an hour ago  891 MB
quay.io/ansible/ansible-runner             latest      912ba7432e89  7 hours ago        876 MB
quay.io/ansible/ansible-builder            latest      35d2481da9e9  8 hours ago        769 MB
ghcr.io/mysidlabs/sid-security-ee          latest      7d187fedb8ea  2 months ago       681 MB
quay.io/ansible/ansible-navigator-demo-ee  0.6.0       e65e4777caa3  5 months ago       1.35 GB
```




