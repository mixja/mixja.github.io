---
layout: post
title: Deploying the Cisco vWLC using Ansible
description: How to deploy Cisco's Virtual Wireless Controller using Ansible and VMWare Fusion
tags:
  - ansible
  - cisco
  - osx
  - vmware
  - fusion
  - wireless
  - wlc
  - vwlc
  - controller
image:
  feature: bridge-bg.jpg
  credit: Keith Cuddeback
  creditlink: https://www.flickr.com/photos/in2photos/8234899589/
published: true
---

In this article, I will describe how to deploy Cisco's Virtual Wireless Controller (vWLC) on VMWare Fusion using Ansible.

Although Ansible is most often used for deploying infrastructure and applications or orchestrating continuous delivery workflows, it can be used to automate almost any task.
  
I will create an Ansible playbook that will automate all of the manual tasks you would normally take to get the vWLC up and running.

**UPDATE 6th October, 2015: I have refactored the original playbook that was the subject of this article into a reusable <a href="https://galaxy.ansible.com/list#/roles/5387" target="_blank">Ansible Galaxy role</a>.  This article has been updated accordingly and has numerous changes from the original article.**
     

## Cisco Virtual Wireless Controller

The vWLC is a software appliance in Cisco's Wireless Controller product line that can manage up to 200 access points and 6000 clients.  

Although you most often deploy the vWLC in production to VMWare ESX, it is very useful to be able to run a vWLC up on your laptop for testing and demonstration purposes.  

## The Problem

So what's the problem we are trying to solve here?  

Well, part of creating <a href="http://cloudhotspot.co" target="_blank">CloudHotspot</a> is having a development environment that incorporates Cisco's vWLC for testing and development purposes.  

I wanted to automate deployment of the vWLC, initially looking to:

- **Vagrant** - a no go, the rather strange SSH arrangement on the vWLC pretty much breaks the model Vagrant is expecting to work with.
- **Packer** - I was able to create a Packer build that configured a vWLC from scratch, but sending a bunch of key presses over VNC with carefully timed pauses seemed somewhat fragile.

The only other method I could come up with was to leverage the vWLC <a href="http://www.cisco.com/c/en/us/td/docs/wireless/controller/7-2/configuration/guide/cg/cg_gettingstarted.html#wp1249285" target="_blank">**AutoInstall feature**</a>, which uses the classic TFTP network boot approach employed by Cisco IOS and many other network devices.  

AutoInstall requires a few ancillary pieces of infrastructure to get it working:

- **DHCP server** - used to advertise the IP address of a TFTP server from which network devices can download configuration files (see <a href="http://www.cisco.com/c/en/us/td/docs/wireless/controller/7-2/configuration/guide/cg/cg_gettingstarted.html#wp1144108" target="_blank">how the Cisco vWLC uses DHCP for Autoinstall</a>)
- **TFTP server** - hosts configuration files for bootstrapping network devices (see <a href="http://www.cisco.com/c/en/us/td/docs/wireless/controller/7-2/configuration/guide/cg/cg_gettingstarted.html#wp1144143" target="_blank">how the Cisco vWLC selects a configuration file</a>)

VMWare Fusion ships with a DHCP server for the host networking functions, and OS X ships with a TFTP daemon (disabled by default), so all of the necessary ingredients to make AutoInstall work are already available in a VMWare Fusion environment.

All that is needed is a little orchestration and automation magic from Ansible.  

## Quick Start

This purpose of this article is to explain in detail how the Ansible role called `mixja.vwlc` that <a href="https://galaxy.ansible.com/list#/roles/5387" target="_blank">I've published on Ansible Galaxy</a> actually works.

However it is worthwhile to give a quick overview of how to use the playbook before we delve into details.  A <a href="https://github.com/cloudhotspot/ansible-vwlc-playbook" target="_blank">sample playbook</a> is published on Github which you should use to get started.

The target user experience here is to:

- Download the OVA appliance from CCO
- Review the <a href="https://github.com/cloudhotspot/ansible-vwlc-playbook#quick-start">Quick Start</a> section in the sample playbook README.  Here you will note that you have to provide a valid OVA source image and a destination root for the deployed virtual machine.
- Define configuration parameters as described in the <a href="https://github.com/cloudhotspot/ansible-vwlc-playbook#configuration-variables">sample playbook README</a>.  
- Run the playbook as demonstrated below.

To install the Ansible Galaxy role:

`ansible-galaxy install mixja.vwlc`

To run the playbook:

`ansible-playbook site.yml`

And to run the playbook and overwrite a previous installation:

`ansible-playbook site.yml --extra-vars wlc_vm_overwrite=true`     

## Workflow

The rest of this article will now discuss the `mixja.vwlc` role in detail.  The source for this role is <a href="https://github.com/cloudhotspot/ansible-vwlc-role" target="_blank">published on Github</a>, and is published on <a href="https://galaxy.ansible.com/list#/roles/5387" target="_blank">Ansible Galaxy</a>. 

The high-level workflow of the playbook is as follows:

- <a href="#deploying-vm">Deploying the virtual machine</a>
- <a href="#introspecting">Introspecting the deployed virtual machine</a>
- <a href="#configuring-tftp">Configure the inbuilt OS X TFTP daemon to serve an appropriate configuration file for the controller</a>
- <a href="#configuring-dhcp">Configure the VMWare DHCP server</a>
- <a href="#autoinstall-cleanup">AutoInstall the vWLC and clean up once provisioning is complete</a>  

