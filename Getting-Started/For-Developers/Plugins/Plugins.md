## 插件

vlib实现了一种简单的插件DLL机制。VLIB客户端应用程序指定目录以搜索插件.DLL，并指定要应用的名称过滤器（如果需要）。VLIB需要非常早地加载插件。

加载后，插件DLL机制使用dlsym在新加载的插件中查找并验证vlib_plugin_registration数据结构。

有关插件的更多信息，请参阅[添加插件](https://github.com/penybai/vpp-docs/blob/master/Getting-Started/For-Developers/Adding-a-plugin/Adding-a-plugin.md)。