---
layout: post
title:  "Ceph Manager模块加载源码追溯"
date:   2020-05-15 13:00:00 +0800
categories: ceph src
---
Ceph 集群版本: v15.2.1 Octopus  
Ceph 安装方式: cephadm  

上文提到在 Ceph 生产环境遇到这样一条错误信息：

```sh
MGR_MODULE_ERROR: Module 'rbd_support' has failed: Not found or unloadable
```

我们再来追一下那条错误信息。

通过[官方文档][ceph-mgr modules]的信息，我们知道是 Ceph Manager 启动时加载 `rbd_support` 模块失败了，而且 `rbd_support` 模块属于always_on的，但是目前Ceph官方文档里并没有介绍这个 `rbd_support` 模块起到的作用。

查看Ceph源代码，这条错误来自 src/mgr/PyModuleRegistry.cc:

```c++
void PyModuleRegistry::get_health_checks(health_check_map_t *checks) {
  std::map<std::string, std::string> failed_modules;
  
  // report failed always_on modules as health errors
  for (const auto& name : mgr_map.get_always_on_modules()) {
    if (!active_modules->module_exists(name)) {
      if (failed_modules.find(name) == failed_modules.end() &&
          dependency_modules.find(name) == dependency_modules.end()) {
        failed_modules[name] = "Not found or unloadable";
      }
    }
  }
  
  if (!failed_modules.empty()) {
    std::ostringstream ss;
    if (failed_modules.size() == 1) {
      auto iter = failed_modules.begin();
      ss << "Module '" << iter->first << "' has failed: " << iter->second;
    }
    auto& d = checks->add("MGR_MODULE_ERROR", HEALTH_ERR, ss.str(),
                failed_modules.size());
    }
}
```

这段代码的意思跟错误消息提示的意思一样，就是 standby Manager在启动时加载 `rbd_support` 模块的时候失败了。

未完

[ceph-mgr modules]: https://docs.ceph.com/docs/master/mgr/administrator/