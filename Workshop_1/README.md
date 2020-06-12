# Using Ansible To Provision From Satellite
Satellite has an excellent tool for provisioning machines, both from a physical and virtual standpoint.  However sometimes a need may exist to create many machines (or just one) in a more automated fashion.


## Satellite Prep
Configuring Repository Syncs, Lifecycle Environments, and Content Views are the core concepts of Satellite, so it is assumed that those are already set up in for this process.

There are however a few things that require additional configuration:
- Compute Resources
- Compute Profiles


## Finding Specific Names for Repos, the quick way
There are numerous ways to find things in Satellite, however one of the quickest is the `hammer` command.  Some useful commands to considering during this process are:
Useful hammer commands
`hammer repository list` # lists all of the repos and their names (necessary for kickstarts)

`hammer proxy list` # lists all of the satellite smart-proxies (including capsules for the remote sites)

`hammer content-view list` # lists all of the content views available

`hammer lifecycle-environment list` # lists all of the lifecycle environments

`hammer --help` # when you don't know exactly what you're looking for

###Compute Resources
Compute Resources are analagous to your virtualization manager, and can include Red Hat Virtualization, Red Hat OpenStack, VMWare, Hyper-V, Google, AWS, Azure, and Libvirt.

To configure these, throught the Satellite UI, navigate to **Infrastructure-->Compute Resources**, then select the appropriate provider and fill out the fields.  

**You will need to repeat this process for each unique manager/datacenter combo for hypervisors and for each region for cloud providers**

A good approach for this would be to adopt a sensical naming architecture (even if unnessary at first) such as DC1-VirtManager1.example.com or Azure-West1-Tenant1.

###Compute Profiles
Compute profiles are analagous to what most cloud providers refer to as instance types, consisting of standardized CPU/RAM/Disk combinations.

To configure these, through, the Satellite UI, navigate to **Infrastructure-->Compute Profiles** and create the appropriate profiles that your organization desires.  
*Note: remember the purpose of automation is to handle MOST of your cases, not necessarily all of them. Keep the number of compute profiles you have to match your most common use cases.*


## Enter The Foreman Modules
Foreman is one of the backend applications that make up the suite of services provided by Satellite.  By utilizing Ansible modules for Foreman we can take an automated approach towards accessing the functions provided by Foreman.

### Installing The Modules and Pre-requisites
Foreman is not one of the default modules that is distributed with Ansible, however it is easily installed from Ansible Galaxy with the command:
`ansible-galaxy collection install theforeman.foreman`

**(this should be performed on all Ansible Tower application nodes)**

Documentation for all of the Foreman modules can be found here:
<https://theforeman.org/plugins/foreman-ansible-modules/>

For the module to work properly there are a few python dependencies that must be filled, namely:
- PyYAML
- apypie
- ipaddress for the foreman_subnet module on Python 2.7
- rpm for the RPM support in the katello_upload module
- debian for the DEB support in the katello_upload module

These can be installed via pip:
`sudo pip3 install apypie`

**(this should be performed on all Ansible Tower application nodes)**


### Using The Modules
The main module of interest here is the **foreman_host** module.

This module will allow for the creation/deletion/modification of hosts.

A basic play for creating a server might look like this:

```
  - name: "Create a host"
    foreman_host:
      username: "satellite_admin"
      password: "satellite_admin_pw"
      server_url: "https://satellite.example.com"
      name: "new-server.example.com"
      compute_resource: "virtualization-manager.example.com"
      architecture: "x86_64"
      build: true
      domain: "example.com"
      compute_profile: "Large"
      organization: "Satellite Org"
      location: "Satellite Location"
      mac: e0:d5:5e:2b:08:90
      ip: 192.168.100.239
      root_pass: "new-servers-root-pw"
      subnet: "192.168.100.0/24"
      provision_method: "build"
      lifecycle_environment: "Production"
      content_view: "Satellite Content View"
      operatingsystem: "RHEL Server 7.8"
      ptable: "Kickstart default"
      content_source: "satellite.example.com"
      pxe_loader: "PXELinux BIOS"
      kickstart_repository: "Red Hat Enterprise Linux 7 Server Kickstart x86_64 7.8"
      validate_certs: no
      state: present

```
After running this playbook we will see that a couple of things have been accomplished:
- A host was created in Satellite, and assigned all of the appropriate information (Org, Location, Lifecycle Enviornment, Content View, etc)
- A host was created in our virtualization environment, but not turned on

### Turning on the virtual machine
Because you're not trying to supplement your automation with manual intervention...

Good news, we can accomplish this by utilizing the foreman\_host\_power module.

```
  - name: "Start up host"
    foreman_host_power:
      username: "satellite_admin"
      password: "satellite_admin_pw"
      server_url: "https://satellite.example.com/"
      name: "new-server.example.com"
      state: on
      validate_certs: no
```

### Go find a tasty beverage of choice
Once the machine starts up it will automatically start the PXE booting process, and the the installation of the requested software.

## Making it available to consumers
What is the best kind of automation?  The kind you don't have to be involved in.  So, let's take this to the next level and get out of the way and allow users to request this themselves!

### Living in this (ivory) Tower
Ansible Tower is the logical next step when you need to level up your automation game through sharing, scheduling, auditing, etc.

There are a few items that we will have to addres in our playbooks to make them more Tower-friendly (and useful).

