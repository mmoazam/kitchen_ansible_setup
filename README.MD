
# Tutorial by Kehinde Sanni
https://www.alibabacloud.com/blog/test-your-ansible-deployment-with-inspec-and-kitchen-on-alibaba-cloud_595438

 Test Your Ansible Deployment with Inspec and Kitchen on Alibaba Cloud
Alibaba Clouder October 12, 2019 13,640 0
In this tutorial, you'll learn how to write tests for your ansible playbooks on Alibaba Cloud using Inspec and Kitchen.

By Kehinde Sanni, Alibaba Cloud Community Blog author.

InSpec is an open-source testing framework for infrastructure with a human- and machine-readable language for specifying compliance, security and policy requirements. As configuration practices have become more efficient, they are no longer the bottleneck of a system. Rather, testing, security, or compliance are the new bottlenecks. Among these, workflows are often the cause of hiccup. To address this problem, Chef launched InSpec as a framework that can be used by both security professionals and DevOps gurus alike to deliver software at high velocity without sacrificing software quality along the way.

Kitchen is a test harness tool to execute your configured code on one or more platforms in isolation. A driver plugin architecture is used that allows you to run your code on infrastructure provided by several different cloud providers as well as take advantage of several different data virtualization technologies. For example, you can use Alibaba Cloud ECS, Apache CloudStack, Digital Ocean, Rackspace, and OpenStack, among many others. Many testing frameworks are already supported out of the box including Bats, shUnit2, RSpec, Serverspec, among many other options out there.

In this tutorial, you'll learn how to write tests for your ansible playbooks running on an Alibaba Cloud Elastic Compute Service (ECS) instance with Ubuntu 18.04 installed. We'll use kitchen, an integration tool for developing and testing infrastructure code and software on isolated target platforms. Test-kitchen supports testing frameworks such as InSpec, Serverspec, and Bats. By the end of this tutorial, you'll be able to test your ansible-playbook deployment.
Prerequisites

To complete this tutorial, you'll need to have the following installed on your local machine.

    VirtualBox
    Ruby
    Vagrant

Downloading Chef Development Kit (ChefDK)

Before you start writing tests for ansible deployment, you'll first need to setup ChefDK on your local machine. The Chef development kit contains all the tools you'll need, such as inSpec and kitchen to develop and test your infrastructure. You can find the current/stable release based on your OS along with their checksums on the ChefDK download page.

If you're a MacOS and Linux user, you'll need to ensure that the version of Ruby that is included with the Chef development kit is configured as the default version of Ruby by running this command.

user@chef:~$ echo 'eval "$(chef shell-init bash)"' >> ~/.bash_profile && source ~/.bash_profile

You are using the chef shell-init subcommand to do this operation. If you're using a shell different from bash such as zsh, fish, and Windows PowerShell (posh), run the command below replacing SHELL_NAME with your shell and .SHELL_PROFILE with your shell profile.

user@chef:~$ echo 'eval "$(chef shell-init SHELL_NAME)"' >> ~/.SHELL_PROFILE && source ~/.SHELL_PROFILE

Initalizing Kitchen

Before initializing Kitchen, you'll need to create and move into a directory where you can keep your project files. You'll call the folder ansible_testing_dir. Also, you can use any name of your choice.

user@chef:~$ mkdir ~/ansible_testing_dir && cd ~/ansible_testing_dir

Within the project directory, you'll run the kitchen init command specifying ansible_playbook as the provisioner to initialize kitchen.

user@chef:~$ kitchen init --provisioner=ansible_playbook

This will create a new file and directories which should look like:

ansible_testing_dir
├── test
│   └── integration
│       └── default
├── chefignore
└── kitchen.yml

Note that in the above code:

    test/integration/default is where you'll save your test files to.
    chefignore is where you ignore chef related files, but you won't be using it in this tutorial.
    kitchen.yml: This file describes your testing configuration, which is what you want to test and the target platforms.

In order for your test to work with Ansible, you have to install the kitchen-ansible gem on your machine. You can run the command below to do so:

user@chef:~$ gem install kitchen-ansible

Creating the ansible-playbook

In this step, you'll want to create a playbook and roles that setup Nginx and Node.js. Your tests will be run against the playbook, and the tests will ensure conditions specified in the playbook are met.

You'll create a roles directory for both the Nginx and Node.js role

user@chef:~$ mkdir -p roles/{nginx,nodejs}/tasks

This will create a directory structure as shown:

roles
├── nginx
│   └── tasks
└── nodejs
    └── tasks

Now, you'll create a main.yml file in roles/nginx/tasks directory. In this file, you'll create a task that sets up and starts nginx.

---
- name: Update cache repositories and install Nginx
  apt:
    name: nginx
    update_cache: yes

- name: Change nginx directory permission
  file: 
    path: /etc/nginx/nginx.conf
    mode: 0750

- name: start nginx
  service:
    name: nginx
    state: started

You are also going to create a main.yml file in roles/nodejs/tasks to define a task that sets up nodejs.

