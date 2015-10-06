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
  - ovf
image:
  feature: hoops-bg.jpg
  credit: Keith Cuddeback
  creditlink: https://www.flickr.com/photos/in2photos/6217731177/
published: true
---

In my <a href="http://pseudo.co.de/ansible-cisco-vwlc" target="_blank">last post</a>, I described how to automate the deployment of the Cisco Virtual Wireless Controller in an OS X VMWare Fusion environment using Ansible.  

In this article, I will discuss how you can automate deployment of the Cisco Cloud Services Router (CSR) 1000v using VMWare Fusion and Ansible.

As you will see, the deployment for the CSR 1000V is quite different from the Cisco Virtual Wireless Controller.

## Cisco CSR 1000V

The CSR 1000V is a software appliance based upon Cisco's ASR 1000 product family, providing enterprise and service provider grade routing in the cloud.  

Although you most often deploy the CSR 1000V in production to VMWare ESX, it is very useful to be able to run the CSR 1000V up on your laptop for testing and demonstration purposes.

The CSR 1000V is <a href="https://software.cisco.com/download/release.html?mdfid=284364978&softwareid=282046477&release=3.14.1S&relind=AVAILABLE&rellifecycle=ED&reltype=latest" target="_blank">distributed in a number of formats</a> and this article will focus on deployment using the OVA format.

## The Problem  

Part of creating <a href="http://cloudhotspot.co" target="_blank">CloudHotspot</a> is having a development environment that incorporates the CSR 1000V for testing and development purposes.

The CSR 1000V supports the <a href="http://www.cisco.com/c/en/us/td/docs/ios-xml/ios/isg/configuration/xe-3s/isg-xe-3s-book.html" target="_blank">Intelligent Subscriber Gateway (ISG)</a> and <a href="http://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iwag/configuration/xe-3s/iwag-xe-3s-book/iwag-overview.html" target="_blank">Intelligent Wireless Access Gateway (iWAG)</a> feature sets, both of which enable important subscriber management features that the future <a href="http://cloudhotspot.co" target="_blank">CloudHotspot</a> product will support.

I wanted to automate deployment of the CSR 1000V locally in my OS X/VMWare Fusion environment and found that I could leverage the OVF Runtime Environment feature that is supported by the CSR 1000V (which I will discuss shortly).

## Enter Ansible

To support the various deployment aspects I discuss in this article, I have created <a href="https://github.com/cloudhotspot/ansible-csr1000v-role" target="_blank">an Ansible role</a>, which is <a href="https://galaxy.ansible.com/list#/roles/5368" target="_blank">published on Ansible Galaxy as `mixja.csr1000v`</a>, and you can install as follows:

{% highlight bash %}
$ brew install ansible
...
$ ansible-galaxy install mixja.csr1000v
...
{% endhighlight %}

Note I have also published a <a href="https://github.com/cloudhotspot/ansible-csr1000v-playbook" target="_blank">sample playbook on Github</a>.  The <a href="https://github.com/cloudhotspot/ansible-csr1000v-playbook/blob/master/README.md" target="_blank">README file</a> provides full details on how to deploy the CSR 1000V using my role.

In addition to deploying the CSR 1000V, the role does some useful things like setting up a DHCP reservation and creating an entry in your `/etc/hosts` file.  I'm not going to cover these details in this article, rather just discuss an interesting feature of VMWare called the OVF Runtime Environment, how the CSR 1000V uses this feature, and how my Ansible role automates configuration of this environment on VMWare Fusion.

## The OVF Runtime Environment  

The CSR 1000V includes support for the <a href="http://www.virtuallyghetto.com/2012/06/ovf-runtime-environment.html" target="_blank">OVF Runtime Environment</a>, which allows you to pre-configure the CSR 1000V virtual machine at deployment time.

If you import the CSR 1000V OVA image into VMWare vCenter (**File -> Deploy OVF Template**), you are able to specify a number of different parameters before the virtual machine is deployed:

<figure>
  <img src="/images/vcenter-ovf.png" alt="vCenter OVF Deployment">
</figure>

vCenter automatically creates the OVF runtime environment, which incorporates the configuration parameters shown above and makes them available to the guest virtual machine.

How the guest virtual machine gets access to these parameters is via the **OVF environment transport**.  VMWare supports two OVF environment transport options:

- **ISO method** - an XML-based OVF environment file is mounted as an IDE device using an ISO file created during the OVA import process.  The guest operating system processes the environment file accordingly.  This method is used by the CSR 1000V. 
- **VMWare Tools method** - the guest operating system uses VMWare tools to introspect the OVF environment file via the guestInfo.ovfEnv variable.    

