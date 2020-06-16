# PART 1: Using Ansible To Provision From Satellite
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

### Compute Resources
Compute Resources are analagous to your virtualization manager, and can include Red Hat Virtualization, Red Hat OpenStack, VMWare, Hyper-V, Google, AWS, Azure, and Libvirt.

To configure these, throught the Satellite UI, navigate to **Infrastructure-->Compute Resources**, then select the appropriate provider and fill out the fields.  

**You will need to repeat this process for each unique manager/datacenter combo for hypervisors and for each region for cloud providers**

A good approach for this would be to adopt a sensical naming architecture (even if unnessary at first) such as DC1-VirtManager1.example.com or Azure-West1-Tenant1.

### Compute Profiles
Compute profiles are analagous to what most cloud providers refer to as instance types, consisting of standardized CPU/RAM/Disk combinations.

To configure these, through, the Satellite UI, navigate to **Infrastructure-->Compute Profiles** and create the appropriate profiles that your organization desires.  
*Note: remember the purpose of automation is to handle MOST of your cases, not necessarily all of them. Keep the number of compute profiles you have to match your most common use cases.*


## Enter The Foreman Modules
Foreman is one of the backend applications that make up the suite of services provided by Satellite.  By utilizing Ansible modules for Foreman we can take an automated approach towards accessing the functions provided by Foreman.

## Installing The Modules and Pre-requisites
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


## Using The Modules
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

## Turning on the virtual machine
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

## Go find a tasty beverage of choice
Once the machine starts up it will automatically start the PXE booting process, and the the installation of the requested software.  Nicely done!  Part two will cover how to take our work and make it more Ansible Tower appropriate.