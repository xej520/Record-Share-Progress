# <center>calicoctl create命令介绍</center>   
备注:
当前版本v2.6  
## calicoctl create 命令帮助文档  
calicoctl create --help 
```
Set the Calico datastore access information in the environment variables or
or supply details in a config file.

Usage:
  calicoctl create --filename=<FILENAME> [--skip-exists] [--config=<CONFIG>]

Examples:
  # Create a policy using the data in policy.yaml.
  calicoctl create -f ./policy.yaml

  # Create a policy based on the JSON passed into stdin.
  cat policy.json | calicoctl create -f -

Options:
  -h --help                 Show this screen.
  -f --filename=<FILENAME>  Filename to use to create the resource.  If set to
                            "-" loads from stdin.
     --skip-exists          Skip over and treat as successful any attempts to
                            create an entry that already exists.
  -c --config=<CONFIG>      Path to the file containing connection
                            configuration in YAML or JSON format.
                            [default: /etc/calico/calicoctl.cfg]

Description:
  The create command is used to create a set of resources by filename or stdin.
  JSON and YAML formats are accepted.

  Valid resource types are:

    * node
    * bgpPeer
    * hostEndpoint
    * workloadEndpoint
    * ipPool
    * policy
    * profile

  Attempting to create a resource that already exists is treated as a
  terminating error unless the --skip-exists flag is set.  If this flag is set,
  resources that already exist are skipped.

  The output of the command indicates how many resources were successfully
  created, and the error reason if an error occurred.  If the --skip-exists
  flag is set then skipped resources are included in the success count.

  The resources are created in the order they are specified.  In the event of a
  failure creating a specific resource it is possible to work out which
  resource failed based on the number of resources successfully created.
```
