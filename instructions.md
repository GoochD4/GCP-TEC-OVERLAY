
# Create Network Overlay and Configure SD-WAN Components

In previous labs, we built a Cloud on-ramp using two FortiGates deployed as a High Availability pair sandwiched between two Load Balancers.  We also built a remote site using a single FortiGate and Ubuntu server.  The next step is to securely connect the remote location with the cloud on-ramp.  In the following excercises, we will configure the IPsec overlay.  BGP will be used to share routes between locations.  Once the overlay is in place, we will configure SD-WAN to monitor SLA

* Network Diagram

    ![diagram1](images/network-diagram.jpeg)

***

## Chapter 1 - Build Network Overlay

***[Deployment exercise - estimated duration 45min]***

<details>

<summary>In this chapter, we will create the dialup IPsec VPN hub on the on-ramp FortiGate and configure the remote site to connect to it.   </summary>

### Task 1 - Add Forwarding Rule to the Load Balancer

#### Tidbit - There is no way to add forwarding rules to the Load Balancer using the GUI Console.  For this step, we will need to open a cloud shell and use the Google SDK.  As you will see in the following steps, we need to create forwarding rules to allow the load balancer to forward UDP ports 500 and 4500 to the FortiGate

* We will be using the below Google sdk command to create the forwarding rule.  Copy and paste the below command into your favorite text editor.  The next few steps will help us get the require environment variables (project-id, lb-ip)

```sh
gcloud compute forwarding-rules create udp-ipsec --backend-service=projects/<project-id>/regions/us-central1/backendServices/fgt-qlabs-bes-elb-us-central1 --address=<lb-ip> --ports=500,4500 --region=us-central1 --ip-protocol=UDP
```

* From the GCP console dashboard, select click on the cursor **>_** at the top of the screen.  This will open a **CLOUD SHELL Terminal** at the bottom of the screen.

    ![overlay1](images/open-shell.jpg)

* Get the project-id by copying it from the cloud shell prompt.  We do not need the open and close parentheses.

    ![overlay2](images/get-project.jpg)

* Get the lb-ip.  Under hamburger menu  select MORE PRODUCTS > Network services > **Load balancing**.  In the center of the screen, Click on LOAD BALANCERS > **fgt-qlabs-bes-elb-us-central1**.

    ![overlay3](images/load-balancing.jpg)

* In the center of the screen, under **Frontend** copy the IP address

    ![overlay4](images/frontend-ip.jpg)

* Once you have the project-id and lb-ip, update the sdk command from earlier and input it into the cloud shell.  Below is an example of what that command should look like.

    ![overlay5](images/sdk-sample.jpg)

* You should now see the new rule under **Frontend**

### Task 2 - Configure IPsec VPN Hub on cloud on-ramp FortiGate

#### Tidbit - The GCP Load Balancer does not perform Destination NAT on inbound traffic to the FortiGate, meaning that the Destination IP address in the UDP port 500 IPsec packets are set to the Load Balancer's external IP address.  We will need to add this IP as secondary to the "WAN" interface (port1) on FortiGate

* Log into the active FortiGate of the cloud on-ramp HA pair.  On the left pane, select **Network** > **Inerfaces**.  Click on port1 and select **Edit**  Under Address, toggle the **Secondary IP address** button and input the lb-ip from earlier.

    ![overlay6](images/secondary-ip.jpg)

* Open a CLI console in the active FortiGate by clicking on the cursor **>_** icon or using SSH to the public management IP.  Copy the below configurations into your favorite text editor and "set local-gw" to the lb-ip. Once completed, copy and paste thes configurations into the cli console.

```sh
config vpn ipsec phase1-interface
    edit HUB1
        set type dynamic
        set interface port1
        set ike-version 2
        set local-gw <lb-ip>
        set peertype any
        set net-device disable
        set mode-cfg enable
        set proposal aes256-sha256
        set add-route disable
        set dpd on-idle
        set ipv4-start-ip 10.10.1.2
        set ipv4-end-ip 10.10.1.25
        set ipv4-netmask 255.255.255.0
        set psksecret Fortinet1!
        set dpd-retryinterval 60
    next
end


config vpn ipsec phase2-interface
    edit HUB1
        set phase1name HUB1
        set proposal aes256-sha256
    next
end
config system interface
   edit HUB1
        set vdom root
        set ip 10.10.1.254 255.255.255.255
        set allowaccess ping
        set type tunnel
        set remote-ip 10.10.1.1 255.255.255.0
        set snmp-index 18
        set interface port1
    next
end

config firewall policy
    edit 0
        set name ipsec-in
        set srcintf HUB1
        set dstintf port2
        set action accept
        set srcaddr all
        set dstaddr all
        set schedule always
        set service ALL
        set nat enable
    next
    edit 0
        set name ipsec-internet
        set srcintf HUB1
        set dstintf port1
        set action accept
        set srcaddr all
        set dstaddr all
        set schedule always
        set service ALL
        set nat enable
    next
end

```