## <a name="deploying-vm"></a>Deploying the Virtual Machine

Deploying the OVA image sounds simple enough but to make the playbook fairly idiot-proof and user friendly, we need to consider a couple of what-if scenarios:

- What if we've already deployed the image to the desired location?
- What if a virtual machine is running at the desired location? 

We need to handle these scenarios appropriately, before the virtual machine can be extracted from the OVA image.

### Establishing some Facts 
The first set of tasks in the role are described in the <a href="https://github.com/cloudhotspot/ansible-vwlc-role/blob/master/tasks/set_facts.yml" target="_blank">`set_facts.yml`</a> file, which are used to create a few internal variables used throughout the role:

{% highlight yaml %}
{% raw %}
---
- name: Pad VM destination root path with /
  set_fact: wlc_vm_padded_destination='{{ wlc_vm_root }}/'
- name: Extract root directory name from padded VM root path
  set_fact: wlc_vm_safe_dst="{{ wlc_vm_padded_destination | dirname }}"
- name: Get full path to VM folder
  set_fact: wlc_vm_safe_dst_full_path="{{ wlc_vm_safe_dst }}/{{ wlc_vm_name }}.vmwarevm"
- name: Get full path fo VMX file
  set_fact: wlc_vm_vmx_path="{{ wlc_vm_safe_dst_full_path }}/{{ wlc_vm_name }}.vmx"
{% endraw %}
{% endhighlight %}

The above tasks require the following inputs:

- `wlc_vm_root` - Defines the root folder where the virtual machine will be deployed. This must be specified by the user as input to the role.
- `wlc_vm_name` - Defines the name of the virtual machine.  The default value is `wlc01`  

### Making some Checks 
Next, the tasks in <a href="https://github.com/cloudhotspot/ansible-vwlc-role/blob/master/tasks/checks.yml" target="_blank">`checks.yml`</a> verify that the `ovftool` utility is installed, and determines if a virtual machine is already deployed at the target destination path:

> <a href="https://www.vmware.com/support/developer/ovf/" target="_blank">VMWare provides a free tool called `ovftool` to registered VMWare users</a>, and this tool must be installed to deploy virtual machines from OVA files using the command line.

{% highlight yaml %}
{% raw %}
---
- name: Check for ovftool
  shell: pkgutil --pkgs | awk '/com.vmware.ovftool.application/'
  register: wlc_pkgutil_ovftool
  changed_when: False
- name: Fail if VMWare OVF Tools are not installed
  fail: msg="VMWare OVF Tools are required.  Please install and retry."
  when: 'not {{ wlc_pkgutil_ovftool.stdout | match("com.vmware.ovftool.application") }}'
- name: Get ovftool path
  shell: pkgutil --files com.vmware.ovftool.application | grep -FE 'ovftool$'
  register: wlc_ovftool_path
  changed_when: false
- name: Check VM path
  stat: path='{{ wlc_vm_safe_dst_full_path }}'
  register: wlc_vm_exists
  changed_when: False
- name: Fail if VM path exists
  fail: msg="VM already exists.  Please set wlc_vm_overwrite variable to any value to overwrite the existing VM"
  when: (wlc_vm_exists.stat.isdir is defined) and (wlc_vm_overwrite is not defined)
{% endraw %}
{% endhighlight %}

The `name: Check VM path` task checks if the desired VM location already exists.  Note the following convention to create this location:

{% raw %}
`{{ wlc_vm_root }}/{{ wlc_vm_name }}.vmwarevm`  
{% endraw %}

So the VM location will be `/path/to/vm/root/wlc01.vmwarevm` assuming a VM name of `wlc01`. 

Note I'm using a calculated `vm_safe_dst_full_path` variable, which is derived from the above convention.  This variable is manipulated to ensure we get the correct full path without any duplicate forward slashes. 

If the VM location already exists, the entire playbook is configured to fail in the `name: Fail if VM path exists` task, unless the `wlc_vm_overwrite` variable is defined with any value.  

This approach protects you from accidentally overwriting an existing virtual machine, but still allows you to explicitly overwrite it if that is your intention as demonstrated below:

`$ ansible-playbook site.yml --extra-vars wlc_vm_overwrite=true`

### Creating the VM Location

With initial facts set and checks out of the way, the `create_vm.yml` tasks create the virtual machine folder. 

{% highlight yaml %}
{% raw %}
---
- name: Get VMX path if existing VM is running
  shell: "'{{ wlc_vmrun_path }}' list | grep -F '{{ wlc_vm_safe_dst_full_path }}' || true"
  register: wlc_vmx_path
  when: wlc_vm_exists.stat.isdir is defined
  changed_when: wlc_vmx_path is defined and wlc_vmx_path.stdout != ""
  notify:
    - stop vm hard
    - pause three seconds
- meta: flush_handlers
- name: Remove existing VM path
  file: path='{{ wlc_vm_safe_dst_full_path }}' state=absent
- name: Create VM path
  file: path='{{ wlc_vm_safe_dst_full_path }}' state=directory
