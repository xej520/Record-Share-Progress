写在开篇，是为了提醒自己， 
1. 要善始善终   
2. 莫着急，做好每一件小事    

参考文献：  
https://blog.csdn.net/dkfajsldfsdfsd/article/details/79525187  
http://www.qingpingshan.com/m/view.php?aid=389184   

学习这个的目的，是为了了解下面的内容:  
https://github.com/containernetworking/plugins   

# 一、 mavclan要知道的小知识点
- `macvlan并不创建网络，只创建虚拟网卡，`
- macvlan会`共享物理网卡`所链接的`外部网络`，实现的效果跟桥接模式是一样的。   
1. `macvlan 既不创建网络，主要有什么特性？或者说，macvlan的使用场景？   `
- macvlan主要是用来解决效率问题的。  
- 也就是说macvlan是效率贵高的跨主机网络虚拟化解决方案之一。  
- 适合在对网络性能要求极高的场景下。   
2. `网络虚拟化的目的？`   
就是在多租户场景下，在统一的底层网络之上，单独为`每个租户`虚拟出自己的网络从而达到`隔离`的目的。  

3. `macvlan属于什么解决方案呢？或者说，macvlan到底是干什么的，？ 或者说，有什么用？`   
- macvlan是网卡虚拟化方案  
- macvlan将一张物理网卡设置多个mac地址，就是一变多，一对多；类似于鸣人的影分身之术，  `注意： 需要物理网卡，打开混杂模式`   
![如何设置网卡为混杂模式](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/8C68A76FBDF64411BE7E3360BC68C282/23134)   
- 既然有了多个mac地址，就可以设置多个IP地址了，从而实现，一块物理网卡链接到交换机，变成多个虚拟网卡链接到交换机。   

4. macvlan是linux kernel提供的一种network driver类型，  `如何查看当前内核是否加载了该driver呢？`   

- lsmod | grep macvlan （查看是否加载了）
- modprobe macvlan    (手动加载macvlan驱动到内核)
- /drivers/net/macvlan.c  (源码地址)   

5. `macvlan的工作模式？ `
- Bridge：属于同一个parent接口的macvlan接口之间挂到同一个bridge上，可以二层互通（经过测试，发现这些macvlan接口都无法与parent 接口互通）。   
![bridge mode](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/E24C5BC89EA54EE58CB3F2CBBEE5C658/23136)   
- VPEA：所有接口的流量都需要到外部switch才能够到达其他接口。  
![vpea mode](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/FACE82BD610B4DEF964E2997A4C2F462/23138) 
- Private：接口只接受发送给自己MAC地址的报文。   
![private mode](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/73BDF262F53C4E6C926F9856BEE8B41C/23140)    



# 二、 测试
## 1. 基本练习测试，创建一个macvlan， 并分配一个ip(也就是链接到某一网络)   
![创建虚拟网卡ens33.01](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/BA10538AA158423C833B2255EA2C4F15/23142)   
![激活虚拟网卡ens33.01](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/AB51A0C806314066BCA198D3BE084D33/23144)   
![分配IP地址](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/D8960BA5E8674C6D9586CE034472204A/23148)   
![删除虚拟网卡ens33.01 ip link delete 网卡名字](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/7155E5C91E9842A49EEE4E2B3DF8E357/23150)  

## `2. 测试在不同命名空间下，两个macvlan类型虚拟网卡，是否可以直接通信   `
1. 创建两个虚拟网卡ens33.01, ens33.02  
```
ip link add link ens33 name ens33.01 type macvlan mode bridge   
ip link add link ens33 name ens33.02 type macvlan mode bridge   
```
![创建虚拟网卡](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/DBE7FD69E9A14069BE5B6307E6E6ABC7/23152)  

2. 创建两个命名空间  
```   
ip netns add ns1  
ip netns add ns2  
```
![创建命名空间](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/ADBBD89DDD88481596F55E4B2D91C946/23154)  

3. 将两个虚拟网卡ens33.01,ens33.02分别分配到ns1,ns2命名空间里  
```
ip link set ens33.01 netns ns1    
ip link set ens33.02 netns ns2   
```
![给虚拟网卡分配命名空间](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/9E15E1DFFE004A2FAD7618261342D44E/23156)  
![在命名空间中，尝试执行命令](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/BC4DF96041F14D6CA81D0EF1BDAEA991/23158)

4. 使用dhclient给虚拟网卡ens33.01, ens33.02分配IP   
```
ip net exec ns1 dhclient ens33.01   
ip net exec ns2 dhclient ens33.02   lL
```
![给虚拟网卡分配IP](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/5C0C69D69F2C424999EDC9BEBC54F385/23160)  
![查看分配的IP](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/AC6EE23F58464897979CC7CAC75B3C24/23162)   
5. 测试是否可以ping同宿主机    
![是否可以ping通本宿主机](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/1F3589AFE07E47F19BADD2CC03C729F6/23164)   
6. 测试ens33.01 与  ens33.02 之间是否可以互相ping通    
![是否可以ping通ns2命名空间下的ens33.02虚拟网卡？](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/DA9F866C8B934A79898282A922BC2143/23166)   

## `3. 测试同一命名空间下的虚拟网卡，是否可以ping通么？  `   
![创建虚拟网卡，分配IP](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/D86008B080354A27901DF5F4BE162E12/23168)  
![测试网络是否通](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/1B19ACE356464416A333781422344A4E/23170)  

感觉很怪异，不知道为啥，同一个命名空间下的虚拟网卡，居然不通。










