# Qcloud Ansible Inventory

Qcloude's Ansible Dynamic Inventory script

[Ansible](https://docs.ansible.com/) 是自动化配置工具，Dynamic Inventory 允许使用脚本获到需要配置的机器列表和信息。

可以参考hosts.yml的示例ansible playbook如何为所有cvm生成hosts文件来方便访问，如果换成调用DNSPod API就能实现自动更新DNS记录。


# 开始

## 安装

可以直接clone本项目作为ansible playbook的根目录，或者把inventory目录复制到您ansible playbook的根目录下，并使用inventory目录作为inventory host file

> 设置host file可以使用命令参数 `-i` 或者在 ansible.cfg 中配置，参考本项目中的 ansible.cfg

如果您还有其它的 inventory 条目，也放到 inventory 目录下，参考 ansible multiple inventory sources [相关文档](http://docs.ansible.com/intro_dynamic_inventory.html#using-multiple-inventory-sources)

脚本qcloud.py使用python 2，在默认python 版本为3的环境下使用自行修改qcloud.py的第一行。
该脚本依赖腾讯云的命令行工具，可以使用pip安装
```
sudo pip install tccli
```
## 配置

首先需要配置好 `tccli`
```
tccli configure
```

复制示例配置文件并进行修改。配置文件必须命名为 `qcloud.ini` 并且和 `qcloud.py` 在同一目录下。
```
cp inventory/qcloud.example.ini inventory/qcloud.ini
```
然后配置SSH连接信息，cvm对应云主机，配置中支持Python的 `%` 替换，比如cvm中的 `%(InstanceName)s` 会替换成云主机的名称。括号中可以是任何 `tccli` 返回结果中的字段，另外为了方便使用 IP，还有以下额外字段可以使用
-	`PublicIp` BGP 机房出口 IP
-	`InnerIp` 内网 IP

配置文件还支持对某个 cvm 进行单独配置，只要新建新的小节，以资源类型和名称作为小节的名字，比如 cvm ops 会优先使用 `cvm.ops` 小节中的配置，见下面例子。命名规则见本文档后面的内容。
```
[cvm.ops]
host = ops.example.com
port = 2222
user = ops
```
## 测试
直接运行脚本 `qcloud.py`，没有错误应该会打印出符合ansible dynamic inventory要求的JSON，然后可以运行Ansible列表所有机器
```
ansible all -i qcloud.py --list
```
如果一切正常可以测试下 demo playbook hosts.yml
```
ansible-playbook hosts.yml
```
该playbook会在当前目录生成hosts，如果覆盖/etc/hosts可以使用里面配置的主机名比如 `ops.cvm` 来访问cvm主机了。而如果云主机已经配置能使用sudo进行ansible操作，那么这些主机名也更新到所有的云主机上了。

## 缓存
示例配置中默认开启了缓存，如果改变了 qcloud 的设置想要立即更新主机信息，可以手动执行下面命令刷新缓存
```
python inventory/qcloud.py --refresh-cache
```
还可以在inventory/qcloud.ini中关闭缓存功能

```
cache_disable = True
```

# 命名规则
## 主机名

即相应资源在Ansible中使用的host名称。cvm使用InstanceName字段, 也就是管理后台中可以修改的名称。
## 主机组

首先所有的资源都按照类型进行了分组，cvm主机都属于cvm这个组。

~~另外cvm还支持自定义分组，规则是使用『描述』(对应 API 返回结果中的Description)。业务组名称使用英文逗号分隔之后并加上`des_` 前缀即为该主机要加入的Ansible主机组。比如主机ops的业务组名称是 `dev,public` 那么在Ansible中会包含在组`des_dev`和`des_public`中。Qcloud本身的Tag也会转成对应的分组，规则是`KEY_VALUE`。~~
> 腾讯云无描述字段，可以利用主机名以特定分割符然后添加组信息或者利用标签字段实现分组功能，需要自行研究解决

## 主机变量名
所有API返回结果以及上面提到的额外IP字段都会嵌套在主机变量qcloud下，比如在jinja2模板中引用出口IP
``` yaml
{{ qcloud.PublicIpAddresses }}
```

# 参考资料
- [aliyun-ansible-inventory](https://github.com/doitian/aliyun-ansible-inventory)
- [Ansible](http://www.ansible.com)
- [dynamic inventory](http://docs.ansible.com/intro_dynamic_inventory.html)
- [腾讯云](https://cloud.tencent.com/)
- [腾讯云地域列表](https://cloud.tencent.com/document/api/213/15692#.E5.9C.B0.E5.9F.9F.E5.88.97.E8.A1.A8)