- name: Extract OVA using ovftool
  command: "'/{{ wlc_ovftool_path.stdout }}' '{{ wlc_ova_source }}' '{{ wlc_vm_vmx_path }}'"
{% endraw %}
{% endhighlight %}

Before creating the VM location I check if there is an existing VM running (assuming the VM location already exists and `wlc_vm_overwrite` has been defined).  

As the intention in this scenario is to overwrite an existing VM, we need to first stop the VM (if it is running) in order to remove the existing VM folder and files.

To do this, I use the `vmrun list` command which is included as part of the VMWare Fusion application. This is defined in the `name: Get VMX path if existing VM is running` task, using good old `grep` to extract the full path of the running VM vmx file at the target VM location.

Here's an example of the full output of the `vmrun list` command:

{% highlight console %}
$ /Applications/VMware\ Fusion.app/Contents/Library/vmrun list
Total running VMs: 1
/Users/jmenga/Virtual Machines.localized/wlc01.vmwarevm/wlc01.vmx
{% endhighlight %}

Note I use `changed_when` with a boolean expression to determine if the VM is actually running.  This is useful as the `stop vm hard` and `pause three seconds` handlers will only be called if `changed_when` evaluates to true:

{% highlight yaml %}
{% raw %}
- name: stop vm hard
  command: '"{{ wlc_vmrun_path }}" stop "{{ wlc_vm_vmx_path }}" hard'
  become: no
  
- name: pause three seconds
  pause: seconds=3
{% endraw %}
{% endhighlight %}

A few points to note here:

- I explicitly force the `stop vm hard` handler to run in the context of the user executing the playbook.  If you call this handler from a task that is running as root, the handler will run as root unless you specify `become: no`.  This is important for the `vmrun` command, as it only lists Virtual Machines running in the context of each user.    
- The `pause three seconds` handler prevents a race condition where the existing virtual machine shutdown may not complete gracefully before the next task that attempts to remove the existing virtual machine.
- The `meta: flush_handlers` task in `create_vm.yml` forces handlers to execute immediately.  By default, handlers run at the end of a play, which may not be the desired behaviour.     

At this point, the existing VM location (if it previously existed) can be safely removed using the `name: Remove existing VM path` task and the target VM location created using the `name: Create VM path` task.

The final step is to deploy the virtual machine from the OVA image, which is completed in the `name: Extract OVA using ovftool` task.  This task references the `wlc_ova_source` variable, which must be provided explicitly as input to the role.  The user must supply their own vWLC OVA image, which can be downloaded from <a href="https://software.cisco.com/download/release.html?mdfid=284464214&softwareid=280926587&release=8.0.120.0&relind=AVAILABLE&rellifecycle=ED&reltype=latest" target="_blank">Cisco</a> (CCO login required). 

> Pre-creating the target VM location alters the behaviour of the `ovftool`, which is used to deploy the virtual machine from the OVA image.  If you run this command and the destination VMX parent folder does not exist, ovftool behaves difficultly and creates another folder in the format {% raw %}`<vm name>.vmwarevm`{% endraw %} under the specified parent folder and then places the vmx file in this folder.  To avoid this behaviour, you must precreate the target VM parent folder.

## <a name="introspecting"></a>Introspecting the Virtual Machine

The next tasks that are executed are defined in the <a href="https://github.com/cloudhotspot/ansible-vwlc-role/blob/master/tasks/checks.yml" target="_blank">`introspect.yml`</a> file:

{% highlight yaml %}
{% raw %}
---
- name: Configure service port as Share with my Mac
  lineinfile: >
    dest='{{ wlc_vm_vmx_path }}'
    regexp='^ethernet0.connectionType ='
    line='ethernet0.connectionType = "nat"'
  notify:
    - start vm
    - stop vm
- meta: flush_handlers
- name: Get service port MAC address
  shell: cat '{{ wlc_vm_vmx_path }}' | awk -F'"' '/ethernet0.generatedAddress = /{print $2}'
  register: wlc_vm_mac_address
  changed_when: False
- name: Get vmnet8 IP address
  shell: ifconfig vmnet8 | awk '/inet/{print $2}'
  register: wlc_vmnet8_ip_address
  changed_when: False
- name: Set host IP address fact
  set_fact: wlc_host_ip_address={{ wlc_vmnet8_ip_address.stdout }}
- name: Set VM MAC access fact
  set_fact: wlc_vm_mac_address={{ wlc_vm_mac_address.stdout }}
- name: Set VM IP address fact
  set_fact: wlc_vm_ip_address={{ wlc_vmnet8_ip_address.stdout | regex_replace(wlc_vm_mgmt_ip_regex_match, wlc_vm_mgmt_ip_regex_replace) }}
- name: Set VM IP gateway fact
  set_fact: wlc_vm_ip_gateway={{ wlc_vmnet8_ip_address.stdout | regex_replace(wlc_vm_mgmt_ip_regex_match, wlc_vm_mgmt_gateway_regex_replace) }}
{% endraw %}
{% endhighlight %}

The `name: Configure Ethernet0 as Share with my Mac` task reconfigures the `ethernet0` network interface connection type to **Share with my Mac** using the very useful `lineinfile` Ansible module.  This results in the following entry in the vmx file:

`ethernet0.connectionType = "nat"`

