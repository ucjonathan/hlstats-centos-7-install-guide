As root, first we need to make a directory to upload files into
```
mkdir ~/import
```

Enable SSHD
```
systemctl enable sshd
systemctl start sshd
```

From your windows machine using Cygwin or other log source.  Replace 198.168.171.138 with whatever the IP is of your HLStats server.
```
scp *.log root@192.168.171.138:~/import/
```

Go to your HLStats admin located at `http://SERVER_IP/hlstats/index.php?mode=admin` and Configure the game server.  
In this example we have a game server running on the LAN at 192.168.2.10
This command cats all the log files through to the hlstats script.  The server-ip and server-port parameters on the hlstats.pl script at the IP and port of your game server and NOT the hlstats server itself.
```
cd ~hlstats/daemon
cat ~root/import/*.log | perl ./hlstats.pl --stdin -m Normal --server-ip 192.168.2.10 --server-port 27500 -t
```
