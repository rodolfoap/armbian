# NFS Install

```
# apt -y install nfs-kernel-server rpcbind

# echo "portmap: 192.168.1." >> /etc/hosts.allow

# /etc/init.d/rpcbind restart
	[ ok ] Restarting rpcbind (via systemctl): rpcbind.service.

# echo "/nfs 192.168.1.0/255.255.255.0(rw,no_root_squash,subtree_check)" >> /etc/exports

# exportfs -a

# /etc/init.d/nfs-kernel-server reload
	[ ok ] Reloading nfs-kernel-server configuration (via systemctl): nfs-kernel-server.service.

# showmount -e
	Export list for mc1:
	/nfs 192.168.1.0/255.255.255.0
```