This setting is important, as it ensures the service port on the vWLC appliance will use internal VMWare Fusion <a href="http://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=1022264" target="_blank">NAT networking mode</a> and the VMWare Fusion DHCP server.  The other `ethernet1` interface will remain in the default bridged networking mode.

> The Cisco vWLC appliance comes with two network interfaces.  `ethernet0` is the service port and `ethernet1` is the management port that connects access points.

At the end of this first task, the virtual machine is started and then immediately stopped.  The reason for this is that we need to generate a MAC address for the virtual machine service port interface, which does not happen until the virtual machine is started for the first time.  After bouncing the virtual machine, the `ethernet0.generatedAddress` key in the virtual machine vmx file will be populated with a MAC address:

`ethernet0.generatedAddress = "00:0c:29:0d:ec:56"`

The `name: Get service port MAC address` task parses the virtual machine VMX file to retrieve the MAC address, which is required to configure a DHCP reservation for the vWLC virtual machine service port.   

> Notice that `awk` is our friend here :)  You may notice that I use `awk` and `grep` interchangeably and in general the usual differences apply.  One key difference is that `grep` always returns an error if there is no match, where as `awk` does not.  This can litter your playbook output with unsightly errors, even if you choose to ignore errors (which IMHO is an Ansible antipattern).  One way to work around the `grep` error return code issue is to add on `|| true` at the end of the `grep` command.

The `name: Get vmnet8 IP address` task determines the IP address being used for the `Share with my Mac` vmnet8 network adapter.  This interface is connected to the service port of the vWLC virtual machine, and because 
the OS X TFTP daemon binds to all network interfaces, we can specify this IP address as the TFTP server address.  We also can derive the network portion of the vmnet8 network adapter, which we will need to configure our DHCP reservation later on.

The final tasks set a number of facts that are required for later tasks:

- `wlc_host_ip_address` - the host IP address of the `vmnet8` adapter.  This IP address is used as the TFTP server address by vWLC.
- `wlc_vm_mac_address` - the MAC address of the vWLC service port. 
- `vlc_vm_ip_address` - the IP address of the vWLC service port.  This is calculated as the network portion of the `vmnet8` IP address combined with the value of the `wlc_vm_svc_ip_octet` variable.  As the `vmnet8` IP address always uses a /24 subnet mask, this results in the first three octets of the `vmnet8` IP address plus the `wlc_vm_svc_ip_octet` value.  E.g. given a `vmnet8` IP address of 192.168.100.1 and `wlc_vm_svc_ip_octet` value of 121, the `vlc_vm_ip_address` will be 192.168.100.121.
- `wlc_vm_ip_gateway` - the router and DNS server address for the `vmnet8` network.  On VMWare Fusion, this is always the .2 address on the `vmnet8` network (e.g. 192.168.100.2 continuing on from the previous example).
 
These facts are used to configure TFTP and DHCP settings as you will see shortly.
 
## <a name="configuring-tftp"></a>Configuring the OS X TFTP Server
OS X ships with a TFTP server that is disabled by default.  The `tftp.yml` play defines the various tasks required to configure and enable the TFTP server:

{% highlight yaml %}
{% raw %}
---
- name: Check if TFTP daemon is running
  shell: launchctl list | awk /com.apple.tftp/
  become: yes
  register: wlc_tftp_daemon_status
  changed_when: wlc_tftp_daemon_status.stdout != ""
  notify:
    - stop system tftp daemon
- name: Deploy TFTP plist
  template: 
    src: "tftp.plist.j2" 
    dest: "{{ wlc_tftp_plist }}"
    mode: 0644
  become: yes
- name: Ensure TFTP path exists
  file: 
    path: "{{ wlc_tftp_path }}"
    state: directory
    mode: 0777
- name: Deploy WLC file
  template:
    src: "{{ wlc_config_file | default('ciscowlc.cfg.j2') }}"
    dest: "{{ wlc_tftp_path }}/ciscowlc.cfg"
  changed_when: true
  notify:
    - start user tftp daemon
{% endraw %}
{% endhighlight %}

The first task determines if a TFTP daemon is currently running.  If this is the case, the `stop system tftp daemon` handler in <a href="https://github.com/cloudhotspot/ansible-vwlc-role/blob/master/handlers/main.yml" target="_blank">`handlers/main.yml`</a> stops the TFTP daemon:

{% highlight yaml %}
{% raw %}
- name: stop system tftp daemon
  command: launchctl unload '{{ wlc_system_tftp_plist }}'
  become: yes
{% endraw %}
{% endhighlight %}
 
> OS X El Capitan includes a new feature called <a href="https://developer.apple.com/library/mac/releasenotes/MacOSX/WhatsNewInOSX/Articles/MacOSX10_11.html#//apple_ref/doc/uid/TP40016227-DontLinkElementID_17" target="_blank">system integrity protection</a>, which prevents even root/sudo access from modifying OS X system files.  This includes the standard `/System/Library/LaunchDaemons/tftp.plist` file (which is the value of the `wlc_system_tftp_plist` variable in the handler above) that is used to configure the OS X TFTP server.  Although you can disable system integrity protection, the role avoids having to do this by creating a plist file outside of the protected OS X system file system (hence the reference to user and system TFTP daemons in the tasks).
 
In the `name: Deploy TFTP plist` task, a Jinja 2 template is used to configure the relevant settings in the file:

{% highlight xml %}
{% raw %}
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Disabled</key>
  <false/>
  <key>Label</key>
  <string>com.apple.tftpd</string>
  <key>ProgramArguments</key>
  <array>
    <string>/usr/libexec/tftpd</string>
    <string>-i</string>
    <string>{{ wlc_tftp_path }}</string>
  </array>
  <key>inetdCompatibility</key>
  <dict>
    <key>Wait</key>
    <true/>
  </dict>
  <key>InitGroups</key>
  <true/>
  <key>Sockets</key>
  <dict>
    <key>Listeners</key>
    <dict>
      <key>SockServiceName</key>
      <string>tftp</string>
      <key>SockType</key>
      <string>dgram</string>
    </dict>
  </dict>
</dict>
</plist>
{% endraw %}
{% endhighlight %}

The template enables the TFTP server by setting the `<key>Disabled</key>` value to `<false/>` and also includes the `wlc_tftp_path` variable (`/Users/Shared/tftp` by default) to specify the folder that the TFTP server should serve.  The `name: Ensure TFTP path exists` task ensures this folder is present.

The `name: Deploy WLC file` task then deploys the Cisco vWLC configuration file that will served via TFTP.  The playbook allows you to provide your own config file by setting the `wlc_config_file` variable - if this variable is not defined, the playbook deploys a basic configuration derived from the `templates/ciscowlc.cfg.j2` template:

{% highlight text %}
{% raw %}
# WLC Config Begin

config mdns service origin all AirTunes 
config mdns service create AirTunes _raop._tcp.local. origin all lss disable 
config mdns service origin all Airplay 
config mdns service create Airplay _airplay._tcp.local. origin all lss disable 
config mdns service origin all HP_Photosmart_Printer_1 
config mdns service query enable HP_Photosmart_Printer_1 
config mdns service create HP_Photosmart_Printer_1 _universal._sub._ipp._tcp.local. origin all lss disable query enable 
config mdns service origin all HP_Photosmart_Printer_2 
config mdns service query enable HP_Photosmart_Printer_2 
config mdns service create HP_Photosmart_Printer_2 _cups._sub._ipp._tcp.local. origin all lss disable query enable 
config mdns service origin all HomeSharing 
config mdns service query enable HomeSharing 
config mdns service create HomeSharing _home-sharing._tcp.local. origin all lss disable query enable 
config mdns service origin all Printer-IPP 
config mdns service create Printer-IPP _ipp._tcp.local. origin all lss disable 
config mdns service origin all Printer-IPPS 
config mdns service create Printer-IPPS _ipps._tcp.local. origin all lss disable 
config mdns service origin all Printer-LPD 
config mdns service create Printer-LPD _printer._tcp.local. origin all lss disable 
config mdns service origin all Printer-SOCKET 
config mdns service create Printer-SOCKET _pdl-datastream._tcp.local. origin all lss disable 
config mdns profile service add default-mdns-profile AirTunes 
config mdns profile service add default-mdns-profile Airplay 
config mdns profile service add default-mdns-profile HP_Photosmart_Printer_1 
config mdns profile service add default-mdns-profile HP_Photosmart_Printer_2 
config mdns profile service add default-mdns-profile HomeSharing 
config mdns profile service add default-mdns-profile Printer-IPP 
config mdns profile service add default-mdns-profile Printer-IPPS 
config mdns profile service add default-mdns-profile Printer-LPD 
config mdns profile service add default-mdns-profile Printer-SOCKET 
config mdns profile create default-mdns-profile 
config ap packet-dump truncate 0 
config ap packet-dump buffer-size 2048 
config ap packet-dump capture-time 10 
config ap preferred-mode ipv4 all 
config 802.11a cac voice sip bandwidth 64 sample-interval 20 
config 802.11a cac voice sip codec g711 sample-interval 20 
config switchconfig strong-pwd lockout attempts mgmtuser 3 
config switchconfig strong-pwd lockout time mgmtuser 5 
config interface dhcp management primary {{ wlc_dhcp_server_ip_address }}
config interface dhcp service-port enable 
config interface port management 1 
config interface address management {{ wlc_mgmt_ip_address }} {{ wlc_mgmt_ip_mask }} {{ wlc_mgmt_ip_gateway }}
config interface address virtual {{ wlc_virtual_ip_address }}
config database size 2048 
config mgmtuser add {{ wlc_admin_username }} {{ wlc_admin_password }} read-write 
config mobility group domain {{ wlc_mobility_group_name }} 
config certificate generate webadmin 
config advanced 802.11a channel add 36 
config advanced 802.11a channel add 40 
config advanced 802.11a channel add 44 
config advanced 802.11a channel add 48 
config advanced 802.11a channel add 52 
config advanced 802.11a channel add 56 
config advanced 802.11a channel add 60 
config advanced 802.11a channel add 64 
config advanced 802.11a channel add 149 
config advanced 802.11a channel add 153 
config advanced 802.11a channel add 157 
config advanced 802.11a channel add 161 
config advanced 802.11b channel add 1 
config advanced 802.11b channel add 6 
config advanced 802.11b channel add 11 
config sys-nas {{ wlc_name }} 
config network rf-network-name {{ wlc_rf_network_name }}
config network multicast l2mcast disable service-port 
config network multicast l2mcast disable virtual 
config time ntp interval {{ wlc_ntp_interval }} 
config time ntp server 1 {{ wlc_ntp_server }}
config sysname {{ wlc_name }} 
config country NZ 
config wlan exclusionlist 1 60 
config wlan security wpa enable 1 
config wlan security web-auth server-precedence 1 local radius ldap 
config wlan create 1 "{{ wlc_ssid }}" "{{ wlc_ssid }}" 
config wlan interface 1 management 
config wlan broadcast-ssid enable 1 
config wlan session-timeout 1 1800 
config wlan mfp client enable 1 
config wlan wmm allow 1 
config wlan enable 1 
config 802.11b 11gsupport enable 
config 802.11b cac voice sip bandwidth 64 sample-interval 20 
config 802.11b cac voice sip codec g711 sample-interval 20 

