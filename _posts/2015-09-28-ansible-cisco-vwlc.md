---
layout: post
title: Deploying the Cisco Virtual Wireless Controller using Ansible
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
published: false
---

Outside of Docker, <a href="http://www.ansible.com" target="_blank">Ansible</a> seems to be one of the hottest DevOps tools these days.  

Although Ansible is most often used for deploying infrastructure and applications or orchestrating continuous delivery workflows, it can be used to automate almost any task.

In this article, I will describe how to deploy Cisco's Virtual Wireless Controller (vWLC) on VMWare Fusion using the OVA appliance image that is generally available to registered CCO users.  

I will create an Ansible playbook that will automate all of the manual tasks you would normally take to get the vWLC up and running.     

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

## Workflow

> <a href="https://github.com/cloudhotspot/ansible-cisco-vwlc" target="_blank">I've published an Ansible playbook on Github</a>, and the rest of this article will discuss the workflow and each of the specific tasks required.

The high-level workflow is as follows:

- Prepare destination folders for the virtual machine
- Deploy the virtual machine from the OVA appliance image
- Configure the inbuilt OS X TFTP daemon to serve an appropriate configuration file for the controller
- Configure the VMWare DHCP server
- Power on the vWLC and clean up once provisioning is complete  

## Preparing the Virtual Machine Destination folders

Deploying the OVA image sounds simple enough but to make the playbook fairly idiot-proof and user friendly, we need to consider a couple of what-if scenarios:

- What if we've already deployed the image to the desired location?
- What if a virtual machine is running at the desired location? 

The first set of tasks is described in the `prepare_vm_folder.yml` play and focuses on creating the folder for the destination virtual machine, taking into account the scenarios listed above.

{% highlight yaml %}
{% raw %}
---
- name: Check if VM already exists
  hosts: localhost
  connection: local
  vars_files:
    - vm_vars.yml
  tasks: 
    - name: Check VM path
      stat: path='{{ vm_safe_dst_full_path }}'
      register: vm_exists
      changed_when: False
    - name: Fail if VM path exists
      fail: msg="VM already exists.  Please set vm_overwrite variable to any value to overwrite the existing VM"
      when: (vm_exists.stat.isdir is defined) and (vm_overwrite is not defined)
    - name: Get VMX path if existing VM is running
      shell: "'{{ vmrun_path }}' list | grep -F '{{ vm_safe_dst_full_path }}' || true"
      register: vmx_path
      when: vm_exists.stat.isdir is defined
      changed_when: False
    - name: Stop VM if it is running
      shell: "'{{ vmrun_path }}' stop '{{ vmx_path.stdout }}'"
      when: vmx_path is defined and vmx_path.stdout != ""
    - name: Remove existing VM path
      file: path='{{ vm_safe_dst_full_path }}' state=absent
    - name: Create VM path
      file: path='{{ vm_safe_dst_full_path }}' state=directory
{% endraw %}
{% endhighlight %}

There's a few variables in there which are included in the `vm_vars.yml` file:

{% highlight yaml %}
{% raw %}
# Input values, modify as required
vm_name: "wlc01"
vm_destination: "/Users/jmenga/Virtual Machines.localized"
vmrun_path: "/Applications/VMware\ Fusion.app/Contents/Library/vmrun"
ova_source: "/Users/jmenga/Downloads/AIR-CTVM-K9-8-0-120-0.ova"

# Calculated values, do not modify
vm_padded_destination: "{{ vm_destination }}/"
vm_safe_dst: "{{ vm_padded_destination | dirname }}"
vm_safe_dst_full_path: "{{ vm_safe_dst }}/{{ vm_name }}.vmwarevm"
{% endraw %}
{% endhighlight %}

Here's a description of what's happening in the `prepare_vm_folder.yml` play above:

### Checking the Target VM Location
I first check if the desired VM location already exists in the `name: Check VM path` task.  Note I'm using the following convention to create this location:

{% raw %}
`{{ vm_destination }}/{{ vm_name }}.vmwarevm`  
{% endraw %}

