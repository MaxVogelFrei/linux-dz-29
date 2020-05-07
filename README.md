# NFS

## Задание
Vagrant стенд для NFS или SAMBA  
NFS или SAMBA на выбор:  
  
vagrant up должен поднимать 2 виртуалки: сервер и клиент  
на сервер должна быть расшарена директория  
на клиента она должна автоматически монтироваться при старте (fstab или autofs)  
в шаре должна быть папка upload с правами на запись  
- требования для NFS: NFSv3 по UDP, включенный firewall  

## Решение
```bash
vagrant up
```
разворачивает стенд из 2 ВМ server и client  
стенд настраивается плэйбуком [nfs.yml](nfs.yml)  
### Server
Создание папки для RO-шары и upload c правами на запись для всех (только на сервере)
```yaml
    - name: create shared folder
      file:
        path: "/var/nfs_share"
        state: directory
        mode: '0755'

    - name: create upload
      file:
        path: "/var/nfs_share/upload"
        state: directory
        mode: '0777'
      when: server
```
Разрешение подключения с ip клиента и включение nfs v3
```yaml
    - name: share conf 
      lineinfile:
        path: /etc/exports
        line: "/var/nfs_share 192.168.29.151(rw,sync,all_squash,no_subtree_check)"
      when: server

    - name: uncomment nfsv3
      replace:
        path: /etc/nfs.conf
        regexp: "{{ item.regexp }}"
        replace: "{{ item.replace }}"
      loop:
        - { regexp: "^#[nfsd]", replace: "[nfsd]"  }
        - { regexp: "^# vers3=y", replace: "vers3=y"  }
      when: server
```
Разрешение сервисов в firewalld
```yaml
    - name : firewalld config
      firewalld:
        zone: public
        service: "{{ item }}"
        permanent: yes
        immediate: yes
        state: enabled
      loop:
        - mountd
        - rpc-bind
        - nfs3
      when: server
```
### Client
Запись в fstab для автоматического подключения при загрузке
```yaml
    - name: mount the nfsshare in client side
      mount:
        fstype: nfs
        opts: nofail,proto=udp,vers=3
        dump: 0
        passno: 2
        src: "192.168.29.150:/var/nfs_share"
        path: "/var/nfs_share"
        state: mounted
      when: not server
```

## Проверка
Доступ на запись в /var/nfs_share отсутствует  
```bash
[root@centos7 linux-dz-29]# vagrant ssh client
Last login: Thu May  7 19:05:07 2020 from 10.0.2.2
[vagrant@client ~]$ cd /var/nfs_share/
[vagrant@client nfs_share]$ ll
total 0
drwxr-xrwx. 2 root root 6 May  7 19:04 upload
[vagrant@client nfs_share]$ touch 123
touch: cannot touch '123': Permission denied
```
Доступ на запись в /var/nfs_share/upload разрешен  
```bash
[vagrant@client nfs_share]$ cd upload
[vagrant@client upload]$ touch 123
[vagrant@client upload]$ ll
total 0
-rw-rw-r--. 1 nfsnobody nfsnobody 0 May  7 19:06 123
```

