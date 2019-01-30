
# kubelet multus-cni调用关系  
1. kubectl调用RunPod()开始启动container;   

2. setUpPod()调用网络配置插件(k8s的CNI);
3. 网络配置插件根据k8s配置(默认位于/etc/cni/net.d/下),调用Multus,进入Multus逻辑;
4. Multus首先调用主pulgin,然后调用其他的plugin创建相应的interface，plugin包括flannel，IPAM， SR-IOV CNI plugin等。