You can examine the OVF environment settings by selecting **Inventory -> Virtual Machine -> Settings** and then clicking the **Options** tab:

<figure>
  <img src="/images/vcenter-ovf-environment.png" alt="vCenter OVF Environment Settings">
</figure>

Clicking the **View...** button in the **OVF Environment** section shows the OVF environment document.  This is handy as it shows you exactly how the OVF environment file is constructed.

As you can see, vCenter fully supports deployment via the OVF environment.  Unfortunately VMWare Fusion does not support OVF environment creation, so if you want to take advantage of the OVF environment feature, you have to manually create the environment yourself.  

This is where we can use Ansible to automate the various manual steps we normally would need to take.

## Creating the OVF Environment on VMWare Fusion

To create the OVF environment on VMWare Fusion (using the ISO transport supported by the CSR 1000V), the following steps are required:

- <a href="#create-ovf-environment">Create an OVF environment file</a>
- <a href="#apply-commands">Apply custom configuration commands to the OVF environment file (optional)</a>
- <a href="#create-iso">Create an ISO image</a>
- <a href="#connect-iso">Connect the ISO image to the virtual machine</a>

### <a name="create-ovf-environment"></a>Creating an OVF Environment File

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

### <a name="apply-commands"></a>Applying Custom Configurations 

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

### <a name="create-iso"></a>Creating an ISO for the OVF Environment

The ISO method of creating the OVF Environment Runtime is simple:

- Create OVF Environment File in a file called `ovf-env.xml`
- Create an ISO that contains the `ovf-env.xml` file

You can complete all of the tasks above manually, however the `mixja.csr1000v` role automates these tasks for you as outlined below.

Here is how the OVF environment file is created based upon the OVF environment template file listed above:

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

Here is how custom configuration commands are added by reading in the custom config file (if specified) and using the `with_indexed_items` loop to add the commands to the OVF environment file:

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

Here is how to create the OVF Environment ISO using the `hdiutil` OS X utility:

{% highlight yaml %}
{% raw %}
- name: Create OVF environment ISO
  command: 'hdiutil makehybrid -iso -joliet -o "{{ csr_vm_safe_dst_full_path }}/{{ csr_vm_name }}-ovf_env.iso" "{{ csr_vm_safe_dst_full_path }}/ovf_env/"'
{% endraw %}
{% endhighlight %}

### <a name="connect-iso"></a>Connecting the OVF Environment ISO 

The final step of connecting the OVF Environment ISO to the virtual machine is achieved by configuring the virtual machine VMX file:

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

With the ISO connected, the virtual machine can be started.  The CSR 1000V will detect the presence of the OVF environment ISO and will configure itself according to the OVF environment file settings.

## CSR 1000V Deployment Walkthrough

The CSR 1000V deployment process consists of a number of phases discussed in this section.

Upon initial boot the CSR 1000V detects an operating system does not current exist.  

The CSR 1000V installs the operating system from the CSR 1000V install ISO (this is a different ISO from the OVF environment ISO):

