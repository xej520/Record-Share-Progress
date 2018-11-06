fabric8io/kubernetes-client

1、创建默认的k8s客户端

Config config = new ConfigBuilder().withMasterUrl("http://172.16.3.30:23333").build();
KubernetesClient client = new DefaultKubernetesClient(config);
2、获取目标资源的操作权，相应实现类的对象,常见如下。

client.namespaces() 
client.services()
client.pods()  
client.customResources()
client.storage()
client.network()
3、获取操作权之后，进行资源的CRUD。

创建：
Service service = client.services().inNamespace(namespace).create(service);
 
更新：
Namespace namespace = client.namespaces().withName(name).get();
 //update resources
client.namespaces().createOrReplace(namespace);
 
查询：
ServiceList services = client.services().inNamespace("default").list();
Service service = client.services().inNamespace("default").withName("myservice").get()
     
删除：
client.services().inNamespace("default").withName("myservice").delete();
注：以上是连缀调用，先修改资源，再进行资源的操作，还有一种内建的方式，如下，但是不方便维护。

client.services().inNamespace("default").withName("myservice").edit()
                     .editMetadata()
                       .addToLabels("another", "label")
                     .endMetadata()
                     .done();
4、补充：自定义资源的操作

//1、获取自定义资源模板（如果想操作模板，替换get方法即可）
String crdName = "ftpclusters.ftp.bonc.com";
CustomResourceDefinition crd = client.customResourceDefinitions()
   .withName(crdName).get();
 
//2、获取自定义资源操作权，也可以叫CRD client（指的是通过模板生成的实例的操作权）
MixedOperation<FtpCluster, FtpList, DoneableFtp, Resource<FtpCluster, DoneableFtp>> ftpClient = client.customResources(crd, FtpCluster.class, FtpList.class, DoneableFtp.class);
 
//3、同k8s内置资源一样，进行CRUD调用即可
CustomResourceList<FtpCluster> customResourceList = ftpCrdClient.list();
ftpCrdClient.create(ftpCluster);
ftpCrdClient.createOrReplace(ftpCluster);
 
//4、注意自定义资源的model,必须继承于基本的model，才能被client管理起来。
public class FtpCluster extends CustomResource
public class FtpList extends CustomResourceList<FtpCluster> {
public class DoneableFtp extends CustomResourceDoneable<FtpCluster> {
5、附录

客户端使用的是fabric8io开发的kubernetes-client项目。https://github.com/fabric8io/kubernetes-client

<!--kubernetes-client 中包括了 对应版本的kubernetes-model依赖 -->
<dependency>
    <groupId>io.fabric8</groupId>
    <artifactId>kubernetes-client</artifactId>
    <version>4.0.0</version><!-- 对应k8s的1.9版本-->
</dependency>

