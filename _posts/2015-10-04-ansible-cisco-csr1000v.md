---
layout: post
title: Deploying the Cisco CSR 1000v using Ansible
description: How to deploy Cisco's CSR 1000v using Ansible and VMWare Fusion
tags:
  - ansible
  - cisco
  - osx
  - vmware
  - fusion
  - csr1000v
published: false
---
## Introduction
In my <a href="http://pseudo.co.de/ansible-cisco-vwlc" target="_blank">last post</a>, I described how to automate the deployment of the Cisco Virtual Wireless Controller in an OS X VMWare Fusion environment using Ansible.

In this article, I will discuss how to do the same thing, but this time for the Cisco Cloud Services Router (CSR) 1000v.    

## Cisco CSR 1000V

The CSR 1000V is a software appliance based upon Cisco's ASR 1000 product family, providing enterprise and service provider grade routing in the cloud.  

Although you most often deploy the CSR 1000V in production to VMWare ESX, it is very useful to be able to run the CSR 1000V up on your laptop for testing and demonstration purposes.

## The Problem  

Part of creating <a href="http://cloudhotspot.co" target="_blank">CloudHotspot</a> is having a development environment that incorporates the CSR 1000V for testing and development purposes.

The CSR 1000V supports the <a href="http://www.cisco.com/c/en/us/td/docs/ios-xml/ios/isg/configuration/xe-3s/isg-xe-3s-book.html" target="_blank">Intelligent Subscriber Gateway (ISG)</a> and <a href="http://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iwag/configuration/xe-3s/iwag-xe-3s-book/iwag-overview.html" target="_blank">Intelligent Wireless Access Gateway (iWAG)</a> feature sets, both of which are important aspects of the future <a href="http://cloudhotspot.co" target="_blank">CloudHotspot</a> services.

I wanted to automate deployment of the CSR 1000V locally in my OS X/VMWare Fusion environment and settled on leveraging the OVF Runtime Environment feature that is supported by the CSR 1000V.

## Getting Started

I have created <a href="https://github.com/cloudhotspot/ansible-csr1000v-role" target="_blank">an Ansible role</a>, which is <a href="https://galaxy.ansible.com/list#/roles/5368" target="_blank">published on Ansible Galaxy as `mixja.csr1000v`</a>, and you can install as follows:

{% highlight bash %}
$ brew install ansible
...
$ ansible-galaxy install mixja.csr1000v
...
{% endhighlight %}

I have also published a <a href="https://github.com/cloudhotspot/ansible-csr1000v-playbook" target="_blank">sample playbook on Github</a>.  The <a href="https://github.com/cloudhotspot/ansible-csr1000v-playbook/blob/master/README.md" target="_blank">README file</a> provides full details on how to deploy the CSR 1000V using my role.

## The OVF Runtime Environment  

The CSR 1000V includes support for the <a href="http://www.virtuallyghetto.com/2012/06/ovf-runtime-environment.html" target="_blank">OVF Runtime Environment</a>, which allows you to pre-configure the CSR 1000V virtual machine at deployment time.

If you import the CSR 1000V OVA image into VMWare vCenter, you are able to specify a number of different parameters before the virtual machine is deployed:

vCenter supports the creation of the OVF Runtime Environment, which incorporates the configuration parameters shown above and makes them available to the guest virtual machine.

How the guest virtual machine gets access to these parameters is by one of two options:

- **ISO method** - an XML-based OVF Environment File is mounted as an IDE device using an ISO file created during the OVA import process.  The guest operating system processes the environment file accordingly.  This method is used by the CSR 1000V.
- **VMWare Tools method** - the guest operating system uses VMWare tools to introspect custom configuration parameters that are set in the virtual machine VMX file.  <a href="http://www.virtuallyghetto.com/2014/10/how-to-evaluate-the-vsphere-vcsa-beta-running-on-vmware-fusion-workstation.html" target="_blank">The vCenter Server Appliance (VCSA) uses this approach.</a>     

With either of the above methods, it is possible to manually create the OVF environment yourself - all vCenter does is automate the various tasks required.  

VMWare Fusion does not support creation of the OVF environment, so if you want to take advantage of the OVF environment feature, you have to manually create the environment yourself.  

This is where we can use Ansible to automate the various steps we need to take.

### Creating an OVF Environment File

The following is the <a href="https://github.com/cloudhotspot/ansible-csr1000v-role/blob/master/templates/ovf-env.xml.j2" target="_blank">OVF environment file template</a> that is used in the <a href="https://galaxy.ansible.com/list#/roles/5368" target="_blank">`mixja.csr1000v`</a> role:

