
相似问题如下:  
https://github.com/kubernetes/kubernetes/issues/57280  
https://www.cnblogs.com/yuxiaoba/p/10104815.html  


1. Multus: Err in tearing down failed plugins: Multus: error in invoke Delegate add - "flannel": failed to allocate for range 0: no IP addresses available in range set: 10.244.2.1-10.244.2.254   
原因如下：  
![](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/D83BE58DBCCE4325BBB266837C08F99F/23069)  
删除多余的ip地址   
![rm -rf 10*](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/18C75A1C88114BCB80AF69120FAD4C64/23073)  
查看pod的当前状态   
![查看pod](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/BB60D161A4A04645B8162B70EFE2E6BA/23075)  
再次查看/var/lib/cni/networks目录   
![](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/BF1379344A9F4B5F89EBD47F6D8F1555/23071)   






























