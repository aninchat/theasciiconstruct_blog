---
title: "SD-Access Programmability Part I - creating a Nornir network inventory from DNAC"
date: 2021-12-29T19:38:59+05:30
draft: false
tags: [python, sda, nornir]
description: "In this post, we look at how to generate a Nornir network inventory from DNA-Center, using python."
---

## Introduction and topology

I hear that absence makes the heart grow fonder. Did you miss me?


I wanted to kick this section off with an extensive look at all options to programmatically interact with Cisco DNA Center (DNAC, going forward). But I don't want to re-hash what's already out there - my friend [Robert Csapo](https://twitter.com/robertcsapo) already has an amazing post on Medium for this that you should absolutely read (and he's articulated this far better than I ever could). You can find it [here](https://robertcsapo.medium.com/3-simple-ways-to-use-cisco-dna-center-platform-apis-7eee49b76287). 



> Note: the only thing I'd add to this is that since the time of his writing, there has been an official [Ansible collection]((https://galaxy.ansible.com/cisco/dnac)) released for DNAC  as well as a [Terraform provider](https://registry.terraform.io/providers/cisco-en-programmability/dnacenter/latest/docs) with related resources. If this is your cup of tea, then these are now available as well.


So, instead, we're going to jump right in. My goal through this broader section is to use Python to automate some interesting use cases that I've been thinking of, over the past year or so, for DNAC and SD-Access. 


In this particular post, we're going to build a network inventory from DNAC, that can in turn be leveraged by Nornir and Scrapli. Why build a network inventory, why Nornir and why Scrapli?


Some of the initial use cases I have in mind involve a lot of screen scraping and more specifically, configuration parsing. I could use a northbound API that DNAC exposes for gathering the configuration of network devices in it's inventory (/dna/intent/api/v1/network-device/{networkDeviceId}/config) - but this makes things a little more complicated and slow. I would first need to gather a list of all network devices and their IDs (/dna/intent/api/v1/network-device/), then feed those IDs into the configuration API. Couple this with our current DNAC API rate limits and this approach doesn't scale really well.


Thus, I decided to go the traditional route - screen scraping by connecting to the devices directly via SSH. I still need a list of network devices though - this is where Nornir comes in. Like Ansible, [Nornir](https://nornir.readthedocs.io/en/latest/) is an automation framework but the key difference is that it can be used natively in Python. You're not forced to learn the idiosyncrasies of a domain specific language (that is not relevant to anything outside of it). 


Nornir also allows for concurrency - remember, speed becomes a factor once you think of scaling up your network devices. Between Scrapli and Netmiko, I don't think you can really go wrong with your choice. Scrapli is more modern and was written specifically with speed in mind (the author's own words), thus Scrapli it is. 


For this post, we're going to be using the following topology, which is fully integrated and managed by my DNAC:

![static1](/images/cisco/sda_prog_1/dnac_nornir_inv_1.jpg)

So, the first thing I need is a list of devices from the DNAC inventory. DNAC exposes an API for this, which we will use here. All business APIs for DNAC can be found by navigating to Platform -> Developer Toolkit. An example screenshot below:

![static1](/images/cisco/sda_prog_1/dnac_nornir_inv_2.jpg)



The API that we're interested in is - '/dna/intent/api/v1/network-device/'. DNAC APIs function with the concept of an authorized token. Thus, every API must have an 'X-Auth-Token' header, with the authorized token as it's value. 


This token can be generated using the Authentication API:

![static1](/images/cisco/sda_prog_1/dnac_nornir_inv_3.jpg)



I have the following function that accesses the API (taking DNAC IP address, username and password as an input) and returns the authorized token. 

```python
def get_token_for_dnac(ip_address, username, password):
    """ function to get a valid token for DNAC API
    access. Input needed is DNAC IP address, username
    and password
    """

    # build complete URL first

    token_url = "https://" + ip_address + "/dna/system/api/v1/auth/token"

    # use a POST call to get token

    warnings.filterwarnings("ignore")
    token_call = requests.post(token_url, auth=HTTPBasicAuth(username, password), headers={"Content-Type": "application/json"}, verify=False)

    # return actual token

    return(token_call.json()['Token'])
```




This token can now be used to access all subsequent APIs. Remember, the DNAC token is only valid for 60 minutes post which a new token must be generated again. With this token, I can use the network device API to get a list of all network devices. The following function does this:

```python
def get_network_devices_from_dnac(ip_address, token):
    """ function to get complete list of network
    devices from DNAC using DNACs IP address and 
    the authentication token as an input
    """

    # build complete URL first

    network_device_url = "https://" + ip_address + "/dna/intent/api/v1/network-device/"

    # use a GET call to return this data

    network_device_call = requests.get(network_device_url, headers={"Content-Type":"application/json", "X-Auth-Token": token}, verify=False)

    # return network device list

    return(network_device_call.json()['response'])
```




So, what does the API response look like? Let's take a look at this using pdb (python debugger). I have added a breakpoint just before the return statement. 

```python
(Pdb) l
 49  	
 50  	    # return network device list
 51  	
 52  	    breakpoint()
 53  	
 54  ->	    return(network_device_call.json()['response'])
 55  	
 56  	def parse_device_list(device_list):
 57  	    """ parse through device list returned
 58  	    from DNAC and save it in a new dictionary with
 59  	    relevant information only
```



The response is a dictionary with one key called 'response'. Because the output is too long, I'll simply add a screenshot of it here:

![static1](/images/cisco/sda_prog_1/dnac_nornir_inv_4.jpg)




Thus, the API is simply returning a list of dictionaries, where each dictionary is a network device and it's properties (like software version, IP address and so on). If no network devices exist, an empty list is returned. 


Next up is the code to convert this list of network devices retrieved from DNAC into a dictionary that can be used to convert into a hosts file for Nornir. This function is simply looping through the list of dictionaries we pass into it, and saving the IP address of the network device and the role into another dictionary.

```python
def parse_device_list(device_list):
    """ parse through device list returned
    from DNAC and save it in a new dictionary with
    relevant information only
    """

    parsed_device_list = {}

    # loop through device list and store information
    # in a temporary dictionary

    for device in device_list:

        temp_dict = {}

        # top level key for dictionary is going to be
        # the hostname of the device; doing this
        # explicitly to improve readability of code

        temp_dict[device['hostname']] = {}

        # for nornir hosts file, the hostname should be the
        # IP address of the device

        temp_dict[device['hostname']]['hostname'] = device['managementIpAddress']
        temp_dict[device['hostname']]['groups'] = [device['role'].lower()]

        parsed_device_list.update(temp_dict)

    return parsed_device_list
```



The parsed dictionary we return looks like this:

```python
(Pdb) l
 79  	        temp_dict[device['hostname']]['groups'] = [device['role'].lower()]
 80  	
 81  	        parsed_device_list.update(temp_dict)
 82  	
 83  	    breakpoint()
 84  ->	    return parsed_device_list
 85  	
 86  	def main():
 87  	    dnac_ip_address = input("Enter the IP address for DNAC: ")
 88  	    dnac_username = input("Enter username for DNAC: ")
 89  	    dnac_password = getpass("Enter password for DNAC: ")
(Pdb) parsed_device_list
{'HQ-Border1.tatooine.com': {'hostname': '192.168.99.1', 'groups': ['distribution']}, 'HQ-Border2.tatooine.com': {'hostname': '192.168.99.2', 'groups': ['distribution']}, 'HQ-Edge-1.tatooine.com': {'hostname': '192.168.1.70', 'groups': ['access']}, 'HQ-Edge-2.tatooine.com': {'hostname': '192.168.1.71', 'groups': ['access']}}
```




A feedback that I got from Dmitry Figol for this specific snippet of code is that there are several instances where I re-use device['hostname']. A good practice here would be to just assign this to a variable and define the complete dictionary inline itself. It would look something like this:

```python
def parse_device_list(device_list):
    """ parse through device list returned
    from DNAC and save it in a new dictionary with
    relevant information only
    """

    parsed_device_list = {}

    # loop through device list and store information
    # in a temporary dictionary

    for device in device_list:

        temp_dict = {}

        device_hostname = device['hostname']
        temp_dict = {device_hostname: {
            'hostname': device['managementIpAddress'],
            'groups': [device['role'].lower()]}
            }

        parsed_device_list.update(temp_dict)

    return parsed_device_list
```



And that's all of the functions that are needed here. The final piece we'll look at is the main function. 


In the main function, we'll take several inputs from the user:


1. IP address of DNAC, along with username and password. This is needed to generate the authentication token for all APIs.

2. Credentials to login to network devices - this will eventually be added to the defaults file that Nornir uses.

3. File path to store the hosts and defaults file that will be generated as part of this script. 


Let's walk through this now - I'll be breaking it into segments.


First, we gather several inputs from the user. A try/except block is used to generate an authentication token from DNAC. If this fails, a custom exception is raised.

```python
def main():
    dnac_ip_address = input("Enter the IP address for DNAC: ")
    dnac_username = input("Enter username for DNAC: ")
    dnac_password = getpass("Enter password for DNAC: ")

    # try to get valid token for DNAC now

    try:
        token = get_token_for_dnac(dnac_ip_address, dnac_username, dnac_password)
        rich.print("[green]Retrieved token from DNAC for subsequent API calls")
    except:
        raise AuthenticationError("Error getting token for DNAC. Please try again with correct credentials\n")

    # get username/password for network devices along with 
    # list of network devices from DNAC

    rich.print("[blue]\n=============================================\n")
    device_username = input("Enter the username for network devices: ")
    device_password = getpass("Enter the password for network devices: ")
```



Next, we get the list of network devices from DNAC using the function we defined earlier. If no devices were found, we return out. 


Once we have this list, we parse through it (again, using the function we defined earlier) to convert it into a dictionary that can be used to generate a Nornir hosts file.

```python
    dnac_device_list = get_network_devices_from_dnac(dnac_ip_address, token)

    # device list could be empty if no devices are present in DNAC
    # return if empty

    if dnac_device_list:
        rich.print("[green]Retrieved device list from DNAC")
    else:
        rich.print("[red]No devices found in DNAC inventory")
        return

    # parse through device list to build nornir hosts file

    rich.print("[blue]\n=============================================\n")
    hosts_dict = parse_device_list(dnac_device_list)
```



Next, we take this dictionary and convert it into a hosts.yaml file for Nornir to use. Again, this is inside a try/except block to ensure we catch any potential issues with file opening and write permissions. Once the file is open, the yaml.dump method is used to write into a yaml file - this method takes a Python dictionary and converts into a yaml format. This is why we had our network devices stored as a dictionary earlier.

```python
    # convert hosts file into yaml and save it in user specified directory

    try:
        hosts_file_path = input("Please enter complete path where Nornir hosts file should be saved: ")
        rich.print("[green]Attempting to create Nornir hosts file")
        hosts_file = open(hosts_file_path, "w")
        hosts_file.write("---\n\n")
        yaml.dump(hosts_dict, hosts_file)
        hosts_file.close()
        rich.print("[green]Created Nornir hosts file")
    except:
        rich.print("[red]Could not open file. Check path and/or file, directory permissions")
        return
```


We repeat the same process for the defaults file as well. Let's execute the entire code now. 

```
(Nornir2.5) aninchat@aninchat-ubuntu:~/Automation/Python/Nornir2.5_Projects$ python create_nornir_inventory_from_dnac.py 
Enter the IP address for DNAC: 10.104.233.91
Enter username for DNAC: admin
Enter password for DNAC: 
Retrieved token from DNAC for subsequent API calls

=============================================

Enter the username for network devices: aninchat
Enter the password for network devices: 
Retrieved device list from DNAC

=============================================

Please enter complete path where Nornir hosts file should be saved: /home/aninchat/Automation/Python/Nornir2.5_Projects/Sample/hosts.yaml
Attempting to create Nornir hosts file
Created Nornir hosts file

=============================================

Please enter complete path where Nornir defaults file should be saved: /home/aninchat/Automation/Python/Nornir2.5_Projects/Sample/defaults.yaml
Attempting to create Nornir defaults file
Created Nornir defaults file
```


This correctly generates the following files in the specified path:

```
(Nornir2.5) aninchat@aninchat-ubuntu:~/Automation/Python/Nornir2.5_Projects/Sample$ tree
.
├── config.yaml
├── defaults.yaml
├── group.yaml
├── hosts.yaml
├── nornir.log
└── sample_nornir_script.py
```


Finally, let's run a very simple Nornir script that proves this works. The goal is to just run the command 'show vrf' against the inventory that we just created for Nornir.

```python
from nornir import InitNornir
from nornir_scrapli.tasks import send_command
from nornir_utils.plugins.functions import print_result

nr = InitNornir(config_file='config.yaml')

result = nr.run(task=send_command, command='show vrf')
print_result(result)
```


The output of this is:

```
(Nornir2.5) aninchat@aninchat-ubuntu:~/Automation/Python/Nornir2.5_Projects/Sample$ python sample_nornir_script.py 
send_command********************************************************************
* HQ-Border1.tatooine.com ** changed : False ***********************************
vvvv send_command ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
  Name                             Default RD            Protocols   Interfaces
  Group1                           1:4101                ipv4        Lo1021
                                                                     Vl3001
                                                                     LI0.4101
  Group2                           1:4099                ipv4        Vl3002
                                                                     LI0.4099
  Group3                           1:4100                ipv4        Vl3003
                                                                     LI0.4100
  Mgmt-vrf                         <not set>             ipv4,ipv6   Gi0/0

  Platform iVRF Name               iVRF Id               Interfaces
  __Platform_iVRF:_ID00_           0                     LI3/2
^^^^ END send_command ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* HQ-Border2.tatooine.com ** changed : False ***********************************
vvvv send_command ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
  Name                             Default RD            Protocols   Interfaces
  Group1                           1:4101                ipv4        Lo1021
                                                                     Vl3005
                                                                     LI0.4101
  Group2                           1:4099                ipv4        Vl3006
                                                                     LI0.4099
  Group3                           1:4100                ipv4        Vl3007
                                                                     LI0.4100
  Mgmt-vrf                         <not set>             ipv4,ipv6   Gi0/0

  Platform iVRF Name               iVRF Id               Interfaces
  __Platform_iVRF:_ID00_           0                     LI3/2
^^^^ END send_command ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* HQ-Edge-1.tatooine.com ** changed : False ************************************
vvvv send_command ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
  Name                             Default RD            Protocols   Interfaces
  Group1                           <not set>             ipv4        LI0.4101
                                                                     Vl1021
  Group2                           <not set>             ipv4        LI0.4099
  Group3                           <not set>             ipv4        LI0.4100
  Mgmt-vrf                         <not set>             ipv4,ipv6   Gi0/0
^^^^ END send_command ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* HQ-Edge-2.tatooine.com ** changed : False ************************************
vvvv send_command ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
  Name                             Default RD            Protocols   Interfaces
  Group1                           <not set>             ipv4        LI0.4101
                                                                     Vl1021
  Group2                           <not set>             ipv4        LI0.4099
  Group3                           <not set>             ipv4        LI0.4100
  Mgmt-vrf                         <not set>             ipv4,ipv6   Gi0/0
^^^^ END send_command ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```


The entire code can be found on github, [here](https://github.com/aninchat/nornir_inventory_from_dnac).


Being a network automation beginner (beginner is probably an understatement too), I wanted to write about potential use cases I see in my current line of work (DNAC and SD-Access). I also wanted to break down my own thinking and in particular, feedback that I get from architects like Dmitry that helps me improve my thinking when writing/reviewing code. 


I hope this was informative.