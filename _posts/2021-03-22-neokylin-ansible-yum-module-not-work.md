---
layout: post
title: "中标麒麟系统ansible执行yum模块报错的问题分析"
description: ""
categories: [shell]
tags: [bash, ]
---

> 在使用中标麒麟V7Update6版本时，遇到了一个ansible执行报错的问题

* Kramdown table of contents
{:toc .toc}

## 问题现象

在中标麒麟（neokylin）系统中部署某服务，使用到了ansible，但是执行时发现有yum模块的task报错如下：
```
TASK [common : Install basic rpms] **************************************************************************
fatal: [node01]: FAILED! => {"changed": false, "msg": ["Could not detect which major revision of yum is in use, which is required to determine module backend.", "You can manually specify use_backend to tell the module whether to use the yum (yum3) or dnf (yum4) backend})"]}
```

报错为yum模块无法判断出系统的yum版本，提示需要手工执行yum的use_backend参数。同样的task在原生RHEL7系统执行没有遇到任何问题，看样子调入了中标麒麟的某个坑里。

## 问题分析

根据报错，很明确是因为ansible无法自动判断出系统使用的yum版本导致，我们知道当ansible中yum模块不指定use_backend参数时，将尝试自动判断，而ansible的setup模块可以获取对应的必要信息，
其中一个变量ansible_pkg_mgr及对应yum后端模块，接下来我们执行setup模块输出ansible_pkg_mgr变量来验证下我们的判断：

```
# ansible -i hosts node01 -m setup -a "filter=ansible_pkg_mgr"
node01 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false
}
```

果然没有办法获取到ansible_pkg_mgr变量，先看下系统版本信息:
```
~]# cat /etc/neokylin-release
NeoKylin Linux Advanced Server release V7Update6 (Chromium)
```

接下来根据报错提示信息找到ansible相关代码，在yum.py中，相关代码如下：
ansible/plugins/action/yum.py
```
        if module not in ["yum", "yum4", "dnf"]:
            facts = self._execute_module(module_name="setup", module_args=dict(filter="ansible_pkg_mgr", gather_subset="!all"), task_vars=task_vars)
            display.debug("Facts %s" % facts)
            module = facts.get("ansible_facts", {}).get("ansible_pkg_mgr", "auto")
            if (not self._task.delegate_to or self._task.delegate_facts) and module != 'auto':
                result['ansible_facts'] = {'pkg_mgr': module}

        if module != "auto":

            if module == "yum4":
                module = "dnf"

            if module not in self._shared_loader_obj.module_loader:
                result.update({'failed': True, 'msg': "Could not find a yum module backend for %s." % module})
            else:
                # run either the yum (yum3) or dnf (yum4) backend module
                new_module_args = self._task.args.copy()
                if 'use_backend' in new_module_args:
                    del new_module_args['use_backend']

                display.vvvv("Running %s as the backend for the yum action plugin" % module)
                result.update(self._execute_module(module_name=module, module_args=new_module_args, task_vars=task_vars, wrap_async=self._task.async_val))
                # Now fall through to cleanup
        else:
            result.update(
                {
                    'failed': True,
                    'msg': ("Could not detect which major revision of yum is in use, which is required to determine module backend.",
                            "You can manually specify use_backend to tell the module whether to use the yum (yum3) or dnf (yum4) backend})"),
                }
            )
            # Now fall through to cleanup

```
如代码所示，当执行yum未指定use_backend参数时，ansible会执行setup模块并根据ansible_pkg_mgr来自动判断yum的版本，获取不到则会报错，继续看下该参数的获取过程，找到pkg_mgr.py，关键代码如下：

ansible/module_utils/facts/system/pkg_mgr.py
```
    def collect(self, module=None, collected_facts=None):
        facts_dict = {}
        collected_facts = collected_facts or {}

        pkg_mgr_name = 'unknown'
        for pkg in PKG_MGRS:
            if os.path.exists(pkg['path']):
                pkg_mgr_name = pkg['name']

        # Handle distro family defaults when more than one package manager is
        # installed or available to the distro, the ansible_fact entry should be
        # the default package manager officially supported by the distro.
        if collected_facts['ansible_os_family'] == "RedHat":
            pkg_mgr_name = self._check_rh_versions(pkg_mgr_name, collected_facts)
... ...

 def _check_rh_versions(self, pkg_mgr_name, collected_facts):
        if collected_facts['ansible_distribution'] == 'Fedora':
            if os.path.exists('/run/ostree-booted'):
                return "atomic_container"
            try:
                if int(collected_facts['ansible_distribution_major_version']) < 23:
                    for yum in [pkg_mgr for pkg_mgr in PKG_MGRS if pkg_mgr['name'] == 'yum']:
                        if os.path.exists(yum['path']):
                            pkg_mgr_name = 'yum'
                            break
                else:
                    for dnf in [pkg_mgr for pkg_mgr in PKG_MGRS if pkg_mgr['name'] == 'dnf']:
                        if os.path.exists(dnf['path']):
                            pkg_mgr_name = 'dnf'
                            break
            except ValueError:
                # If there's some new magical Fedora version in the future,
                # just default to dnf
                pkg_mgr_name = 'dnf'
        elif collected_facts['ansible_distribution'] == 'Amazon':
            pkg_mgr_name = 'yum'
        else:
            # If it's not one of the above and it's Red Hat family of distros, assume
            # RHEL or a clone. For versions of RHEL < 8 that Ansible supports, the
            # vendor supported official package manager is 'yum' and in RHEL 8+
            # (as far as we know at the time of this writing) it is 'dnf'.
            # If anyone wants to force a non-official package manager then they
            # can define a provider to either the package or yum action plugins.
            if int(collected_facts['ansible_distribution_major_version']) < 8:
                pkg_mgr_name = 'yum'
            else:
                pkg_mgr_name = 'dnf'
        return pkg_mgr_name

```
以上代码可以看到当判断系统为红帽系，则会继续判断系统版本信息，当主版本号<8则使用yum，否则使用dnf，这里我们初步判断为麒麟对系统做了某些修改导致无法获取到主版本号。先执行setup获取发行版代号验证下是否执行了上述逻辑：

