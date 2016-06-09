## <center>基于名称的虚拟服务器</center>

Nginx首先要解决的的是怎么处理请求。我们来创建三个虚拟服务器，他们都监听80端口。

    server {
     listen      80;
     server_name example.org www.example.org;
     ...
    }

    server {
     listen      80;
     server_name example.net www.example.net;
     ...
    }

    server {
     listen      80;
     server_name example.com www.example.com;
     ...
    }
在这个配置文件中，请求头中的Host字段决定这个请求应该用那个服务器处理。如果这个值不匹配任何一个服务器名称，或者这个请求头不包含这个Host字段，Nginx将这个请求交给所以监听这个端口的默认服务器处理。在这个配置中，，默认的服务器是第一个————这是nginx的默认行为。在监听的路径中使用，default_server参数可以明确的指定那个服务器是默认的。

>server {  
>listen      80 **default_server**;  
>server_name example.net www.example.net;  
>...  
>}  

default_server参数从0.8.21版本开始使用的。更早的版本请用default参数代替。

注意，默认的服务器是所监听端口的一个属性，而不是服务器名称的属性。

## <center>怎么处理未定义服务器名称的请求</center>
如果一个请求没有“Host”头字段,应该是不允许的，对于这种情况，可以这样定义一个服务器：

>server {  
>    listen      80;  
>    server_name "";  
>    return      444;  
>}

在这里，这个服务器名被设置成一个空的字符串，它将匹配没有“Host”头的请求，一个nginx的特别的不标准的状态码444将被返回，然后关闭连接。
>从0.8.48版本开始，这已经是默认的配置了，所以这个server_name ""可以省略。更早的版本，这个机器的主机名应该被使用作为默认的服务器名。

## <center>基于名称和基于IP的虚拟服务器混合使用</center>
让我们看下一个更加复杂的配置，这里一些虚拟服务器监听不同的地址：

    server {
        listen      192.168.1.1:80;
        server_name example.org www.example.org;
        ...
    }

    server {
        listen      192.168.1.1:80;
        server_name example.net www.example.net;
        ...
    }

    server {
        listen      192.168.1.2:80;
        server_name example.com www.example.com;
        ...
    }

在这个配置中，nginx首先检查请求的IP地址和端口是否和server块指令监听的IP地址和端口一致。然后检测请求的“Host”头字段是否和server块的server_name一致。如果没有发现一样的服务器名称，这个请求将交给默认的服务器处理。例如，一个www.example.com的请求被192.168.1.1：80端口接收到，这个请求将交给192.168.1.1:80端口的默认服务器处理，也就是第一个服务器，因为这个端口上面没有定义www.example.com的服务器名称。

正如我们提到的，default_server是监听端口的一个属性，所以不同的端口可以定义不同的默认服务器：

    server {
        listen      192.168.1.1:80;
        server_name example.org www.example.org;
        ...
    }

    server {
        listen      192.168.1.1:80 default_server;
        server_name example.net www.example.net;
        ...
    }

    server {
        listen      192.168.1.2:80 default_server;
        server_name example.com www.example.com;
        ...
    }

## <center>一个简单的PHP站点配置</center>
现在让我们看下对于一个典型的，简单的PHP站点，Nginx是怎么选择location去处理一个请求的。

    server {
        listen      80;
        server_name example.org www.example.org;
        root        /data/www;

        location / {
            index   index.html index.php;
        }

        location ~* \.(gif|jpg|png)$ {
            expires 30d;
        }

        location ~ \.php$ {
            fastcgi_pass  localhost:9000;
            fastcgi_param SCRIPT_FILENAME
                          $document_root$fastcgi_script_name;
            include       fastcgi_params;
        }
    }

nginx首先搜索最具体的前缀location，不管location的顺序是什么。在这个配置中只有一个“/”的location前缀，因为它匹配任何的请求，它将请求最后的去处。然后nginx检查这个配置locations列表中的所有正则表达式。第一次匹配表达式后就会停止搜索，nginx将使用这个location。如果没有表达式匹配这个请求，nginx将使用最具体的，最先找到的前缀location。

注意所有的locations仅仅是一个请求不包含参数的URI的一部分。这是因为在查询字符串中参数表现可能是有很多种方式，例如：

>/index.php?user=john&page=1  
>/index.php?page=1&user=john

而且，在查询字符串里面可以有任何其他的东西：

>/index.php?page=1&something+else&user=john

现在让我们看下这个配置是怎么处理请求的：

* 请求 “/logo.gif” 首先匹配前缀location “/” ，然后匹配正则表达式 “\.(gif|jpg|png)$”，因此它将被后面的这个location处理。使用这个指令“root /data/www” 这个请求被映射成/data/www/logo.gif这个文件，这个文件将被发生给客户端。
* 请求“/index.php”也是首先匹配前缀location“/”,然后匹配正则表达式“\.(php)$”。因此，它将被后者处理，它被传递给一个监听localhost:9000的FastCGI服务器处理。fastcgi_param指令这个FastCGI参数SCRIPT_FILENAME为“/data/www/index.php”，FastCGI服务器将执行这个文件。$document_root这个变量等价于root指令的值，$fastcgi_script_name变量等价于请求的URI,例如“/index.php”。
* 请求“/about.html”只和前缀location“/”匹配，它将被这个location处理。使用“root /data/www”这个指令，这个请求将被映射为文件/data/www/about.html,这个文件将被发生给客户端。
* 处理请求“/”是最复杂的。它只和前缀location“/”匹配，因此它被这个location处理。这个index指令将根据它的参数和“root /data/www”指令检查index文件是否存在。如果/data/www/index.html文件不存在，但是/data/www/html.php存在，这个指令将在内部重定向到“/index.php”，nginx将像这个请求是被客户端发起的一样再次搜索locations。最终这个请求将被FastCGI服务器处理。