So the VM location will be `/Users/jmenga/Virtual Machines.localized/wlc01.vmwarevm` using the input values shown. 

Note I'm using a calculated `vm_safe_dst_full_path` variable, which is derived from the above convention.  This variable is manipulated to ensure we get the correct full path without any duplicate forward slashes. 

If the VM location already exists, the entire playbook is configured to fail in the `name: Fail if VM path exists` task, unless the `vm_overwrite` variable is defined with any value (note the heavy use of Ansible conditionals using the `when` clause).  

This approach protects you from accidentally overwriting an existing virtual machine, but still allows you to explicitly overwrite it if that is your intention as demonstrated below:

`$ ansible-playbook site.yml --extra-vars vm_overwrite=true`

### Creating the VM Location
Before creating the VM location I check if there is an existing VM running (assuming the VM location already exists and `vm_overwrite` has been defined).  As the intention in this scenario is to overwrite an existing VM, we need to first stop the VM (if it is running) in order to remove the existing VM folder and files.

To do this, I use the `vmrun list` command which is included as part of the VMWare Fusion application. This is defined in the `name: Get VMX path if existing VM is running` task, using good old `grep` to extract the full path of the running VM vmx file at the target VM location.

Here's an example of the full output of the `vmrun list` command:

{% highlight console %}
$ /Applications/VMware\ Fusion.app/Contents/Library/vmrun list
Total running VMs: 1
/Users/jmenga/Virtual Machines.localized/wlc01.vmwarevm/wlc01.vmx
{% endhighlight %}

Next I stop the VM (only if it is running) in the `name: Stop VM if it is running` task, using the `vmrun stop` command.  

Finally, I can safely remove the existing VM location (if it previously existed) using the `name: Remove existing VM path` task and create the target VM location using the `name: Create VM path` task.

> Pre-creating the target VM location alters the behaviour of the `ovftool`, which is used to deploy the virtual machine from the OVA image, disucssed in the next section.

## Deploying the Virtual Machine from the OVA Image

Now we are ready to deploy the virtual machine from the OVA image.  These tasks are defined in the `deploy_ova.yml` play:

{% highlight yaml %}
{% raw %}
---
- name: Verify ovftool is installed
  hosts: localhost
  connection: local
  tasks:
    - name: Check for ovftool
      shell: pkgutil --pkgs | awk '/com.vmware.ovftool.application/'
      register: pkgutil_ovftool
      changed_when: False
    - name: Fail if VMWare OVF Tools are not installed
      fail: msg="VMWare OVF Tools are required.  Please install and retry."
      when: 'not {{ pkgutil_ovftool.stdout | match("com.vmware.ovftool.application") }}'
    - name: Get ovftool path
      shell: pkgutil --files com.vmware.ovftool.application | grep -FE 'ovftool$'
      register: ovftool_path
      changed_when: false

- name: Extract OVA
  hosts: localhost
  connection: local
  vars_files:
    - vm_vars.yml
  tasks:
    - name: Extract OVA using ovftool
      command: "'/{{ ovftool_path.stdout }}' '{{ ova_source }}' '{{ vm_safe_dst_full_path }}/{{ vm_name }}.vmx'"
    - name: Configure Ethernet0 as Share with my Mac
      lineinfile: >
        dest='{{ vm_safe_dst_full_path }}/{{ vm_name }}.vmx'
        regexp='^ethernet0.connectionType ='
        line='ethernet0.connectionType = "nat"'
{% endraw %}
{% endhighlight %}

### The ovftool utility
<a href="" target="_blank">VMWare provides a free tool called `ovftool` to registered VMWare users</a>, and this tool must be installed to deploy virtual machines from OVA files using the command line.  

The `name: Verify ovftool is installed` set of tasks verifies `ovftool` is installed and extracts the full path to the `ovftool` executable using various `pkgutil` commands.

