# What is this?

This is a document outlining an opinionated process for installing a tester AAP environment during a Proof of Concept (PoC) or similar situation. It takes you through the entire process and vaults all local admin passwords so as to not leave password data in plaintext.

The only software being used in this process that does not come with a minimal RHEL install is wget and of course ansible-core.  
**There are no external scripts to get, and no repos to clone.**

This document presumes an AAP 2.7 VM-based containerized install in the single-machine [Growth Topology](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/plan-ref_cont_a_env_a).

# Why don’t we automate this?

- Good candidates for automation are things that need to be done frequently. This is only done occasionally (by customers; Red Hatters who do this all the time is another matter).  
- A PoC is an opportunity to learn how AAP works. You won't learn anything about how the install works if you obscure the process away.  
- There is no automated process for this which is covered by an AAP support agreement. Therefore you would need to utilize unsupported code. Even validated content like `infra.aap_utilities` does not fall under any support agreement.  
- A typical participant in a PoC doesn’t know Ansible yet. It’s not a good time to get into writing Ansible just to install the product.

# This is pretty long...

There are less complex methods of doing this, but they tend to involve manual effort editing files (longer time taken overall) and less care in managing credentials. We are striking a balance here.

# Install procedure

## Prerequisites
**Before starting**, ensure that the machine to be used meets minimum requirements:

- RHEL 9.6+ or RHEL 10, no other Linux distributions supported  
- A non-root user has been created for AAP to run under. It should have sudo permission.  
- At least 4 CPU cores and 16gb RAM  
- At least 80gb available under `/var`  
- At least 10gb available under `/tmp`  
- At least 15gb available in the directory you intend to run the installer from (likely to be `/home`)  
- 100gb is acceptable if all storage is under /  
- Storage is SSD-based (3000 IOPS)

## Install steps
1. SSH into the machine and become the AAP user if not logged into it directly.  
2. Ensure the machine is on a supported RHEL version  
   1.  `cat /etc/redhat-release`  
3. Ensure the machine has at least 16gb of memory to proceed  
   1.  `free -g`  
4. Ensure the machine has free space according to the above guidelines  
   1. `df -h`  
5. Install required software for the install process  
   1.  `sudo dnf install ansible-core wget`  
