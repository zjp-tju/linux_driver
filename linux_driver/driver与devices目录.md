driver : /sys/bus/platform/drivers
devices:/sys/bus/platform/devices   其中的设备与/proc/device-tree设备树的节点相连

![图 0](../images/d451767a5040e850ee049685f17444c799c1efcc0912acb8796bec786e04a93e.png)  
![图 1](../images/bece34dbb6faf516893cd51ffedce26ec38b8cd9f31d55acb1f93783550181f7.png)  

class目录：sys/class  
/sys/class 是由kernel在运行时导出的，目的是通过文件系统暴露出硬件的层级关系。
建立的都是相关文件的软链接