# WLC Config End
{% endraw %}
{% endhighlight %}

The template inserts various user configurable variables that are described in <a href="https://github.com/cloudhotspot/ansible-vwlc-role/blob/master/defaults/main.yml" target="_blank">`/defaults/main.yml`</a>:

{% highlight yaml %}
{% raw %}
# WLC configuration settings
wlc_name: wlc01
wlc_admin_username: admin
wlc_admin_password: Pass1234

wlc_mgmt_ip_address: 192.168.1.6
wlc_mgmt_ip_mask: 255.255.255.0
wlc_mgmt_ip_gateway: 192.168.1.254
wlc_dhcp_server_ip_address: 192.168.1.254 
wlc_virtual_ip_address: 1.1.1.1

wlc_mobility_group_name: "{{ wlc_name }}"
wlc_rf_network_name: "{{ wlc_name }}"
wlc_ntp_server: 64.99.80.30
wlc_ntp_interval: 3600
wlc_ssid: Test SSID
{% endraw %}
{% endhighlight %}

The task will deploy the configuration file to a file named `ciscowlc.cfg` - this name is used as it is one of the file names that the vWLC will attempt to download from the TFTP server as part of the AutoInstall feature (<a href="http://www.cisco.com/c/en/us/td/docs/wireless/controller/7-2/configuration/guide/cg/cg_gettingstarted.html#wp1144143" target="_blank">see here for more details</a>). 
 
## <a name="configuring-dhcp"></a>Configuring the VMWare DHCP Server
The required network environment to support the AutoInstall feature is almost in place.  All that remains is to configure the VMWare DHCP server as follows:

- Create a DHCP reservation for the vWLC service port interface.
- The DHCP reservation must include the BOOTP next server setting, which is used by the vWLC during AutoInstall to determine the IP address of the TFTP server to download its configuration from 

Configuring a DHCP reservation is useful for the following reasons:

- We know the IP address that will be assigned to the service port.  We need this to configure the `/etc/hosts` file on the host system and to determine when the vWLC has provisioned successfully.
- We can constrain custom DHCP settings (i.e. BOOTP next server) to the vWLC virtual machine only.  This avoids unforeseen side effects that might be caused by adding these settings globally.  

This requires two tasks that are defined in the <a href="https://github.com/cloudhotspot/ansible-vwlc-role/blob/master/defaults/main.yml" target="_blank">`dhcp.yml`</a> file:
 
{% highlight yaml %}
{% raw %}
---
- name: Remove previous DHCP reservations
  blockinfile:
    dest: '{{ wlc_dhcpd_conf_path }}'
    marker: "# {mark} ANSIBLE MANAGED BLOCK - {{ wlc_vm_name }} {{ wlc_vm_ip_address }}"
    content: ""
  become: yes
- name: Add DHCP reservation
  blockinfile:
    dest: '{{ wlc_dhcpd_conf_path }}'
    marker: "# {mark} ANSIBLE MANAGED BLOCK - {{ wlc_vm_name }} {{ wlc_vm_ip_address }}"
    insertafter: EOF
    content: |
      host {{ wlc_vm_name }} {
        hardware ethernet {{ wlc_vm_mac_address }};
        fixed-address {{ wlc_vm_ip_address }};
        option domain-name-servers {{ wlc_vm_ip_gateway }};
        option domain-name localdomain;
        default-lease-time 1200;
        max-lease-time 1200;  
        option routers {{ wlc_vm_ip_gateway }};
        next-server {{ wlc_host_ip_address }};
      }
  become: yes
  notify:
    - stop vmware networking
    - start vmware networking
    - start vm
- meta: flush_handlers
{% endraw %}
{% endhighlight %}

First any previous DHCP reservations are removed.  The VMWare DHCP configuration is controlled by the `/Library/Preferences/VMware Fusion/vmnet8/dhcpd.conf` file and an unmodified example is shown below:

{% highlight text %}
{% raw %}
# Configuration file for ISC 2.0 vmnet-dhcpd operating on vmnet8.
#
# This file was automatically generated by the VMware configuration program.
# See Instructions below if you want to modify it.
#
# We set domain-name-servers to make some DHCP clients happy
# (dhclient as configured in SuSE, TurboLinux, etc.).
# We also supply a domain name to make pump (Red Hat 6.x) happy.
#