---
- name: Update caches respository
  apt:
    update_cache: yes

- name: Add gpg key for NodeJS LTS
  apt_key:
    url: "https://deb.nodesource.com/gpgkey/nodesource.gpg.key"
    state: present

- name: Install the NodeJS LTS repo
  apt_repository:
    repo: "deb https://deb.nodesource.com/node_{{ NODEJS_VERSION }}.x {{ ansible_distribution_release }} main"
    state: present
    update_cache: yes

- name: Install Node.js
  apt:  
    name: nodejs
    state: present

- name: Install create react app
  command: npx create-react-app my-app

You'll create a file named playbook.yml in your root directory. In this file, you define your ansible configurations including its variables and the order in which you want your roles to run and much more.

---
 - hosts: all
   become: true
   remote_user: ubuntu
   vars:
     NODEJS_VERSION: 8

Writing Your Test

Before you start to go on and write your test, let's take a moment to look at the format of an Inspec test. As with many test frameworks, Inspec code resembles natural human language. Inspec has two main components, the subject to examine and the subject's expected state.

# you can use plain tests
describe '<entity>' do
  it { <expection> }
end

# you can also add controls here
control 'Can be anything unique' do  # A unique ID for this control
  impact 0.7                         # The criticality, if this control fails and must be a value between 0.0 and 1.0
  title 'A human-readable title'     # A human-readable title
  desc  'An optional description'
  describe '<entity' do              # The actual test
    it { <expectation> }
  end
end

The <entity> is the subject you want to examine, for example, a package name, service, file, or network port. The <expectation> part specifies the desired result or expected state. For example, Nginx should be installed or should have a specific version. You can check this documentation to learn more about Inspec format.

Using the format above, you'll use InSpec's package resource to test if Node.js is installed. You'll create a file named sample.rb in your test/integration/default directory and put the tests there.

describe package('nodejs') do
  it { should be_installed }
end

To run this test, you need to edit your kitchen.yml to specify the playbook you created earlier and add some new configurations. Replace the content of kitchen.yml with this:

---
driver:
  name: vagrant

provisioner:
  name: ansible_playbook
  hosts: test-kitchen
  playbook: ./playbook.yml

verifier:
  name: inspec

platforms:
  - name: ubuntu-16.04
    verifier:
      inspec_tests:
        - test/integration/default

suites:
  - name: default

The kitchen.yml file includes the following configs:

    driver: The driver option specifies the name of the driver that will be used to create platform instances or virtual machines used during testing.

    provisioner: The provisioner option takes care of configuring the virtual machine provided by the driver. This is most commonly a configuration management framework, and is used for both Chef or Shell. You are specifying the following options for provisioner
        name: The name of the configuration management framework to use.
        hosts: The group of hosts defined in your inventory file.
        playbook: The path to the ansible-playbook file.
        verifier: Specifies which application to use when running tests, such as inspec.

        platforms: The platforms options include the following:
            name: The name of the OS or distro to be used when creating the virtual machine.
            verifier: This specifies that the project contains Inspec tests and the tests directory.
        suites: Suites is a collection of test suites, with each suite_name grouping defining an aspect of a cookbook/roles to be tested.

Run the kitchen test command to run the test

user@chef:~$ kitchen test

The results are as follows:

----> Starting Kitchen (v2.2.5)
-----> Cleaning up any prior instances of <default-ubuntu-1604>
-----> Destroying <default-ubuntu-1604>...
       ==> default: Forcing shutdown of VM...
       ==> default: Destroying VM and associated drives...
       Vagrant instance <default-ubuntu-1604> destroyed.
       Finished destroying <default-ubuntu-1604> (0m5.41s).
-----> Testing <default-ubuntu-1604>
-----> Creating <default-ubuntu-1604>...
       Bringing machine 'default' up with 'virtualbox' provider...
       [SSH] Established
       Vagrant instance <default-ubuntu-1604> created.
       Finished creating <default-ubuntu-1604> (0m39.29s).
-----> Converging <default-ubuntu-1604>...
-----> Installing Chef Omnibus to install busser to run tests
       PLAY [all] *********************************************************************
       
       TASK [Gathering Facts] *********************************************************
       ok: [localhost]
       
       PLAY RECAP *********************************************************************
       localhost                  : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
       
       Downloading files from <<default-ubuntu-1604>>
       Finished converging <<default-ubuntu-1604>> (0m55.05s).
-----> Setting up <<default-ubuntu-1604>>...
$$$$$$ Running legacy setup for Vagrant Driver
       Finished setting up <default-ubuntu-1604>> (0m0.00s).
-----> Verifying <<default-ubuntu-1604>>...
       Loaded tests from {:path=>". ansible_testing_dir.test.integration.default"} 

Profile: tests from {:path=>"ansible_testing_dir/test/integration/default"} (tests from {:path=>"ansible_testing_dir.test.integration.default"})
Version: (not specified)
Target:  ssh://vagrant@127.0.0.1:2222

 <^> System Package nodejs<^>
     <^>×  should be installed <^>
    <^> expected that System Package nodejs is installed<^>