{% highlight text %}
%IOSXEBOOT-4-FILESYS_ERRORS_CORRECTED: (rp/0): bootflash 1 contained errors which were auto-corrected.
%IOSXEBOOT-4-FILESYS_ERRORS_CORRECTED: (rp/0): bootflash 5 contained errors which were auto-corrected.
%IOSXEBOOT-4-FILESYS_INVALID: (rp/0): partition /dev/bootflash6 has an invalid filesystem that must be re-initialized.
%IOSXEBOOT-4-FILESYS_REPAIR: (rp/0): The system will repair /dev/bootflash6 now.
%IOSXEBOOT-4-FILESYS_REPAIR: (rp/0): This will cause a complete loss of data.
%IOSXEBOOT-4-FILESYS_INVALID: (rp/0): partition /dev/bootflash7 has an invalid filesystem that must be re-initialized.
%IOSXEBOOT-4-FILESYS_REPAIR: (rp/0): The system will repair /dev/bootflash7 now.
%IOSXEBOOT-4-FILESYS_REPAIR: (rp/0): This will cause a complete loss of data.
%IOSXEBOOT-4-FILESYS_INVALID: (rp/0): partition /dev/bootflash8 has an invalid filesystem that must be re-initialized.
%IOSXEBOOT-4-FILESYS_REPAIR: (rp/0): The system will repair /dev/bootflash8 now.
%IOSXEBOOT-4-FILESYS_REPAIR: (rp/0): This will cause a complete loss of data.
%IOSXEBOOT-4-FILESYS_INVALID: (rp/0): partition /dev/bootflash9 has an invalid filesystem that must be re-initialized.
%IOSXEBOOT-4-FILESYS_REPAIR: (rp/0): The system will repair /dev/bootflash9 now.
%IOSXEBOOT-4-FILESYS_REPAIR: (rp/0): This will cause a complete loss of data.
%IOSXEBOOT-4-WATCHDOG_DISABLED: (rp/0): Hardware watchdog timer disabled: watchdog device not found
%IOSXEBOOT-4-BOOT_SRC: (rp/0): Link super image to /tmp/sw/isos
%IOSXEBOOT-4-BOOT_SRC: (rp/0): TarFilePath:/tmp/sw/isos/csr_mgmt_rel.tgz
%IOSXEBOOT-4-BOOT_SRC: (rp/0): Get iosxe-remote-mgmt.03.14.01.S.155-1.S1-std.ova from /bootflash/tar_file_tmp/iosxe-remote-mgmt.03.14.01.S.155-1.S1-std.ova
%IOSXEBOOT-4-BOOT_SRC: (rp/0): !!upgrade the container ova!!
%IOSXEBOOT-4-BOOT_SRC: (rp/0): CD-ROM Boot
%IOSXEBOOT-4-BOOT_CDROM: (rp/0): Using Serial console
%IOSXEBOOT-4-BOOT_CDROM: (rp/0): Installing GRUB
%IOSXEBOOT-4-BOOT_CDROM: (rp/0): Copying super package csr1000v-universalk9.03.14.01.S.155-1.S1-std.SPA.bin
%IOSXEBOOT-4-BOOT_CDROM: (rp/0): Expanding super package on /bootflash
%IOSXEBOOT-4-BOOT_CDROM: (rp/0): Creating /boot/grub/menu.lst
%IOSXEBOOT-4-BOOT_CDROM: (rp/0): Ejecting CD-ROM tray
%IOSXEBOOT-4-BOOT_CDROM: (rp/0): CD-ROM Installation finished
%IOSXEBOOT-4-BOOT_CDROM: (rp/0): Rebooting from HD
{% endhighlight %}

After installation, the CSR 1000V reboots and loads the newly flashed operating system.  The operating system detects that the nvram startup configuration file is missing:

{% highlight text %}
Cisco IOS Software, CSR1000V Software (X86_64_LINUX_IOSD-UNIVERSALK9-M), Version 15.5(1)S1, RELEASE SOFTWARE (fc1)
Technical Support: http://www.cisco.com/techsupport
Copyright (c) 1986-2015 by Cisco Systems, Inc.
Compiled Sun 01-Mar-15 03:58 by mcpre



Cisco IOS-XE software, Copyright (c) 2005-2015 by cisco Systems, Inc.
All rights reserved.  Certain components of Cisco IOS-XE software are
licensed under the GNU General Public License ("GPL") Version 2.0.  The
software code licensed under GPL Version 2.0 is free software that comes
with ABSOLUTELY NO WARRANTY.  You can redistribute and/or modify such
GPL code under the terms of GPL Version 2.0.  For more details, see the
documentation or "License Notice" file accompanying the IOS-XE software,
or the applicable URL provided on the flyer accompanying the IOS-XE
software.


% failed to initialize nvram
{% endhighlight %}

The operating system detects the OVF runtime environment via the attached OVF ISO.  Instead of defaulting to the standard user system configuration wizard, the operating system applies the OVF environment configuration:

{% highlight text %}
...
cisco CSR1000V (VXE) processor (revision VXE) with 2151294K/6147K bytes of memory.
Processor board ID 9G5BXAMY4CO
3 Gigabit Ethernet interfaces
32768K bytes of non-volatile configuration memory.
3988444K bytes of physical memory.
7774207K bytes of virtual hard disk at bootflash:.
.Done!

Press RETURN to get started!

*Oct  3 10:06:37.887: %VUDI-6-EVENT: [serial number: 9G5BXAMY4CO], [vUDI: CSR1000V:9G5BXAMY4CO], A new vUDI has been generated
*Oct  3 10:06:37.887: %VUDI-6-EVENT: [serial number: 9G5BXAMY4CO], [vUDI: CSR1000V:9G5BXAMY4CO], A new vUDI is generated for this new VM instance
*Oct  3 10:06:38.205: %SMART_LIC-6-AGENT_READY: Smart Agent for Licensing is initialized
*Oct  3 10:06:38.310: %IOS_LICENSE_IMAGE_APPLICATION-6-LICENSE_LEVEL: Module name = csr1000v Next reboot level = ax and License = No valid license found
*Oct  3 10:06:46.943: %IOSXE_RP_NV-3-NV_ACCESS_FAIL: Initial read of NVRAM contents failed
*Oct  3 10:06:47.097: %VXE_THROUGHPUT-6-LEVEL: Throughput level has been set to 100 kbps
*Oct  3 10:06:47.097: %VXE_THROUGHPUT-2-LOW_THROUGHPUT: CSR throughput set to low default level 100kbps, system performance can be severely impacted.Please install a valid license, configure the boot level and reload, to switch to a higher throughput
*Oct  3 10:06:47.102: memupgrade reg_invoke_iosxe_license_register_feature: VXE_MEM_LIC_STR