###### VMNET DHCP Configuration. Start of "DO NOT MODIFY SECTION" #####
# Modification Instructions: This section of the configuration file contains
# information generated by the configuration program. Do not modify this
# section.
# You are free to modify everything else. Also, this section must start 
# on a new line 
# This file will get backed up with a different name in the same directory 
# if this section is edited and you try to configure DHCP again.

# Written at: 09/28/2015 19:46:42
allow unknown-clients;
default-lease-time 1800;                # default is 30 minutes
max-lease-time 7200;                    # default is 2 hours

subnet 192.168.232.0 netmask 255.255.255.0 {
	range 192.168.232.128 192.168.232.254;
	option broadcast-address 192.168.232.255;
	option domain-name-servers 192.168.232.2;
	option domain-name localdomain;
	default-lease-time 1800;                # default is 30 minutes
	max-lease-time 7200;                    # default is 2 hours
	option netbios-name-servers 192.168.232.2;
	option routers 192.168.232.2;
}
host vmnet8 {
	hardware ethernet 00:50:56:C0:00:08;
	fixed-address 192.168.232.1;
	option domain-name-servers 0.0.0.0;
	option domain-name "";
	option routers 0.0.0.0;
}
####### VMNET DHCP Configuration. End of "DO NOT MODIFY SECTION" #######
{% endraw %}
{% endhighlight %}

If you are familiar with the ISC DHCPD server, you'll notice this is exactly what VMWare is using for the DHCP service.  This makes it very easy to configure the DHCP server for our needs.

We add a DHCP reservation for the vWLC virtual machine to the bottom of the DHCP configuration file as defined in the `name: Add  DHCP reservation` task.  

This allows us to control the IP address allocated to the vWLC virtual machine and set the BOOTP next server (TFTP Server) that is required for AutoInstall.  This option set using the `next-server` directive in the reservation and specifying the IP address of the host vmnet8 adapter (`wlc_host_ip_address`).

VMWare Fusion uses a default DHCP range of x.x.x.128 - x.x.x.254, so you can reserve any IP address between 3 - 126 (x.x.x.1 and x.x.x.2 are used by VMWare).  Recall that this value is controlled by the `wlc_vm_svc_ip_octet` setting.

The DHCP reservation also needs to adopt the various other settings defined in the standard DHCP scope for the vmnet8 interface.

> To deploy the necessary configuration for the reservation, I'm using a <a href="https://github.com/yaegashi/ansible-role-blockinfile" target="_blank">third-party community module called yaegashi.blockinfile</a>.  

Here is an example of the DHCP configuration file with the DHCP reservation configuration appended to the end of the file:

{% highlight text %}
{% raw %}
# Configuration file for ISC 2.0 vmnet-dhcpd operating on vmnet8.
#
# This file was automatically generated by the VMware configuration program.
# See Instructions below if you want to modify it.
#
# We set domain-name-servers to make some DHCP clients happy
# (dhclient as configured in SuSE, TurboLinux, etc.).
# We also supply a domain name to make pump (Red Hat 6.x) happy.
#

###### VMNET DHCP Configuration. Start of "DO NOT MODIFY SECTION" #####
# Modification Instructions: This section of the configuration file contains
# information generated by the configuration program. Do not modify this
# section.
# You are free to modify everything else. Also, this section must start 
# on a new line 
# This file will get backed up with a different name in the same directory 
# if this section is edited and you try to configure DHCP again.

# Written at: 09/28/2015 19:46:42
allow unknown-clients;
default-lease-time 1800;                # default is 30 minutes
max-lease-time 7200;                    # default is 2 hours

subnet 192.168.232.0 netmask 255.255.255.0 {
	range 192.168.232.128 192.168.232.254;
	option broadcast-address 192.168.232.255;
	option domain-name-servers 192.168.232.2;
	option domain-name localdomain;
	default-lease-time 1800;                # default is 30 minutes
	max-lease-time 7200;                    # default is 2 hours
	option netbios-name-servers 192.168.232.2;
	option routers 192.168.232.2;
}
host vmnet8 {
	hardware ethernet 00:50:56:C0:00:08;
	fixed-address 192.168.232.1;
	option domain-name-servers 0.0.0.0;
	option domain-name "";
	option routers 0.0.0.0;
}
####### VMNET DHCP Configuration. End of "DO NOT MODIFY SECTION" #######

# BEGIN ANSIBLE MANAGED BLOCK wlc01 192.168.232.127
host wlc01 {
  hardware ethernet 00:0c:29:0d:ec:56;
  fixed-address 192.168.232.127;
  option domain-name-servers 192.168.232.2;
  option domain-name localdomain;
  default-lease-time 1200;
  max-lease-time 1200;  
  option routers 192.168.232.2;
  next-server 192.168.232.1;
}
# END ANSIBLE MANAGED BLOCK wlc01 192.168.232.127
{% endraw %}
{% endhighlight %}

The `blockinfile` module includes begin and end marker lines, which are useful for removing the inserted block later during cleanup.

With the modifications made to the DHCP configuration, the `name: Add DHCP reservation` task notifies several handlers, defined in `handlers/main.yml`:

{% highlight yaml %}
{% raw %}
- name: stop vmware networking
  command: '"{{ wlc_vmnet_path }}" --stop'
  become: yes
- name: start vmware networking
  command: '"{{ wlc_vmnet_path }}" --start'
  become: yes