Test Summary: 0 successful, 1 failure, 0 skipped
>>>>>> ------Exception-------
>>>>>> Class: Kitchen::ActionFailed
>>>>>> Message: 1 actions failed.
>>>>>>     Verify failed on instance <<default-ubuntu-1604>>.  Please see .kitchen/logs/<default-ubuntu-1604>.log for more details
>>>>>> ----------------------
>>>>>> Please see .kitchen/logs/kitchen.log for more details
>>>>>> Also try running `kitchen diagnose --all` for configuration

  4.54s user 1.77s system 5% cpu 2:02.33 total

From the output, you could see that your test is failing because you don't have Node.js installed on the VM you provisioned with kitchen. You are going to fix the test by adding the nodejs role to your playbook.yml file and run the test again.

Edit the playbook.yml file to include the nodejs role.

---
 - hosts: all
   become: true
   remote_user: ubuntu
   vars:
    NODEJS_VERSION: 8

   roles:
    - nodejs

Now, you'll rerun the test using the kitchen test command.

user@chef:~$ kitchen test

The results are as follows:

......
Target:  ssh://vagrant@127.0.0.1:2222

  System Package nodejs
       should be installed

Test Summary: 1 successful, 0 failures, 0 skipped
       Finished verifying <default-ubuntu-18> (0m4.89s).
-----> Destroying <default-ubuntu-18>...
       DigitalOcean instance <145512952> destroyed.
       Finished destroying <default-ubuntu-18> (0m2.23s).
       Finished testing <default-ubuntu-18> (2m49.78s).
-----> Kitchen is finished. (2m55.14s)
  4.86s user 1.77s system 3% cpu 2:56.58 total

Your test now passes because you have Node.js installed using the nodejs role.

Here is a summary of what the kitchen test command is doing:

    It destroys the VM if it exists.
    It creates the VM.
    It converges the VM.
    It verifies the VM with InSpec.
    It destroys the VM.

You'll add one more test case to your sample.rb file

...
control 'nginx-conf' do
  impact 1.0
  title 'NGINX configuration'
  desc 'The NGINX config file should owned by root, be writable only by owner, and not writeable or and readable by others.'
  describe file('/etc/nginx/nginx.conf') do
    it { should be_owned_by 'root' }
    it { should be_grouped_into 'root' }
    it { should_not be_readable.by('others') }
    it { should_not be_writable.by('others') }
    it { should_not be_executable.by('others') }
  end
end
...

You'll run the kitchen test command again.

user@chef:~$ kitchen test

The results are as follows:

...
Target:  ssh://vagrant@127.0.0.1:2222

  ×  nginx-conf: NGINX configuration (2 failed)
     ×  File /etc/nginx/nginx.conf should be owned by "root"
     expected `File /etc/nginx/nginx.conf.owned_by?("root")` to return true, got false
     ×  File /etc/nginx/nginx.conf should be grouped into "root"
     expected `File /etc/nginx/nginx.conf.grouped_into?("root")` to return true, got false
       File /etc/nginx/nginx.conf should not be readable by others
       File /etc/nginx/nginx.conf should not be writable by others
       File /etc/nginx/nginx.conf should not be executable by others

  System Package nodejs
       should be installed
Profile Summary: 0 successful controls, 1 control failure
Test Summary: 4 successful, 2 failures

If some of your tests are failing, as just saw here, then you'll need to fix that by adding the nginx role to your playbook file and rerun the test again.

---
...
 roles:
  - nodejs
  - nginx

Run the test again.

user@chef:~$ kitchen test

The results are as follows:

...
Target:  ssh://vagrant@127.0.0.1:2222
    nginx-conf: NGINX configuration
       File /etc/nginx/nginx.conf should be owned by "root"
       File /etc/nginx/nginx.conf should be grouped into "root"
       File /etc/nginx/nginx.conf should not be readable by others
       File /etc/nginx/nginx.conf should not be writable by others
       File /etc/nginx/nginx.conf should not be executable by others

  System Package nodejs
       should be installed

Profile Summary: 1 successful controls, 0 control failures, 0 controls skipped
Test Summary: 6 successful, 0 failures, 0 skipped

Also, you'll need to run the kitchen destroy command to ensure you delete the VM after running your tests.

user@chef:~$ kitchen destroy

The results are as follows:

-----> Starting Kitchen (v1.24.0)
-----> Destroying <default-ubuntu-18>...
       Finished destroying <default-ubuntu-1604> (0m0.00s).
-----> Kitchen is finished. (0m5.07s)
  3.79s user 1.50s system 82% cpu 6.432 total

Conclusion

Through this tutorial, you have gained a general foundation into how you can test your ansible deployment. For a deeper drive into Inspec and Kitchen, you can check out Inspec's official documentation and Kitchen's documentation.