6. Access the downloads page for AAP (adjust to appropriate RHEL release)  
   1. [https\://access.redhat.com/downloads/content/480/ver=2.7/rhel---10/2.7/x86\_64/product-software](https://access.redhat.com/downloads/content/480/ver=2.7/rhel---10/2.7/x86_64/product-software)   
7. Right-click the “Download Now” button next to the release you want and select “Copy link address”  
   1. The link will only remain valid as long as the download page stays active  
   2. Use the bundle installer to avoid repo issues unless very bad internet connection  
8. Download the installer  
   1.  `wget "<paste from step 2>" -O installer.tar.gz`  
9. Validate checksum against the downloads page  
   1.  `sha256sum installer.tar.gz`  
10. Extract the installer  
    1.  `tar -xzf installer.tar.gz`  
11. Change directory into the extracted installation directory  
    1.  `cd ansible-automation-platform[...]`  
12. Copy the example inventory  
    1.  `cp inventory-growth inventory-poc`  
13. Edit the inventory to match desired topology  
    1.  `vi inventory-poc`  
    2. Do not use ‘localhost’ as the hostname, even if the install machine and the controller machine are the same, use the actual hostname  
    3. Replace all instances of “[aap.example.org](http://aap.example.org)” with the actual DNS name of the machine.  
       1. In vi, to quickly replace the placeholder host you can do something like: `:%s/aap.example.org/ip-10-0-28-0.us-east-2.compute.internal/g`  
    4. Remove all lines for setting a password.  
       1. In vi, to quickly remove all such lines, you can do this:  
          `:g/password/d`  
    5. You should end up with something like this (comments removed and some repetitious lines snipped out):
       ```ini
       [automationgateway]
       ip-10-0-28-0.us-east-2.compute.internal
       [automationcontroller]
       ip-10-0-28-0.us-east-2.compute.internal
       # …snip…
       [database]
       ip-10-0-28-0.us-east-2.compute.internal
       [all:vars]
       ansible_connection=local
       postgresql_admin_username=postgres
       bundle_install=true
       bundle_dir='{{ lookup("ansible.builtin.env", "PWD") }}/bundle'
       redis_mode=standalone
       # …snip…
       automationmetrics_pg_host=ip-10-0-28-0.us-east-2.compute.internal
       automationmetrics_controller_read_pg_host=ip-10-0-28-0.us-east-2.compute.internal
       ```
14. Create the admin password  
    1. Decide on two passwords:  
       1. The admin password for all the AAP components, we will keep them identical for this  
       2. The vault password for encrypting/decrypting that value on disk  
       3. They can be the same password if desired (recommended)  
    2. Run: `ansible-vault encrypt_string`  
    3. Type vault password (remember/store this somewhere secure) (you will be asked to enter twice)  
    4. You are about to type a password and it’s going to show up as plain text. If you are on a screen share, you may want to stop temporarily.  
    5. Type admin password (may be the same) \- do not hit Enter afterwards  
    6. Type ctrl+d twice  
    7. Output looks like:  
       ```
       Encryption successful
       !vault |
                 $ANSIBLE_VAULT;1.1;AES256
                 64363463653439386465656462346164613335353831396637666432383333386163306235373234
                 6163386531363734373463353563313731613435656162660a343461373533363762663136353339
                 37623466653966643265326234363538666131386334313734346363393132383366326562666339
                 3864303761656466340a336237636331616538326537393933663064356632616438316332643166
                 6662
       ```
    8. Save the output starting with the `!vault` part as an environment variable with single quotes around it
       ```
       export vault='!vault |
                 $ANSIBLE_VAULT;1.1;AES256
                 62656633613030343666303439386137316232666664326662666332326435353266383361333439
                 6365343830376364393861343530353832383638343735660a636532623061616138376138653166
                 38323438653361323265343265633961366439306232333266383031306362343836393530626432
                 6330626637623432370a343261316464626465623663333035313033373762303861356265373535
                 3232'
       ```
    9. Clear the terminal to resume screen sharing if applicable  
15. Create the secrets file  
    1. Scaffold the secrets file by pulling all the password lines out of the inventory template  
       1.  `grep password inventory-growth > secrets-poc.yml`  
    2. Replace all the placeholder values with the Vault data from earlier (this also converts to YAML formatting)  
       1.  `perl -pi -e 's/=<set your own>/: $ENV{vault}/g' secrets-poc.yml`  
    3. Example result:
       ```yaml
       postgresql_admin_password: !vault |
                 $ANSIBLE_VAULT;1.1;AES256
                 62656633613030343666303439386137316232666664326662666332326435353266383361333439
                 6365343830376364393861343530353832383638343735660a636532623061616138376138653166
                 38323438653361323265343265633961366439306232333266383031306362343836393530626432
                 6330626637623432370a343261316464626465623663333035313033373762303861356265373535
                 3232
       gateway_admin_password: !vault |
                 $ANSIBLE_VAULT;1.1;AES256
                 62656633613030343666303439386137316232666664326662666332326435353266383361333439
                 6365343830376364393861343530353832383638343735660a636532623061616138376138653166
                 38323438653361323265343265633961366439306232333266383031306362343836393530626432
                 6330626637623432370a343261316464626465623663333035313033373762303861356265373535
                 3232
       gateway_pg_password: !vault |
                 $ANSIBLE_VAULT;1.1;AES256
                 62656633613030343666303439386137316232666664326662666332326435353266383361333439
                 6365343830376364393861343530353832383638343735660a636532623061616138376138653166
                 38323438653361323265343265633961366439306232333266383031306362343836393530626432
                 6330626637623432370a343261316464626465623663333035313033373762303861356265373535
                 3232
       # and so on...
       ```
16. Run the installer  
    1. `ansible-playbook ansible.containerized_installer.install -bJK -i inventory-poc -e @secrets-poc.yml`   
    2. You will be prompted for the vault password from earlier  
    3. The `-K` option is only needed if you are a user that needs to provide a password when using sudo.  
    4. This will take a while. As long as the preflight tasks succeed and you start seeing the actual hostname as the task targets, this is a good time to take a break or move on to something else.

# Applying the subscription
A specific user has most likely been given a trial entitlement. That user should be able to simply access AAP once install has completed (using the provided admin password) and follow the prompts to log in with their Red Hat account. The subscription can be applied from there without creating any manifest file.

If the login method does not work, the documentation on how to make a subscription manifest file can be found [here](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.4/html/red_hat_ansible_automation_platform_operations_guide/assembly-aap-obtain-manifest-files#assembly-aap-obtain-manifest-files).
