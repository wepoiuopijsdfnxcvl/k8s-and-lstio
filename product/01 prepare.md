# 离线安装包准备


```

 #在本地能上网的虚拟机上安装 

 yum install -y yum-utils
 yum install -y createrepo
 yum install -y yum-plugin-downloadonly
 
 [root@localhost network]#  yumdownloader
 Loaded plugins: fastestmirror
 Usage: "yumdownloader [options] package1 [package2] 


 #在离线机上分别操作
 
 mkdir /home/soft
 #在离线机上分别操作
 mkdir /home/rpm/{dev,base,network,yum,docker,gcc,analysis} -p
 
 cd /home/rpm
 rpm -Uvh  base/* analysis/* gcc/* dev/* network/* yum/* docker/* --nodeps --force
 

```


# 工具命令

```

 #搜索并卸载
 rpm -qa | grep wget
 yum remove wget
 
 #本地安装
 yum localinstall *.rpm -y
 rpm -Uvh *.rpm --nodeps --force  #可以强制删除现装的包，避免冲突
 
 #下载组
 yumdownloader "@Development Tools" --resolve --destdir /data/yum-pkgs/dev-tools/ --下载软件组
 yumdownloader --resolve --destdir=/root/mypackages/ httpd  --单个包下载
 
 
```
