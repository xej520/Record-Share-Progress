# <center>calicoctl get命令介绍</center>   
备注:
当前版本v2.6  

### calicoctl get --help  
```
Usage:
  calicoctl get ([--scope=<SCOPE>] [--node=<NODE>] [--orchestrator=<ORCH>]
                 [--workload=<WORKLOAD>] (<KIND> [<NAME>]) |
                --filename=<FILENAME>)
                [--output=<OUTPUT>] [--config=<CONFIG>]

Examples:
  # List all policy in default output format.
  calicoctl get policy

  # List a specific policy in YAML format
  calicoctl get -o yaml policy my-policy-1

Options:
  -h --help                    Show this screen.
  -f --filename=<FILENAME>     Filename to use to get the resource.  If set to
                               "-" loads from stdin.
  -o --output=<OUTPUT FORMAT>  Output format.  One of: yaml, json, ps, wide,
                               custom-columns=..., go-template=...,
                               go-template-file=...   [Default: ps]
  -n --node=<NODE>             The node (this may be the hostname of the
                               compute server if your installation does not
                               explicitly set the names of each Calico node).
     --orchestrator=<ORCH>     The orchestrator (valid for workload endpoints).
     --workload=<WORKLOAD>     The workload (valid for workload endpoints).
     --scope=<SCOPE>           The scope of the resource type.  One of global,
                               node.  This is only valid for BGP peers and is
                               used to indicate whether the peer is a global
                               peer or node-specific.
  -c --config=<CONFIG>         Path to the file containing connection
                               configuration in YAML or JSON format.
                               [default: /etc/calico/calicoctl.cfg]

Description:
  The get command is used to display a set of resources by filename or stdin,
  or by type and identifiers.  JSON and YAML formats are accepted for file and
  stdin format.

  Valid resource types are:

    * node
    * bgpPeer
    * hostEndpoint
    * workloadEndpoint
    * ipPool
    * policy
    * profile

  The resource type is case insensitive and may be pluralized.

  Attempting to get resources that do not exist will simply return no results.

  When getting resources by type, only a single type may be specified at a
  time.  The name and other identifiers (hostname, scope) are optional, and are
  wildcarded when omitted. Thus if you specify no identifiers at all (other
  than type), then all configured resources of the requested type will be
  returned.

  By default the results are output in a ps-style table output.  There are
  alternative ways to display the data using the --output option:

    ps                    Display the results in ps-style output.
    wide                  As per the ps option, but includes more headings.
    custom-columns        As per the ps option, but only display the columns
                          that are requested in the comma-separated list.
    golang-template       Display the results using the specified golang
                          template.  This can be used to filter results, for
                          example to return a specific value.
    golang-template-file  Display the results using the golang template that is
                          contained in the specified file.
    yaml                  Display the results in YAML output format.
    json                  Display the results in JSON output format.

  Note that the data output using YAML or JSON format is always valid to use as
  input to all of the resource management commands (create, apply, replace,
  delete, get).

  Please refer to the docs at https://docs.projectcalico.org for more details on
  the output formats, including example outputs, resource structure (required
  for the golang template definitions) and the valid column names (required for
  the custom-columns option).
```  
不同的资源类型，有不同的属性内容， 这里以profile资源类型，进行说明
主要介绍输出格式， calico提供了以下几种输出格式： 

1. ps(默认)  
calicoctl get profile cal_net1 -o ps  

2. wide  (会包含更多的heading信息)  


3. custom-columns (按照指定的列输出，多个列时，按照逗号分隔)  

4. golang-template  

5. yaml  

6. json  