### Variable Replacement
Our ultimate goal is that our customers/users will be able to self-serve and create their own virtual machines without having to write their own playbooks.  In order to enable that we are going to need to work on making our playbook more variable friendly.

```
  - name: "Create a host"
    foreman_host:
      username: "{{ satellite_admin }}"
      password: "{{ satellite_admin_pw }}"
      server_url: "https://satellite.example.com"
      name: "{{ requested_server_name }}
      compute_resource: "{{ compute_resource }}"
      architecture: "x86_64"
      build: true
      domain: "{{ example.com }}"
      compute_profile: "{{ compute_profile }}"
      organization: "{{ satellite_org }}"
      location: "{{ satellite_location }}"
      mac: "{{ 56 | random_mac }}"
      ip: "{{ server_ip }}"
      root_pass: "{{ new-servers-root-pw }}"
      subnet: "{{ subnet }}"
      provision_method: "build"
      lifecycle_environment: "{{ lifecycle_env }}"
      content_view: "{{ content_view }}
      operatingsystem: "{{ operating_system }}"
      ptable: "Kickstart default"
      content_source: "satellite.example.com"
      pxe_loader: "PXELinux BIOS"
      kickstart_repository: "{{ kickstart_repo }}"
      validate_certs: no
      state: present
```
There is a lot going on here (and we're just getting started), but the gist of it is that we have taken any of the information that can be set and given it a nice sensible variable name.  Ultimately these variable will be populated via a Survey in Ansible Tower.

### But there are things in that list I don't want users to enter/know!
Absolutely, let's address that now.

- Satellite Username/PW
  - This can be handled by creating a custom credential-type in Tower that will store the Satellite Username/PW in an encrypted store, that can later be accessed as variables when that credential is associated to the Job Template
- Compute Resource
  - Your user's may not know the name of your RHV-M or vCenter instance, this may be further complicated if you have multiple compute resources available.
  - The suggested approach here would be to offer the user a name they are familiar with in the Survey (like DC1), and in your playbook doing a dictionary lookup (against a dictionary you've already created that maps simple names to real values)
- Lifecycle Environments, Content Views, Content Sources, pxe_loader, kickstart repository
  - The goal of keeping it simple for the user does make it more complicated for us on the backend, however the suggestion here would again be to do a dictionary lookup that maps values (against a dictionary you've created)
- Subnet's and IP's
  - User's may not be familiar with subnet names/masks so the suggested approach would be to use common names and again use the dictionary mapping technique previously mentioned.

###Shut up about dictionary mapping and show me
Sheesh, fine.

These can be imported as vars in any of the ways that is supported, but for the sake of simplicity we will do it in the vars section of our playbook.

```
---
- name: "Power on Host"
  hosts: all
  collections:
  - theforeman.foreman
  vars:
    compute_resources:
      DC1: virtualization-manager-1.example.com
      DC2: virtualization-manager-2.example.com
    lifecycle_envs:
      Prod: Production
      UAT: User_Acceptance_Test
      Dev: Development
    content_views:
      RHEL7: "Red Hat Enterprise Linux 7"
      RHEL8: "Red Hat Enterprise Linux 8"
    kickstart_repos:
      RHEL_7_7: "Red Hat Enterprise Linux 7 Server Kickstart x86_64 7.7"
      RHEL_7_8: "Red Hat Enterprise Linux 7 Server Kickstart x86_64 7.8"
      RHEL_8_0: "Red Hat Enterprise Linux 8 for x86_64 - BaseOS Kickstart 8.0"
    subnets:
      App1_subnet: "192.168.100.0/24"
      App2_subnet: "10.1.0.0/16"
      DMZ: "172.16.0.0/22"

---truncated---


```

###Okay, so now we have a dictionary, how do we access these values?
We map Survey response answers to keys in the dictionary, whose values correspond to what we actually need.

So now we can call these by doing the following:

```
---truncated---

  vars:
    my_dc: DC1
    compute_resources:
    DC1: virtualization-manager-1.example.com
    DC2: virtualization-manager-2.example.com

---truncated---

  - name: "Lookup the Virtualization Manager for the DC"
    set_fact:
      compute_resource: "{{ item.value }}"
    loop: "{{ lookup('dict', compute_resources) }}"
    when: "'{{ my_dc }}' in item.key"

  - name: "Create a host"
    foreman_host:
      username: "{{ satellite_admin }}"
      password: "{{ satellite_admin_pw }}"
      server_url: "https://satellite.example.com"
      name: "{{ requested_server_name }}
      compute_resource: "{{ compute_resource }}"

---truncated---


```
In the example above, assume the variable 'my_dc' gets choosen by the user from a list, where the options are DC1 and DC2 because they just know what Datacenter they want to deploy in.  When they choose D1, we will set a fact that does a lookup of DC1-->virtualization-manager-1.example.com, which will then get used in our playbook to create the VM.

This is a single example, however this can be done for each of the situations where we need to get information from a customer/user but they might not know the exact resource name.  

###What about IP addresses?
Hopefully your organization uses some sort of IPAM that is programmable via an API (or better yet Ansible module).  In most cases this is typically something like InfoBlox.  In the instance of InfoBlox we would use the `nios_next_ip` module to retrieve the IP address and set it as a fact to later be used in the foreman_host module.

###What about DNS?
Satellite can be used to automatically create DNS records, or DNS records can be created using one of the various DNS modules available from providers.  This will not be covered here as DNS providers vary widely.