- name: start vm
  command: '"{{ wlc_vmrun_path }}" start "{{ wlc_vm_vmx_path }}" nogui'
  become: no
{% endraw %}
{% endhighlight %}

These handlers restart VMWare networking using the `vmnet-cli` command (included with VMWare Fusion), allowing the DHCP configuration changes to take effect.  

With the DHCP configuration in place, the virtual machine is then started to begin the AutoInstall process.

> The order of handlers as defined in the handler file in Ansible is important.  I have noticed the order of execution follows the order specified in the handler file, rather than the order specified in the `notify` action of the calling task (as one might expect).
 
> UPDATE: At this point I have also added provisioning of the local `/etc/hosts` file with the vWLC name (as defined by `wlc_vm_name`) and service port IP address (as defined by `wlc_vm_ip_address`).  This occurs by default but can be disabled by setting `wlc_vm_persist_dhcp_reservation` to `no`.

## <a name="autoinstall-cleanup"></a>Virtual Machine AutoInstall and Cleanup
At this point, everything is in place for the (recently booted) vWLC virtual machine to use the AutoInstall feature:

- TFTP daemon is enabled and configured with a vWLC configuration file
- VMWare DHCP server is configured to advertise the TFTP server IP address and issue a DHCP reservation to vWLC virtual machine

### vWLC AutoInstall Process
The following screen shots show the various stages of vWLC AutoInstall.

First time boot of the vWLC virtual machine.  The OVA includes an ISO installer that images the virtual machine hard disk:
<figure>
  <img src="/images/vwlc-install-start.png" alt="vWLC Installation Start">
</figure>

After approximately one minute the hard disk imaging is complete and the appliance reboots:
<figure>
  <img src="/images/vwlc-image-complete.png" alt="vWLC Imaging Complete">
</figure>

One quirk of the vWLC virtual machine is that you have to explicitly enable console output at boot.  This is only needed if you need to access the console for any reason (or take screenshots of the install process :):
<figure>
  <img src="/images/vwlc-press-any-key.png" alt="vWLC press any key">
</figure>

At approximately T=2:30 the setup wizard will be displayed.  This allows you to manually configure the vWLC, which we obviously don't do in this case:
<figure>
  <img src="/images/vwlc-setup-wizard.png" alt="vWLC Setup Wizard">
</figure>

After 30 seconds the AutoInstall process will start automatically.  About 30 seconds into the AutoInstall process, you will see the vWLC virtual machine download the `ciscowlc.cfg` configuration file and reboot:
<figure>
  <img src="/images/vwlc-config-file.png" alt="vWLC Config File Download">
</figure>

After rebooting, the installation will be complete.  The entire process takes just a shade over five minutes.
<figure>
  <img src="/images/vwlc-install-complete.png" alt="vWLC Setup Wizard">
</figure>

### Cleanup
After installation is complete, the following tasks in the `vwlc.yml` file will be executed:

{% highlight text %}
{% raw %}
---
- name: Wait for vWLC to provision
  local_action: wait_for host={{ wlc_vm_ip_address }} port=443 delay=10 timeout=600
  sudo: false
- name: Remove DHCP reservation
  blockinfile:
    dest: '{{ wlc_dhcpd_conf_path }}'
    marker: "# {mark} ANSIBLE MANAGED BLOCK - {{ wlc_vm_name }} {{ wlc_vm_ip_address }}"
    content: ""
  become: yes
  when: not wlc_vm_persist_dhcp_reservation
  notify:
    - stop vmware networking
    - start vmware networking
- name: Remove TFTP plist file
  file: >
    path="{{ wlc_tftp_plist }}"
    state=absent
  notify:
    - stop system tftp daemon
    - restore previous system tftp daemon
  become: yes
- name: Remove TFTP config file
  file: >
    path="{{ wlc_tftp_path }}/ciscowlc.cfg"
    state=absent
  notify:
    - stop system tftp daemon
    - restore previous system tftp daemon
  become: yes
{% endraw %}
{% endhighlight %}

The playbook is configured wait for the vWLC virtual machine to come online with the `name: Wait for WLC to provision` task.  This task attempts to establish a TCP connection to port 443 on the vWLC virtual machine service port IP address. 

> You might be tempted to use port 22 to determine if the vWLC virtual machine is fully provisioned.  The vWLC leaves port 22 open during the AutoInstall process allowing the task to establish a TCP connection, hence using port 22 would mean playbook would think provisioning has finished at a much earlier time.  Port 443 (HTTPS) is only activated once provisioning is fully complete, hence gives a more reliable indication that provisioning is complete.

Once the provisioning is completed, a series of cleanup tasks takes place.  

This cleanup removes the DHCP reservation (by default this task is skipped unless `wlc_vm_persist_dhcp_reservation` is set to `no`), the TFTP plist file, and the TFTP WLC configuration file.    

## Wrap Up
Well this has been a very long article, but hopefully you have some good insights into how you Ansible can automate deployment of the Cisco Virtual Wireless Controller.

If you are like me and regularly need a vWLC instance running in a lab, development or demonstration environment, this playbook should save you a lot of time.

Along the way I've also shown you a few of the internals that support VMWare Fusion and how you can use those to automate certain tasks.  Hopefully you've also picked up a few Ansible tricks that you can apply to your own automation/deployment scenarios.    
