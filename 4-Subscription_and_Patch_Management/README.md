# PART 3: Subscription and Patch Management
Now that we have an installed machine up and running we want to make sure that our machine is registered and updated- after all we don't want to have our users to have to manually update their machines after they get them (because they won't).

# Getting an Ansible Tower inventory that has your new machine
Prior to creating this machine it obviously wasn't in an inventory so we were running the Ansible against Satellite.  Now that we have a machine up and running, we want to take advantage of the scaling ability of Ansible and Tower, so we'll need to find a way to get our new machine in a variable.

## Creating an Inventory Backed by Satellite
Luckily there is a quick and easy integration to create an inventory based on machines that are managed/have been provisioned by Satellite.

1. Create the credential for use with Satellite
   * In Tower, navigate to **Credentials-->+**
   * Name the Credential **Satellite Credential**
   * Create an appropriate description in the **Description** field
   * Choose **Red Hat Satellite 6** as the Credential Type
   * Put the URL for Satellite in the **Satellite 6 URL** field
   * Put in a Satellite username with appropriate privileges in the **Username** field
   * Put in the password for the Satellite user in the **Password** field
2. Create the Satellite Inventory
   * In Tower, navigate to **Inventory-->+-->Inventory**
   * Name the Inventory **Satellite Inventory**
   * Click **Save**
   * Click **Sources** at the top of the Inventory
   * Click the **+** at the top of the page
   * Name the source **Satellite Source**
   * Give the source an appropriate description in the **Description** field
   * From the dropdown on the right for **Source** Choose **Red Hat Satellite 6**
   * The **Credential** field should auto-populate, but if it doesn't fill with the correct credential, click the **Credential** field and choose the credential that we created in **Step 1**
   * If you wish, you may choose to enable one of the **Update Options**, typically **Update on Launch** is a good choice, however with larger inventories this can cause unnessary overhead- plan accordingly   

## Accessing our newly built machine from the inventory (and registering/updating it)
Now that we have an inventory we need to modify our current process a bit so that we can make use of it.

1. First we need to modify our existing playbook a bit so that we can grep out the activation key then use it later.
```
# set stats that can be utilized by other jobs in workflow
- name: "Set stats vars"
set_fact:
    activation_key: "{{ 'RHEL-' + system_os.split(' ')[-1].split('.')[0] + '-' + lifecycle_env }}"
```

We will be using activation keys for registering our servers.  The name format my activation keys use are RHEL-[MAJOR VERSION]-[LIFECYCLE ENVIRONMENT].  Choose a format that makes sense to you (and update the stat as necessary).

2. Second we need to create the next playbook that will register our newly built machine to Satellite

```
---
- hosts: "{{ requested_server_name }}"
  tasks:
    - name: Download and install a copy of the CA Certificate for the Red Hat Satellite 6 server
      yum:
        disable_gpg_check: yes
        name: http://satellite.example.com/pub/katello-ca-consumer-latest.noarch.rpm
        state: present

    - name: Register with Satellite
      redhat_subscription:
        state: present
        activationkey: "{{ activation_key }}"
        org_id: "{{ satellite_org }}"

      #katello-agent is deprecated in RHEL8
    - name: Install katello-agent for RHEL 6 & 7
      yum:
        name: katello-agent
        state: present
      when:
        - ansible_distribution_major_version == "6" or ansible_distribution_major_version == "7"
```

A couple of important items here:
  - We define the host as the requested_server_name so that we are only registering the newly created machine.
  - We download the CA certificates and install them directly from the Satellite server
  - The activation_key and satellite_org vars are being inherited from the previous step in the workflow (which we will cover shortly).
  - The katello-agent is only installed for RHEL 6 & 7 machines because it is depricated on RHEL 8 and you shouldn't have any RHEL 5 (it's 13 years old for goodness sake)

3. Now, before we get into our workflow, we are going to create a third playbook that we can use to update all the packages on the server.

```
---
- name: Update "{{ requested_server_name }} packages"
  hosts: "{{ requested_server_name }}"
  tasks:
    - name: Update packages
      yum:
        name: '*'
        state: latest
```
It's a simple playbook, but very powerful.


## Putting it all together
Now that we've got our playbooks all written, we can tie them all together in Tower by using a workflow.