*Oct  3 10:06:51.101: %VUDI-6-EVENT: [serial number: 9G5BXAMY4CO], [vUDI: ], vUDI initialization is now complete
*Oct  3 10:06:52.363: %SPANTREE-5-EXTENDED_SYSID: Extended SysId enabled for type vlan
*Oct  3 10:06:53.386: %LINK-3-UPDOWN: Interface Lsmpi0, changed state to up
*Oct  3 10:06:53.386: %LINK-3-UPDOWN: Interface EOBC0, changed state to up
*Oct  3 10:06:53.386: %LINEPROTO-5-UPDOWN: Line protocol on Interface LI-Null0, changed state to up
*Oct  3 10:06:53.386: %LINK-3-UPDOWN: Interface LIIN0, changed state to up
*Oct  3 10:06:53.641: %IOSXE-3-PLATFORM:kernel: Failed to activate dev Gi2: error 1
*Oct  3 10:06:53.696: %LINK-3-UPDOWN: Interface GigabitEthernet1, changed state to down
*Oct  3 10:06:53.711: %LINK-3-UPDOWN: Interface GigabitEthernet2, changed state to down
*Oct  3 10:06:53.712: %LINK-3-UPDOWN: Interface GigabitEthernet3, changed state to down
*Oct  3 10:06:54.386: %LINEPROTO-5-UPDOWN: Line protocol on Interface Lsmpi0, changed state to up
*Oct  3 10:06:54.388: %LINEPROTO-5-UPDOWN: Line protocol on Interface EOBC0, changed state to up
*Oct  3 10:06:54.388: %LINEPROTO-5-UPDOWN: Line protocol on Interface LIIN0, changed state to up
*Oct  3 10:06:46.284: %CPPHA-7-START:cpp_ha:  CPP 0 preparing ucode
*Oct  3 10:06:46.610: %CPPHA-7-START:cpp_ha:  CPP 0 startup init
*Oct  3 10:06:46.831: %CPPHA-7-START:cpp_ha:  CPP 0 running init
*Oct  3 10:06:47.359: %CPPHA-7-READY:cpp_ha:  CPP 0 loading and initialization complete
*Oct  3 10:06:47.446: %IOSXE-6-PLATFORM:cpp_cp: Process CPP_PFILTER_EA_EVENT__API_CALL__REGISTER
*Oct  3 10:06:52.628: %IOSXE-3-PLATFORM:kernel: Failed to activate dev Gi1: error 1
*Oct  3 10:06:54.653: %IOSXE-3-PLATFORM:kernel: Failed to activate dev Gi3: error 1
*Oct  3 10:06:54.697: %LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet1, changed state to down
*Oct  3 10:06:54.712: %LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet2, changed state to down
*Oct  3 10:06:54.712: %LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet3, changed state to down
*Oct  3 10:07:12.560: %CVAC-5-XML_PARSED: 18 line(s) of configuration generated from file cdrom1:/ovf-env.xml
...
...
...
Building configuration...

*Oct  3 10:07:34.814: %PKI-6-AUTOSAVE: Running configuration saved to NVRAM[OK]startup-config file open failed (Device or resource busy)

*Oct  3 10:07:37.065: %CVAC-4-CONFIG_DONE: Configuration generated from file cdrom1:/ovf-env.xml was applied and saved to NVRAM. See bootflash:/cvac.log for more details.
*Oct  3 10:07:38.142: %CONFIG_CSRLXC-5-CONFIG_DONE: Configuration was applied and saved to NVRAM. See bootflash:/csrlxc-cfg.log for more details.
{% endhighlight %}
     

## Wrap Up

In this article, I described how the CSR 1000V can be configured at deployment time using the VMWare OVF runtime environment feature.  

Although this feature is fully supported on VMWare vCenter, it is not supported on other VMWare platforms such as VMWare Fusion.

You've seen that you can easily manually create the OVF runtime environment, but doing this manually is cumbersome.  I have created <a href="https://github.com/cloudhotspot/ansible-csr1000v-role" target="_blank">an Ansible role  `mixja.csr1000v` </a> that automates creating the OVF runtime environment, which can save significant time and effort if you regularly need to create the CSR 1000V in VMWare Fusion environments for development or demonstration purposes.

     