# 云服务器 ECS Linux 主机修改主机名

查看当前主机名状态：

```sh
# hostnamectl  或  hostnamectl status

   Static hostname: Fq
   Pretty hostname: Fq
         Icon name: computer-vm
           Chassis: vm
        Machine ID: f0f31005fb5a436d88e3c6cbf54e25aa
           Boot ID: 6c17de8372164600ab64cb91d4c9fabd
    Virtualization: kvm
  Operating System: CentOS Linux 7 (Core)
       CPE OS Name: cpe:/o:centos:centos:7
            Kernel: Linux 3.10.0-693.2.2.el7.x86_64
      Architecture: x86-64
```

@ 在CentOS中，有三种定义的主机名:静态的（static），瞬态的（transient），和灵活的（pretty）。“静态”主机名也称为内核主机名，是系统在启动时从/etc/hostname自动初始化的主机名。“瞬态”主机名是在系统运行时临时分配的主机名，例如，通过DHCP或mDNS服务器分配。静态主机名和瞬态主机名都遵从作为互联网域名同样的字符限制规则。而另一方面，“灵活”主机名则允许使用自由形式（包括特殊/空白字符）的主机名，以展示给终端用户（如Fq）。

只查看静态、瞬态或灵活主机名，分别使用“--static”，“--transient”或“--pretty”选项。

```sh
# hostnamectl --static
# hostnamectl --transient
# hostnamectl --pretty
```

要同时修改所有三个主机名：静态、瞬态和灵活主机名：

```sh
hostnamectl set-hostname Fq
```

查看状态发现问题，static hostname 为小写：

```sh
   Static hostname: fq
   Pretty hostname: Fq
         Icon name: computer-vm
           Chassis: vm
        Machine ID: f0f31005fb5a436d88e3c6cbf54e25aa
           Boot ID: 6c17de8372164600ab64cb91d4c9fabd
    Virtualization: kvm
  Operating System: CentOS Linux 7 (Core)
       CPE OS Name: cpe:/o:centos:centos:7
            Kernel: Linux 3.10.0-693.2.2.el7.x86_64
      Architecture: x86-64
```

就像上面展示的那样，在修改静态/瞬态主机名时，任何特殊字符或空白字符会被移除，而提供的参数中的任何大写字母会自动转化为小写。一旦修改了静态主机名，/etc/hostname 将被自动更新。然而，/etc/hosts 不会更新以保存所做的修改，所以你每次在修改主机名后一定要手动更新/etc/hosts，之后再重启CentOS 7。否则系统再启动时会很慢。

手动更新/etc/hosts：

```sh
vim /etc/hosts

127.0.0.1   Fq
#127.0.0.1  localhost localhost.localdomain localhost4 localhost4.localdomain
::1        localhost localhost.localdomain localhost6 localhost6.localdomai
```

end!