使用下列命令安装

sudo apt-get install vsftpd
安装完后，ftp的配置文件在

/etc/vsftpd.conf
可以使用下列命令来打开，关闭，重启ftp服务

sudo /etc/init.d/vsftpd start
sudo /etc/init.d/vsftpd stop
sudo /etc/init.d/vsftpd restart

使用下列命令，可以看到系统中多了ftp用户组和ftp用户

cat /etc/group
cat /etc/passwd

ftp服务器的目录位置在 /srv/ftp， 这也是匿名用户访问时的根目录。
可以使用下列命令来间接更改目录
cd /srv
sudo rm -d ftp
cd ~/
mkdir ftp
sudo ln -s ftp /srv/ftp

# 配置vsftpd.conf

编辑/etc/vsftpd.conf文件:
