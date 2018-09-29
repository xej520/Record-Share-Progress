# <center>calico不同场景的解决方案</center>
1. calico集群内部可以互相通信，如果一个外部节点想访问calico网络，如何实现呢？  
在profile的spec.ingress中添加一个网络配置即可，例子如下：  
    ```
    {
            "action": "allow",
            "source": {
                "nets": [
                "192.185.115.0/24"
                ]
            },
            "destination": {}
            }
    }
    ```
2. 作为k8s的插件时，能不能指定pod的IP地址呢？   
 https://docs.projectcalico.org/v2.6/reference/cni-plugin/configuration  
可以的  
    #### Requesting a Specific IP address

    You can also request a specific IP address through [Kubernetes annotations](https://kubernetes.io/docs/user-guide/annotations/) with Calico IPAM.
    There are two annotations to request a specific IP address:

    - `cni.projectcalico.org/ipAddrs`: A list of IPv4 and/or IPv6 addresses to assign to the Pod. The requested IP addresses will be assigned from Calico IPAM and must exist within a configured IP pool.

    Example:

    ```yaml
    annotations:
            "cni.projectcalico.org/ipAddrs": "[\"192.168.0.1\"]"
    ```

    - `cni.projectcalico.org/ipAddrsNoIpam`: A list of IPv4 and/or IPv6 addresses to assign to the Pod, bypassing IPAM. Any IP conflicts and routing have to be taken care of manually or by some other system.
    Calico will only distribute routes to a Pod if its IP address falls within a Calico IP pool. If you assign an IP address that is not in a Calico IP pool, you must ensure that routing to that IP address is taken care of through another mechanism.

    Example:

    ```yaml
    annotations:
            "cni.projectcalico.org/ipAddrsNoIpam": "[\"10.0.0.1\"]"
    ```

    > **Note**:
    > - The `ipAddrs` and `ipAddrsNoIpam` annotations can't be used together.
    > - You can only specify one IPv4/IPv6 or one IPv4 and one IPv6 address with these annotations.
    > - When `ipAddrs` or `ipAddrsNoIpam` is used with `ipv4pools` or `ipv6pools`, `ipAddrs` / `ipAddrsNoIpam` take priority.
    {: .alert .alert-info}

3. k8s 和 calico 都有网络策略， 如何只使用k8s的网络策略呢?  

    If you wish to use the Kubernetes `NetworkPolicy` resource then you must set a policy type in the network config.   
    There is a single supported policy type, `k8s`.  
    When set, you must also run calico/kube-controllers with the policy, profile, and workloadendpoint controllers enabled.  

    ```json
    {
        "name": "any_name",
        "cniVersion": "0.1.0",
        "type": "calico",
        "policy": {
        "type": "k8s"
        },
        "kubernetes": {
            "kubeconfig": "/path/to/kubeconfig"
        },
        "ipam": {
            "type": "calico-ipam"
        }
    }
    ```

    When using `type: k8s`, the Calico CNI plugin requires read-only Kubernetes API access to the `Pods` resource in all namespaces.

    Previous versions of the plugin (`v1.3.1` and earlier) supported an alternative type called [`k8s-annotations`](https://github.com/projectcalico/calicoctl/blob/v0.20.0/docs/cni/kubernetes/AnnotationPolicy.md) This uses annotations on pods to specify network policy but is no longer supported.  

4. calico 作为k8s的插件，需要设置访问k8s的配置文件  

    When using the Calico CNI plugin with Kubernetes, the plugin must be able to access the Kubernetes API server in order to find the labels assigned to the Kubernetes pods.  
    The recommended way to configure access is through a `kubeconfig` file specified in the `kubernetes` section of the network config. e.g.

    ```json
    {
        "name": "any_name",
        "cniVersion": "0.1.0",
        "type": "calico",
        "kubernetes": {
            "kubeconfig": "/path/to/kubeconfig"
        },
        "ipam": {
            "type": "calico-ipam"
        }
    }
    ```

    As a convenience, the API location can also be configured directly, e.g.

    ```json
    {
        "name": "any_name",
        "cniVersion": "0.1.0",
        "type": "calico",
        "kubernetes": {
            "k8s_api_root": "http://127.0.0.1:8080"
        },
        "ipam": {
            "type": "calico-ipam"
        }
    }
    ```