### Deploying the OVA Image
The `name: Extract OVA` set of tasks first uses the `ovftool` command to extract the OVA file and deploy it to our target VM location.  The `ova_source` input variable defines the source OVA image (<a href="https://software.cisco.com/download/release.html?mdfid=284464214&softwareid=280926587&release=8.0.120.0&relind=AVAILABLE&rellifecycle=ED&reltype=latest" target="_blank">this image is available here to registered CCO users</a>) and notice we specify the destination VM vmx file. 

> If you run this command and the destination VMX parent folder does not exist, ovftool behaves difficultly and creates another folder in the format {% raw %}`<vm name>.vmwarevm`{% endraw %} under the specified parent folder and then places the vmx file in this folder.  To avoid this behaviour, you must precreate the target VM parent folder (as was described earlier in this article)

The deployed virtual machine includes all of the files necessary to start the VM.  

Before we start the VM, the `name: Configure Ethernet0 as Share with my Mac` task reconfigures the `ethernet0` network interface connection type to **Share with my Mac** using the very useful `lineinfile` Ansible module.  This results in the following entry in the vmx file:

`ethernet0.connectionType = "nat"`

This setting is important, as it ensures the service port on the vWLC appliance will use internal VMWare Fusion networking and the VMWare Fusion DHCP server.  The other `ethernet1` interface will remain in the default bridged networking mode.

> The Cisco vWLC appliance comes with two network interfaces.  `ethernet0` is the service port and `ethernet1` is the management port that connects access points. 
 
## Configuring the OS X TFTP Server
OS X ships with a TFTP server that is disabled by default.  The `tftp.yml` play defines the various tasks required to configure and enable the TFTP server:

{% highlight yaml %}
{% raw %}
---
- name: Configure TFTP
  hosts: localhost
  connection: local
  vars_files:
    - vm_vars.yml
    - wlc_vars.yml
  tasks: 
    - name: Deploy TFTP plist
      template: 
        src: "templates/tftp.plist.j2" 
        dest: "{{ tftp_plist }}"
        mode: 0644
      become: yes
    - name: Ensure TFTP path exists
      file: 
        path: "{{ tftp_path }}"
        state: directory
        mode: 0777
    - name: Deploy WLC file
      template:
        src: "{{ wlc_config_file | default('templates/ciscowlc.cfg.j2') }}"
        dest: "{{ tftp_path }}/ciscowlc.cfg"

- name: Start TFTP
  hosts: localhost
  connection: local
  vars_files:
    - vm_vars.yml
  tasks:
    - name: Check if TFTP daemon is running
      shell: launchctl list | awk /com.apple.tftp/
      become: yes
      register: tftp_daemon_status
      changed_when: False
    - name: Stop TFTP daemon
      command: launchctl unload {{ tftp_plist }}
      become: yes
      when: tftp_daemon_status.stdout != ""
    - name: Start TFTP daemon
      command: launchctl load -F {{ tftp_plist }}
      become: yes  
{% endraw %}
{% endhighlight %}

The configuration of the TFTP server is controlled via the file `/System/Library/LaunchDaemons/tftp.plist` file.  In the `name: Deploy TFTP plist` task, I use a Jinja template to configure the relevant settings in the file:

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
    <string>{{ tftp_path }}</string>
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

The template enables the TFTP server by setting the `<key>Disabled</key>` value to `<false/>` and also includes the `tftp_path` user variable to specify the folder that the TFTP server should serve.

The `name: Deploy WLC file` task then deploys the Cisco vWLC configuration file that will served via TFTP.  The playbook allows you to provide your own config file by setting the `wlc_config_file` variable - if this variable is not defined, the playbook deploys a basic configuration derived from the following template:

{% highlight text %}
{% raw %}
# WLC Config Begin <{{ ansible_date_time.date }} {{ ansible_date_time.time }}>

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

# WLC Config End <{{ ansible_date_time.date }} {{ ansible_date_time.time }}>
{% endraw %}
{% endhighlight %}

The template inserts various variables that are defined in the `wlc_vars.yml` file that are self explanatory:

{% highlight yaml %}
{% raw %}
---
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
 
