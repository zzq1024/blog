# 第一节：Nginx配置

### nginx.conf

* rewrite重定向url（可以美化URL）
* location优先级：(location =) > (location 完整路径) > (location ^~ 路径) > (location ~,~* 正则顺序，如果有两个相同正则，会直接按第一个正则为准，不会被下边的覆盖) > (location 部分起始路径) > (/)
* 如果直接访问ip:port（对应多个HOST），会访问第一个匹配的（引入文件顺序，文件内server配置顺序）；
  如果直接访问host（对应多个port）,会访问第一个匹配的（引入文件顺序，文件内server配置顺序）