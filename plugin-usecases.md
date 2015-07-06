# Chef vRO Plugin Use Cases

## Overall/General Items

 * **IMPROVEMENT:** the auto-population of fields throughout the workflows (data bags, runlists, environments, etc) requires vRO to make repeated HTTP requests to the chef server.  These requests happen after every single keystroke.  Not only does this slow down the vRO UI but it also adds undue work onto the chef server.  Consider a way to delay these HTTP calls until after focus has been changed off the field in question.

 * **IMPROVEMENT:** As stated a few times below, moving to the new validatorless registration that is supported in Chef 12 will make bootstrapping easier and also eliminate some security concerns with shipping and storing the validator private key.

## Add Chef Client and Get Chef Client Details

### 1. Install Chef Client on Nodes

 * Linux: The "Linux/SSH/Install Omnibus Chef Client" workflow works as expected.
 
 * Linux: Knife Bootstrap: After modifying the workflow manually to disable SSL trust verification, and setting an empty runlist, the workflow gets past the error conditions and succeeds provided it is executed against a clean/fresh VM.
 
 * **BUG:** Linux: Knife Bootstrap: It you attempt to run the bootstrap workflow a second time after a failed first attempt, it hangs indefinitely. This is not reproducable by executing `knife bootstrap` manually at the command line.  Even with debug logging enabled, it's unclear what is causing it to hang.
 
 * **BUG:** Linux: Knife Bootstrap: The workflow provides the list of roles, but sets the runlist as "[rolename]" rather than "role[rolename]" - this causes the chef-client to fail, looking for a cookbook called "rolename"
 
 * **GAP:** Linux: Knife Bootstrap: the private key for the <ORG>-validator client must be saved in vRO, meaning it uses the old way of bootstrapping rather than the new client-less method.  This should be updated so the validator private key is not necessary.  If not, the documentation should be updated to indicate that both a user key and the <ORG>-validator key must be registered with vRO; otherwise, some workflows will simply not work (i.e. some required fields are not populated).
 
 * **BUG:** Linux: Knife Bootstrap: because of the old method of registration in use, SSL validation between client and server on first run is necessary, and unless the SSL certificate is trusted, the bootstrap will fail.  There is no way to configure the workflow to disable the node's SSL verify check or to configure SSL certificate trust.  Manually modifying the `Create commands` step of this workflow to add `--node-ssl-verify-mode none` works, but should a input parameter to the workflow. 
 
 * **IMPROVEMENT:** Linux: Knife Bootstrap: This workflow expects to have root access on the "knife runner" host as it creates an /etc/chef directory etc.  This is not necessarily required to do bootstrapping, and many users will find it suboptimal to run this process as a root user on an intermediary host.

 * **IMPROVEMENT:** move knifeRunnerHostname, knifeRunnerUsername, knifeRunnerPassword parameters to vRO configuration items as they are used in more than one workflow, requiring a user to manually modify multiple workflows to set these variables.
 
 * **GAP:** Windows: No ability to bootstrap a Windows machine or install the Chef client.  
    * Consider supporting `knife-windows` plugin, and the ability to run `knife bootstrap windows winrm` and `knife bootstrap windows ssh` via the knife runner host.

    * Consider supporting the ability to use PowerShell to grab the Chef MSI and install.
 
 
### 2. Get details of Chef Client Already Installed on Nodes

 * "Get Client Details" and "Get Node Details" workflows work successfully, but just represents data on the Chef Server, not actual chef-clients installed on machines.  Need to know more specifics on exact use case.
 
 * **IMPROVEMENT:** possible vRO plugin for "knife status" equivalent?
 
## Register and Deregister Node with Chef

### 1. Register node with Chef Server

See above in the "Install Chef Client on Nodes" section for feedback re: the "Knife Bootstrap" workflow.

In reviewing the "Register Node" workflow:

 * **GAP:** only works with non-Windows machines given reliance on SSH
 
 * **BUG:** runlist and environment dropdowns only populate of a <ORG>-validator private key is saved in the vRO config.  If a user chooses not to save this (as it's only used for bootstraps, and shouldn't be required any more since Chef 12 support validator-less bootstrapping), the fields will no populate.
 
 * **BUG:** the runlist field is configured to only return roles.  Many users do not use roles and expect to be able to use individual recipes.  Consider changing `getRoles` to `getRunlistCandidates` as is used in the "Update Runlist on Node" workflow.
 
 * **IMPROVEMENT:** consider calling this "Bootstrap Linux Node" or generically calling it "Bootstrap Node" (and then using logic to determine how to handle Linux vs. Windows) considering this does more than just register the node with the Chef Server... the node is reached out to and chef is installed and chef-client is run.
 
### 2. Deregister node from Chef Server

 * "Delete Node" and "Delete Client" workflows confirmed working.
 
 * **IMPROVEMENT:** Consider creating an "Unregister Node" workflow that deletes the client and node together so the user doesn't have to remember to do both.
 
## Pushing Chef runlist of Recipes and/or Roles on Nodes

### 1. Read list of cookbooks, recipes, and roles.

 * "Get Cookbooks" and "Get Roles" workflows function as expected.
 
 * **GAP:** No way to retrieve recipes, or recipes-per-cookbook, currently other than the `GetRunlistCandidates` helper that is not exposed directly as a workflow.
 
### 2. Update list of nodes, cookbooks, and roles.

 * **NEED MORE INFO** - need additional details on this particular use case.
 
### 3. Assign runlist to nodes.

 * "Update Runlist on Node" workflow works as expected.
 
## Modifying attributes, environments, and data bags

### 1. Update individual node attributes

 * "Set Node Attributes" and "Set Single Node Attribute" workflows work as expected.

### 2. Get environment details and add nodes to environment

 * "Update Environment for Node" workflow works as expected.

### 3. Add, delete, and get data bag items

The following workflows were tested successfully:

 * Add Data Bag Item
 * Get Data Bag Item
 * Get Data Bags
 * Delete Data Bag Item
 * Delete Data Bag
 
 * **BUG**: "Add Data Bag Item Attribute" assumes the attribute is always a string.  I do not see a way to set the attribute value to an integer or another hash.

### 4. Support for encrypted data bag items

The following workflows were tested successfully:

 * Encrypt Data Bag Item
 * Decrypt Data Bag Item
 * Get Encrypted Data Bag Item Attribute

I'm unclear why the following workflows exist as a data bag items are generally encrypted completely or not:

 * Add Encrypted Data Bag Item Attribute
 * Encrypt Data Bag Item Attribute
 * Decrypt Data Bag Item Attribute

These were not tested as knife expects the data bag to be completely encrypted or not.

 * **BUG/SECURITY:** No way to create a data bag item and encrypt it in one step; the "Encrypt Data Bag Item" expects that the data bag item exists.  There will be a period of time where potentially sensitive data is on the chef server unencrypted.  If the encryption step fails, potentially sensitive data is left on the chef server.
 
 * **BUG/SECURITY:** the "encryptionKey" field in these workflows is plaintext and viewable when looking at historical workflow runs.  This could allow any authorized vRO user to ascertain a data bag's encryption key with little-to-no effort.  These fields should be reconfigured as SecureString fields.
 
 * **GAP:** no way to get a full encrypted data bag - only individual attributes

## Day 2 Operations

### 1. Teardown: deregister a node from master

 * Already addressed in "Deregister node from Chef Server" use case.
 
### 2. Update runlist, environment, or attributes for a specific node

 * Already addressed in "Modifying attributes, environments, and data bags" section.

## Ability to add, delete or modify the Chef Server that VMware Products are configured to interact with

**IMPROVEMENT:** make the Chef server list a vRO configuration element and present as a drop-down instead of requiring the user to type it in each time.

### 1. Add or delete Chef server

 * "HTTP-REST/Configuration/Add a REST host" and "Remove a REST host" operate as expected.
 
 * **GAP:** (documentation) The requirement to run the "Add a REST host" workflow did not appear to be documented, but it is a requirement.  Otherwise, workflows will fail with a getHostByName error or similar.  Only the "Set Private Key" workflow requirement was documented.

 * The chef server field in all workflows is a string.  Other than defining as a vRO REST Host target, no other configuration is necessary.  If you don't want to pass in a private key each time, the user should run "Set Private Key" workflow to save the key in the vRO configuration.

### 2. Use different Chef servers

 * As the chef server field is just a string, there is nothing preventing a user from using multiple Chef servers.
 

## Support Matrix

### 1. Chef hosted and on-premise instances should be supported

 * This is confirmed.  Plugin works with all flavors of Chef server.  Only Chef Server 12 was tested, but most (if not all) of the use cases above should be easily supported on Chef Server 11 as needed.

### 2. Windows and Linux operating systems should be supported

 * All interactions between vRO workflows and the Chef server operate independently of what the client OS is.  There were no issues seen with these types of workflows.

 * Linux client support is confirmed and all normal operations appear to function without issue (installation of chef-client, bootstrap, etc.).

 * **GAP:** Windows support for client operations is lacking with the exception of first-run operations.  There appears to be no way to bootstrap a Windows host.  As stated above, it is recommended that the `knife-windows` plugin be supported so that users can use `knife bootstrap windows winrm` to bootstrap.  A lot of work has gone into the 1.0.0 `knife-windows` release (soon-to-be released) and the bootstrap process works wonderfully.  Using `knife bootstrap` and ensuring parity between OS types is a Chef best practice.