{% highlight xml %}
{% raw %}
<?xml version="1.0" encoding="UTF-8"?>
<Environment
     xmlns="http://schemas.dmtf.org/ovf/environment/1"
     xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
     xmlns:oe="http://schemas.dmtf.org/ovf/environment/1"
     xmlns:ve="http://www.vmware.com/schema/ovfenv"
     oe:id=""
     ve:vCenterId="vm-31">
   <PlatformSection>
      <Kind>VMware ESXi</Kind>
      <Version>5.5.0</Version>
      <Vendor>VMware, Inc.</Vendor>
      <Locale>en</Locale>
   </PlatformSection>
   <PropertySection>
         <Property oe:key="com.cisco.csr1000v.CONTROL_PORT.1" oe:value=""/>
         <Property oe:key="com.cisco.csr1000v.MGMT_KEY.1" oe:value=""/>
         <Property oe:key="com.cisco.csr1000v.Mode.1" oe:value=""/>
         <Property oe:key="com.cisco.csr1000v.TUNNEL_PORT.1" oe:value=""/>
         <Property oe:key="com.cisco.csr1000v.config-version.1" oe:value="1.0"/>
         <Property oe:key="com.cisco.csr1000v.domain-name.1" oe:value="{{ csr_domain_name | default('') }}"/>
         <Property oe:key="com.cisco.csr1000v.enable-scp-server.1" oe:value="{{ csr_enable_scp | default ("False") }}"/>
         <Property oe:key="com.cisco.csr1000v.enable-ssh-server.1" oe:value="True"/>
         <Property oe:key="com.cisco.csr1000v.hostname.1" oe:value="{{ csr_name | default("csr1000v") }}"/>
         <Property oe:key="com.cisco.csr1000v.ic-tunnel-header-size.1" oe:value="148"/>
         <Property oe:key="com.cisco.csr1000v.ic-tunnel-ipv4-addr.1" oe:value=""/>
         <Property oe:key="com.cisco.csr1000v.ic-tunnel-ipv4-gateway.1" oe:value=""/>
         <Property oe:key="com.cisco.csr1000v.license.1" oe:value="{{ csr_license_level | default("appx") }}"/>
         <Property oe:key="com.cisco.csr1000v.login-password.1" oe:value="{{ csr_admin_password | default('') }}"/>
         <Property oe:key="com.cisco.csr1000v.login-username.1" oe:value="{{ csr_admin_username | default('') }}"/>
         <Property oe:key="com.cisco.csr1000v.mgmt-interface.1" oe:value="{{ csr_mgmt_interface | default('GigabitEthernet1') }}"/>
         <Property oe:key="com.cisco.csr1000v.mgmt-ipv4-addr.1" oe:value="dhcp"/>
         <Property oe:key="com.cisco.csr1000v.mgmt-ipv4-gateway.1" oe:value="dhcp"/>
         <Property oe:key="com.cisco.csr1000v.mgmt-ipv4-network.1" oe:value=""/>
         <Property oe:key="com.cisco.csr1000v.mgmt-vlan.1" oe:value=""/>
         <Property oe:key="com.cisco.csr1000v.pnsc-agent-local-port.1" oe:value=""/>
         <Property oe:key="com.cisco.csr1000v.pnsc-ipv4-addr.1" oe:value=""/>
         <Property oe:key="com.cisco.csr1000v.pnsc-shared-secret-key.1" oe:value=""/>
         <Property oe:key="com.cisco.csr1000v.privilege-password.1" oe:value=""/>
         <Property oe:key="com.cisco.csr1000v.remote-mgmt-ipv4-addr.1" oe:value=""/>
   </PropertySection>
</Environment>
{% endraw %}
{% endhighlight %}

The `<PropertySection>` element contains all of the various configuration parameters that can be applied to the CSR 1000V at deployment time as part of its provisioning process.

For example, the following snippet would configure the CSR 1000V virtual machine to have a login name of `admin` and password of `pass1234`:
{% highlight xml %}
<Property oe:key="com.cisco.csr1000v.login-password.1" oe:value="pass1234"/>
<Property oe:key="com.cisco.csr1000v.login-username.1" oe:value="admin"/>
{% endhighlight %}

### Applying Custom Configurations 

You can also specify in the OVF environment file any number of custom configuration commands to apply to the deployed CSR 1000V virtual machine:

{% highlight xml %}
<Property oe:key="com.cisco.csr1000v.ios-config-0005.1" oe:value="hostname csr01"/>
<Property oe:key="com.cisco.csr1000v.ios-config-0006.1" oe:value="boot-start-marker"/>
<Property oe:key="com.cisco.csr1000v.ios-config-0007.1" oe:value="boot-end-marker"/>
<Property oe:key="com.cisco.csr1000v.ios-config-0008.1" oe:value="vrf definition Mgmt-intf"/>
<Property oe:key="com.cisco.csr1000v.ios-config-0009.1" oe:value=" address-family ipv4"/>
<Property oe:key="com.cisco.csr1000v.ios-config-0010.1" oe:value=" exit-address-family"/>
<Property oe:key="com.cisco.csr1000v.ios-config-0011.1" oe:value=" address-family ipv6"/>
<Property oe:key="com.cisco.csr1000v.ios-config-0012.1" oe:value=" exit-address-family"/>
<Property oe:key="com.cisco.csr1000v.ios-config-0013.1" oe:value="aaa new-model"/>
{% endhighlight %}
 
The above will run the equivalent snippet from a standard Cisco configuration file:

{% highlight text %}
hostname csr01
!
boot-start-marker
boot-end-marker
!
!
!
vrf definition Mgmt-intf
 address-family ipv4
 exit-address-family
 !
 address-family ipv6
 exit-address-family
!
!
aaa new-model 
{% endhighlight %}

### Creating an ISO for the OVF Environment

The ISO method of creating the OVF Environment Runtime is simple:

- Create OVF Environment File in a file called `ovf-env.xml`
- Create an ISO that contains the `ovf-env.xml` file
- Attach the ISO as the second CD-ROM device and ensure it is connected  

You can do complete all of the tasks above manually, however the `mixja.csr1000v` role automates these tasks for you as outlined below.

Create OVF environment file based upon OVF environment template file:
{% highlight yaml %}
{% raw %}
- name: Create OVF environment folder
  file:
    path: "{{ csr_vm_safe_dst_full_path }}/ovf_env"
    state: directory
- name: Deploy OVF environment file
  template:
    src: "ovf-env.xml.j2" 
    dest: "{{ csr_vm_safe_dst_full_path }}/ovf_env/ovf-env.xml"
{% endraw %}
{% endhighlight %}

Add custom configuration commands by reading in the custom config file (if specified) and using the useful `with_indexed_items` loop to add the commands to the OVF environment file:
{% highlight yaml %}
{% raw %}
- name: Read bootstrap configuration file
  shell: cat {{ csr_config_file }}
  register: csr_config_items
  when: csr_config_file is defined
- name: Insert bootstrap configuration
  lineinfile:
    dest: "{{ csr_vm_safe_dst_full_path }}/ovf_env/ovf-env.xml"
    line: <Property oe:key="com.cisco.csr1000v.ios-config-{{ '%04d' | format(item.0 + 1) }}.1" oe:value="{{ item.1 }}"/>
    insertbefore: "</PropertySection>$"
  with_indexed_items: csr_config_items.stdout_lines
  when: csr_config_items.changed
{% endraw %}
{% endhighlight %}

Create the OVF Environment ISO using the `hdiutil` OS X utility:
{% highlight yaml %}
{% raw %}
- name: Create OVF environment ISO
  command: 'hdiutil makehybrid -iso -joliet -o "{{ csr_vm_safe_dst_full_path }}/{{ csr_vm_name }}-ovf_env.iso" "{{ csr_vm_safe_dst_full_path }}/ovf_env/"'
{% endraw %}
{% endhighlight %}

And finally connect the OVF Environment ISO to the virtual machine by configure the VMX file:

{% highlight yaml %}
{% raw %}
- name: Attach OVF environment ISO to virtual machine
  lineinfile:
    dest: "{{ csr_vm_vmx_path }}"
    line: ide1:1.fileName = "{{ csr_vm_name }}-ovf_env.iso"
    insertbefore: ide1:1.clientDevice = "FALSE"
- name: Change OVF environment CD-ROM to ISO type
  lineinfile:
    dest: "{{ csr_vm_vmx_path }}"
    line: ide1:1.deviceType = "cdrom-image"
    regexp: '^ide1:1.deviceType ='
- name: Connect OVF environment ISO to virtual machine
  lineinfile:
    dest: "{{ csr_vm_vmx_path }}"
    line: ide1:1.startConnected = "TRUE"
{% endraw %}
{% endhighlight %}