#### Tidbit - Notice that we are using mode config here.  This will result in IP addresses being dynamically assigned to the IPsec interface on the remote sites.  Make a mental note of the "ip4v-start-ip" and "ipv4-stop-ip".  This range will be used later to configure BGP

### Task 3 - Configure IPsec VPN on remote site

* Open a CLI console in the FortiGate by clicking on the cursor **>_** icon or using SSH to the public management IP.  Copy the below configurations into your favorite text editor and "set remote-gw" to the lb-ip. Once completed, copy and paste thes configurations into the cli console.

```sh
config vpn ipsec phase1-interface
    edit HUB1
        set interface port1
        set ike-version 2
        set peertype any
        set net-device disable
        set mode-cfg enable
        set proposal aes256-sha256
        set add-route disable
        set dpd on-idle
        set remote-gw <lb-ip>
        set psksecret Fortinet1!
    next
end


config vpn ipsec phase2-interface
    edit HUB1
        set phase1name HUB1
        set proposal aes256-sha256
        set auto-negotiate enable

    next
end

config firewall policy
    edit 0
        set name ipsec-out
        set srcintf port2
        set dstintf HUB1
        set action accept
        set srcaddr all
        set dstaddr all
        set schedule always
        set service ALL
        set nat disable
    next
    edit 0
        set name ipsec-in
        set srcintf HUB1
        set dstintf port2
        set action accept
        set srcaddr all
        set dstaddr all
        set schedule always
        set service ALL
        set nat enable
    next
end
```

* Run the below commands to ensure that the tunnels are up and functioning proplerly.

```sh
get vpn ipsec tunnel summary
diagnose vpn ike gateway list name HUB1
```

#### Useful Link - https://community.fortinet.com/t5/FortiGate/Troubleshooting-Tip-IPsec-VPNs-tunnels/ta-p/195955

### Task 4 - Configure BGP on cloud on-ramp FortiGate

* Copy the below BGP configurations and paste them into the active FortiGate's CLI console.

```sh
config router bgp
    set as 65400
    set ibgp-multipath enable
    set additional-path enable 
    set additional-path-select 4
    config neighbor-group
        edit HUB1
            set remote-as 65400
            set additional-path both
            set adv-additional-path 4
            set route-reflector-client enable
        next
    end
    config neighbor-range
        edit 1
            set prefix 10.10.1.0 255.255.255.0
            set max-neighbor-num 20
            set neighbor-group HUB1
        next
end

    config network
        edit 1
            set prefix 172.20.1.0 255.255.255.0
        next
    end
end
```

#### Tidbit - note the prefix setting under "config neighbor-range".  The dynamic IP addresses assigned to the remote site IPsec VPN interfaces fall within that range, meaning that this router will accept any bgp peer request from those remote sites.

### Task 4 - Configure BGP on remote FortiGate

* Copy the below BGP configurations and paste them into the FortiGate's CLI console.

```sh
config router bgp
    set as 65400
    set ibgp-multipath enable
    set additional-path enable
    set additional-path-select 4
    config neighbor
        edit 10.10.1.254
            set remote-as 65400
            set additional-path receive
        next
    end
    config network
        edit 1
            set prefix 192.168.129.0 255.255.255.128
        next
    end
end

```

* Ensure that BGP peers are established and that routes are being shared on both the hub and remote site.

```sh
get router info bgp summary
get router info routing-table bgp
```

* Below are the expected outputs

    ![overlay7](images/hub-bgp-sum.jpg)

    ![overlay8](images/spoke-bgp-sum.jpg)    

* Ensure continuity from Hub by pinging the remote site Ubuntu server.

```sh
execute ping 192.168.129.3
```

* Ensure continuity from remote site by pinging the Hub Ubuntu server.

```sh
execute ping 172.20.1.5
```

***

</details>

## Chapter 2 - Configure SD-WAN

***[Make it work - estimated duration 15min]***

<details>

<summary>Now that we have configured the overlay, we will add the "WAN" interface (port1) and the IPsec HUB1 interface to SD-WAN.  We will then create SLA monitoring in the remote site.</summary>

### Task 1 - Add interfaces to SD-WAN

#### Tidbit - In FortiOS, interfaces which already have policies attached to them are precluded from being added to SD-WAN.

* On the remote FortiGate, delete the existing firewall policies by opening a console connection and inputting the below configuration.  **Note: This will cause the IPsec tunnel to go down**