## Configuring the VMWare DHCP Server
Configuring the DHCP server is probably the trickiest part of the playbook.  The play is defined in the `dhcp.yml` file:
 
{% highlight yaml %}
{% raw %}
---
- name: Introspect Virtual Machine information
  hosts: localhost
  connection: local
  roles:
    - yaegashi.blockinfile
  vars_files:
    - vm_vars.yml
  tasks:
    - name: Get vmnet8 IP address
      shell: ifconfig vmnet8 | awk '/inet/{print $2}'
      register: vmnet8_ip_address
      changed_when: False
    - name: Get Ethernet0 MAC address
      shell: cat '{{ vm_safe_dst_full_path }}/{{ vm_name }}.vmx' | awk -F'"' '/ethernet0.generatedAddress = /{print $2}'
      register: vm_mac_address
      changed_when: False

- name: Configure VMWare DHCP
  hosts: localhost
  connection: local
  roles:
    - yaegashi.blockinfile
  vars_files:
    - vm_vars.yml
    - wlc_vars.yml
  handlers:
    - include: handlers/vmware.yml
  tasks:
    - name: Add next-server parameter to vmnet8 DHCP configuration
      lineinfile: >
        dest="{{ dhcpd_conf_path }}"
        line="next-server {{ vmnet8_ip_address.stdout }};"
        insertbefore=BOF
      become: yes
    - name: Add temporary DHCP reservation
      blockinfile:
        dest: '{{ dhcpd_conf_path }}'
        insertafter: EOF
        content: |
          host wlc01 {
            hardware ethernet {{ vm_mac_address.stdout }};
            fixed-address {{ vmnet8_ip_address.stdout | regex_replace('(.*)\..*$', '\\1.127') }};
            option domain-name-servers {{ vmnet8_ip_address.stdout | regex_replace('(.*)\..*$', '\\1.2') }};
            option domain-name localdomain;
            default-lease-time 1200;
            max-lease-time 1200;  
            option routers {{ vmnet8_ip_address.stdout | regex_replace('(.*)\..*$', '\\1.2') }};
          }
      become: yes
      notify:
        - stop vmware networking
        - start vmware networking
{% endraw %}
{% endhighlight %}
 
### Examining VMWare and Virtual Machine Networking
In the `name: Introspect Virtual Machine Information` set of tasks, the `name: Get vmnet8 IP address` task determines the IP address being used for the `Share with my Mac` vmnet8 network adapter.  This interface is connected to the service port of the vWLC virtual machine, and because 
the OS X TFTP daemon binds to all network interfaces, we can specify this IP address as the TFTP server address.

The `name: Get Ethernet0 MAC address` task looks up the `ethernet0.generatedAddress` key in the virtual machine vmx file.  This key contains the MAC address generated for the service port on the vWLC virtual machine:

`ethernet0.generatedAddress = "00:0c:29:0d:ec:56"`

We will use this MAC address to configure a DHCP reservation for the vWLC virtual machine.

> Notice in both of the above tasks `awk` is our friend.

Now before either of these tasks, notice that we actually start the virtual machine in the `name: Start virtual machine` task.  The reason for this is that the virtual machine MAC addresses are not generated until the virtula machine is first started.  Here I use the `vmrun start` command.

### Configuring DHCP
The `name: Configure VMWare DHCP` set of tasks modifies the VMWare DHCP configuration file for the vmnet8 interface.  This configuration file is located at `/Library/Preferences/VMware Fusion/vmnet8/dhcpd.conf` and an example is shown below:

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

First, we add the `next-server` directive to the configuration and specify the IP address of the vmnet8 adapter - this must be placed someware above the VMNET DHCP Configuration section.

Next, we add a DHCP reservation for the vWLC virtual machine to the bottom of the DHCP configuration file as defined in the `name: Add temporary DHCP reservation` task.  The main reason for this is that we want our playbook to start up the virtual machine and wait until it has been provisioned before performing some cleanup tasks.  We know if the virtual machine has provisioned by attempting to connect to IP address of the virtual machine, hence why we need to control and track the IP address of the virtual machine.

