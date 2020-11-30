### content_by_lua

```nginx
location /lua {
    default_type text/html;
    #content_by_lua是luagit集成的指令
    content_by_lua '
        #这里表示通过nginx直接输出页面
        ngx.say("<h1>hello world</h1>");
    	#输出浏览器携带参数a
    	ngx.say(ngx.var.arg_a);
    ';
}
```



### content_by_lua_file

```nginx
location /lua {
    default_type text/html;
    #定位到文件/usr/local/openresty/nginx/lua/test.lua
    content_by_lua_file lua/test.lua;
}
```