```sh
config firewall policy
delete 1
delete 2
delete 3
delete 4
end
```

* Navigate to **Network > SD-WAN** and click on **Create New > SD-WAN Member** From the **Interface** drop down, choose **port1**.  Leave all other values as default.

    ![overlay9](images/new-sdwan-member.jpg)

* Navigate to **Network > SD-WAN** and click on **Create New > SD-WAN Member** From the **Interface** drop down, choose **HUB1**.  In the **SD-WAN Zone** drop down, click **Create** and name the new zone "overlay".  Leave all other values as default and click **OK** 

    ![overlay10](images/hub1-sdwan.jpg)

* Open a Console connection and add the below firewall policies.

```sh
config firewall policy
    edit 0
        set name overlay-out
        set srcintf port2
        set dstintf overlay
        set action accept
        set srcaddr all
        set dstaddr all
        set schedule always
        set service SMTP
        set nat enable
    next
    edit 0
        set name vip-in
        set srcintf virtual-wan-link
        set dstintf port2
        set action accept
        set srcaddr all
        set dstaddr ubu-serv
        set schedule always
        set service HTTP
        set nat enable
    next
    edit 0
        set name overlay-in
        set srcintf overlay
        set dstintf port2
        set action accept
        set srcaddr all
        set dstaddr all
        set schedule always
        set service ALL
        set nat enable
    next
    edit 0
        set name port2-out
        set srcintf port2
        set dstintf virtual-wan-link
        set action accept
        set srcaddr all
        set dstaddr all
        set schedule always
        set service ALL
        set nat enable
    next
end
```

#### Tidbit - After interfaces have been added to SD-WAN, Policies are configured using the SD-WAN zone.  This simplifies policy configuration once multiple interfaces are added to the zones

#### useful link - https://docs.fortinet.com/document/fortigate/7.2.3/administration-guide/942095/sd-wan-members-and-zones

### Task 2 - Create SLA monitoring

* Navigate to **Network > SD-WAN > Performance SLAs** and select the test named **Default_Google_Search"**.  Click **Edit**. Under **Participants** select **All SD-WAN Members**.  Leave all other values as default and click **OK**.  

    ![overlay11](images/google-sla.jpg)

* You may need to refresh the browser in order to see the SLA measurements.  Click on **Default_Google Search**.  You should now see performance data updating in real time for both the **HUB1** and **port1** interfaces.

    ![overlay12](images/google-mon.jpg)

* In the fires two steps, we used the default Googel performance SLA monitor.  While it's not unheard of to monitor a Public internet site over an IPSec tunnel to the cloud, a more realistic scenario would be to monitor a resource in our own cloud "Data Center"  Below is an example of a custom performance SLA monitoring hour Hub Ubunt Server (created in lab 3).

    ![overlay13](images/ubu-hub-mon.jpg)

#### useful link - https://docs.fortinet.com/document/fortigate/7.2.3/administration-guide/584396/performance-sla

### Task 3 - Create SD-WAN Rules

* Navigate to **Network > SD-WAN > SD-WAN Rules**.  Click **Create new**  Feel free to play around with the Values here.  At a minimum you will need to provide **Name**, **Destination Address or Internet Service**, **Interface selection strategy** and **Interface and/or Zone preference**.  **Note: The minimum required information will change, depending on which selection strategy you choose.  Our example below uses Best Quality, which additionally, requires us to choos a Measured SLA and Quality Criteria**

    ![overlay14](images/oci-rule.jpg)

#### useful link - https://docs.fortinet.com/document/fortigate/7.2.3/administration-guide/716691/sd-wan-rules 

* **Congratulations!** You have completed this course!  Please answer the questions below.

***
  </details>

***

## Quiz


### Question 1

* All GCP features can be configured from the GUI Console  (True or False)

<details> 

<summary>Answer</summary>

* **False** - As we saw with load balancer forwarding rules, some configurations are only available using the gcloud cli.

</details>

## Question 2

* Which hub ipsec phase1-interface setting enables dynamic assignment of IP address to remote peers?
    a) set type dynamic
    b) set add-route disable
    c) set mode-cfg enable
    d) set peertype any

<details> 

<summary>Answer</summary>

* **C** - set mode-cfg enable along with  set ipv4-start-ip, set ipv4-end-ip and set ipv4-netmask are required on the hub to enable this feature.

</details>

## Question 3

* You must use the Zone ID in security policy for any interface which is added to SD-WAN (TRUE or False)

<details> 

<summary>Answer</summary>

* **True** - once an interface is part of SD-WAN, you can no longer assign policy direcly to that interface.

</details>