Notice that we use a <a href="https://github.com/yaegashi/ansible-role-blockinfile" target="_blank">third-party community module called yaegashi.blockinfile</a>.  This module must first be installed via Ansible Galaxy as follows:

`ansible-galaxy install yaegashi.blockinfile`

Here is an example of the DHCP configuration file with the modifications in place:

{% highlight text %}
{% raw %}
next-server 192.168.232.1;
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

# BEGIN ANSIBLE MANAGED BLOCK
host wlc01 {
  hardware ethernet 00:0c:29:0d:ec:56;
  fixed-address 192.168.232.127;
  option domain-name-servers 192.168.232.2;
  option domain-name localdomain;
  default-lease-time 1200;
  max-lease-time 1200;  
  option routers 192.168.232.2;
}
# END ANSIBLE MANAGED BLOCK
{% endraw %}
{% endhighlight %}

With the modifications made to the DHCP configuration, the `name: Add temporary DHCP reservation` task notifies a couple of handlers, defined in `handlers/vmware.yml`:

{% highlight yaml %}
{% raw %}
---
- name: stop vmware networking
  command: '"/Applications/VMware Fusion.app/Contents/Library/vmnet-cli" --stop'
  become: yes
- name: start vmware networking
  command: '"/Applications/VMware Fusion.app/Contents/Library/vmnet-cli" --start'
  become: yes
{% endraw %}
{% endhighlight %}

These handlers restart VMWare networking, allowing the DHCP configuration changes to take effect.
 
### Virtual Machine AutoInstall and Cleanup
At this point, everything is in place for the vWLC to use the AutoInstall feature:

- TFTP daemon is enabled and configured with a vWLC configuration file
- VMWare DHCP server is configured to advertise the TFTP server IP address and issue a DHCP reservation to vWLC virtual machine

Here is the relevant section from the `vwlc.yml` file:

{% highlight text %}
{% raw %}
- name: Cleanup
  hosts: localhost
  connection: local
  roles:
    - yaegashi.blockinfile
  vars_files:
    - vm_vars.yml
  handlers:
    - include: handlers/vmware.yml
  tasks:
    - name: Wait for WLC to provision
      local_action: wait_for host={{ vmnet8_ip_address.stdout | regex_replace('(.*)\..*$', '\\1.127') }} port=443 delay=10 timeout=600
      sudo: false
    - name: Remove temporary DHCP reservation
      blockinfile:
        dest: '{{ dhcpd_conf_path }}'
        marker: "# {mark} ANSIBLE MANAGED BLOCK"
        content: ""
      become: yes
      notify:
        - stop vmware networking
        - start vmware networking
    - name: Remove next-server parameter from vmnet8 DHCP configuration
      lineinfile: >
        dest="/Library/Preferences/VMware Fusion/vmnet8/dhcpd.conf"
        line="next-server {{ vmnet8_ip_address.stdout }};"
        state=absent
      become: yes
      notify:
        - stop vmware networking
        - start vmware networking
    - name: Stop TFTP daemon if it was not previously running
      command: launchctl unload {{ tftp_plist }}
      become: yes
      when: tftp_daemon_status.stdout == ""
    - name: Remove TFTP config file
      file: >
        path="{{ tftp_path }}/ciscowlc.cfg"
        state=absent
{% endraw %}
{% endhighlight %}

The playbook is configured wait for the vWLC virtual machine to come online with the `name: Wait for WLC to provision` task.  This task attempts to establish a TCP connection to port 443 on the vWLC virtual machine service port IP address.

> You might be tempted to use port 22 to determine if the vWLC virtual machine is fully provisioned.  The vWLC leaves port 22 open during the AutoInstall process allowing the task to establish a TCP connection, hence using port 22 would mean playbook would think provisioning has finished at a much earlier time.  Port 443 (HTTPS) is only activated once provisioning is fully complete.

Once the provisioning is completed, a series of cleanup tasks takes place.  This cleanup removes the various DHCP and TFTP configurations and files, ensuring your VMWare networking configuration is restored to its previous state.