```
# ansible -i hosts node01 -m setup -a "filter=ansible_distribution"
node01 | SUCCESS => {
    "ansible_facts": {
        "ansible_distribution": "RedHat",
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false
}

# ansible -i hosts node01 -m setup -a "filter=ansible_distribution_major_version"
node01 | SUCCESS => {
    "ansible_facts": {
        "ansible_distribution_major_version": "V7Update6",
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false
}
```
通过setup模块的输出结果可看到系统是判断为redhat发行版，但是通过ansible_distribution_major_version获取到的发行版主版本号为V7Update6,
而和上面判断yum版本的代码关联起来看就会发现问题所在，int(collected_facts['ansible_distribution_major_version']) < 8 中，ansible_distribution_major_version 变量在其初始化的代码中对应为为distribution_version.split('.')[:2][0]的取值，而当系统中获取到的值是V7Update6时，该显然无法满足转换为int的要求。接下来看下V7Update6这个关键字的定义位置，根据经验系统版本相关信息应该在/etc/os-release中：

```
~]# cat /etc/os-release
NAME="NeoKylin Linux Advanced Server"
VERSION="V7Update6 (Chromium)"
ID="neokylin"
ID_LIKE="fedora"
VARIANT="Server"
VARIANT_ID="server"
VERSION_ID="V7Update6"
PRETTY_NAME="NeoKylin Linux Advanced Server V7Update6 (Chromium)"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:neokylin:enterprise_linux:V7Update6:GA:server"
HOME_URL="https://www.cs2c.com.cn/"
BUG_REPORT_URL="https://bugzilla.cs2c.com.cn/"

NEOKYLIN_BUGZILLA_PRODUCT="NeoKylin Linux Advanced Server 7"
NEOKYLIN_BUGZILLA_PRODUCT_VERSION=V7Update6
NEOKYLIN_SUPPORT_PRODUCT="NeoKylin Linux Advanced Server"
NEOKYLIN_SUPPORT_PRODUCT_VERSION="V7Update6"
```
这里果然可以看到VERSION_ID的值被定义为```V7Update6```，而系统原生发行版中该值是7，我们来看下os-release中对VERSION_ID参数的说明：

```
man os-release
... ...

       VERSION_ID=
           A lower-case string (mostly numeric, no spaces or other characters outside of 0-9, a-z, ".",
           "_" and "-") identifying the operating system version, excluding any OS name information or
           release code name, and suitable for processing by scripts or usage in generated filenames. This
           field is optional. Example: "VERSION_ID=17" or "VERSION_ID=11.04".
... ...
```

根据man文档中的描述，VERSION_ID取值范围为全小写，通常为数值型，不应有空格或其他特殊字符，可包含的字符为0-9a-z._-,那么这里可以看到两个问题，
第一个问题是kylin的VERSION_ID不符合此描述，包含了大写字符，第二个问题是VERSION_ID可以包含a-z字母，但是通常是数值如17,11.04等。
但由于常见发行版都将此处处理为数值型，就导致ansible按照此约定俗成固化了其获取系统版本的方法，并试图将一个字符串转换为int，不能满足当VERSION_ID包含了字母的情况。

## 验证结论

通过以上判断看到VERSION_ID是导致该问题现象的关键，那么我们可以尝试修改一下该参数值，再执行setup看看是否可以正常工作：
```
# grep VERSION_ID /etc/os-release
VERSION_ID="7"
```

这里我把VERSION_ID修改成了数字7，再执行setup观察ansible_pkg_mgr变量是否能获取到：
```
# ansible -i hosts node01 -m setup -a "filter=ansible_pkg_mgr"
node01 | SUCCESS => {
    "ansible_facts": {
        "ansible_pkg_mgr": "yum",
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false
}
```
可以看到，修改os-release中VERSION_ID为纯数值后，setup就可以正常判断到系统版本，进而可以获取到正确的yum版本了。
通过以上可以看到操作系统中即便是一些不起眼的细枝末节，处理不当也可能引发"连锁反应"。
