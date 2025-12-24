### Persistent storage on a rootless container.

- This demo bind-mounts directories on the host.
- File ownership on the host directories is important to allow the container to access the directory.
- If the container is started by the root user UIDs and GIDs on the host must match the UIDs and GIDs in the container.
- For a rootless container, the UID of the container user is the owner of the bind-mount directory.  Use "podman inspect" to find the user.
- Containers have there own UIDs from the hosts, so UIDs are mapped between the container namespace and host OS.
- "podman unshare" command can be used to run commands inside the container namespace.
- To get the UID mapping, use "podman unshare cat /proc/self/uid_map" to see the container root user maps to the host UID.

### Non-root User mappings.

- Additional work is required for non-root mappings.
- 1st find the user that runs the main application using "podman inspect imagename"
- Or use...."podman exec containername grep username /etc/passwd"
- 2nd Use..."podman unshare chown nn:nn directoryname".
- The directoryname must be in the user home directory because otherwise it wouldn't be part of the user namespace.
- To verify mapping use...."podman unshare cat /proc/self/uid_map".
- To verify the mapping on the host use...."ls -ld /directoryname"

### Instead of student user, I'm using mfau user.

```
[mfau@test ~]$ podman search mariadb | grep red
registry.redhat.io/rhscl/mariadb-101-rhel7                    MariaDB is a multi-user  multi-threaded SQL...
registry.redhat.io/rhel9/mariadb-105                          MariaDB 10.5 SQL database server
registry.redhat.io/rhel10/mariadb-1011                        MariaDB 10.11 SQL database server
registry.redhat.io/rhscl/mariadb-100-rhel7                    MariaDB is a multi-user  multi-threaded SQL...
registry.redhat.io/openshift3/mariadb-apb                     Ansible Playbook Bundle application definiti...
registry.redhat.io/rhel8/mariadb-103                          MariaDB 10.3 SQL database server
registry.redhat.io/rhel8/mariadb-105                          MariaDB 10.5 SQL database server
registry.redhat.io/rhscl/mariadb-105-rhel7                    MariaDB 10.5 SQL database server
registry.redhat.io/rhel9/mariadb-1011                         MariaDB 10.11 SQL database server
registry.redhat.io/rhel8/mariadb-1011                         MariaDB 10.11 SQL database server
registry.redhat.io/rhosp-dev-preview/mariadb-rhel9-operator   OpenStack mariadb operator
registry.redhat.io/openshift4/mariadb-apb                     Ansible Playbook Bundle application definiti...
registry.redhat.io/rhscl/mariadb-103-rhel7                    MariaDB 10.3 SQL database server
registry.redhat.io/rhscl/mariadb-102-rhel7                    MariaDB 10.2 SQL database server
registry.redhat.io/rhosp12/openstack-mariadb                  Red Hat OpenStack Platform 12.0 mariadb
registry.redhat.io/rhosp13/openstack-mariadb                  Red Hat OpenStack Platform 13.0 mariadb
registry.redhat.io/rhosp-dev-preview/mariadb-operator-bundle  OpenStack mariadb operator bundle
registry.redhat.io/rhosp-dev-preview/openstack-mariadb-rhel9  openstack-mariadb
registry.redhat.io/rhosp-beta/openstack-mariadb               openstack-mariadb
registry.redhat.io/rhosp15-rhel8/openstack-mariadb            openstack-mariadb
registry.redhat.io/rhosp-rhel8/openstack-mariadb              openstack-mariadb
registry.redhat.io/rhosp-rhel9/openstack-mariadb              openstack-mariadb-container
registry.redhat.io/rhosp14/openstack-mariadb                  Red Hat OpenStack Container image for openst...
registry.redhat.io/rhosp14-beta/openstack-mariadb             Red Hat OpenStack Beta Container image for o...
registry.redhat.io/rhoso-beta/openstack-mariadb-rhel9         pyxis_65e9b2ac4a4799a068545499
```
### Create a directory to bind-mount to the container.
```
[student@test ~]$ mkdir mydb
```
### Find the container uid.
```
[student@test ~]$ podman inspect registry.redhat.io/rhel9/mariadb-105 | grep -i uid
                    "created_by": "/bin/sh -c . /cachi2/cachi2.env &&     INSTALL_PKGS=\"policycoreutils rsync tar gettext hostname groff-base ${NAME}-server\" &&     yum install -y --setopt=tsflags=nodocs ${INSTALL_PKGS} &&     rpm -V ${INSTALL_PKGS} &&     /usr/libexec/mysqld -V | grep -qe \"${MYSQL_VERSION}\\.\" && echo \"Found VERSION ${MYSQL_VERSION}\" &&     yum -y clean all --enablerepo='*' &&     mkdir -p ${HOME}/data && chown -R mysql.0 ${HOME} &&     test \"$(id mysql)\" = \"uid=27(mysql) gid=27(mysql) groups=27(mysql)\"",
```

### Change ownership of the uid to match the container uid.
```
$ podman unshare chown 27:27 mydb
```

### This shows the container uid mapped to the student uid.
```
[student@test ~]$ podman unshare cat /proc/self/uid_map
         0       1000          1
         1     100000      65536
```

### host UID = host_start + (container_uid - inside_start)
### 100000 + (27 - 1) = 100026
```
[student@test ~]$ ls -ld mydb/
drwxr-xr-x. 2 100026 100026 6 Dec 24 08:58 mydb/
```

### Run the container.
```
[student@test ~]$ podman run -d --name mydb -e MYSQL_ROOT_PASSWORD=password -v /home/student/mydb:/var/lib/mysql:Z registry.redhat.io/rhel9/mariadb-105
7294366ac36e7f0f0e0cc66c83d0c57e6d63897dfd6d69518715c0c7cc0211b4

[student@test ~]$ podman ps
CONTAINER ID  IMAGE                                        COMMAND     CREATED         STATUS         PORTS       NAMES
7294366ac36e  registry.redhat.io/rhel9/mariadb-105:latest  run-mysqld  10 seconds ago  Up 10 seconds  3306/tcp    mydb
```

### The mydb directory has the correct SElinux context "container_file_t".
```
[student@test ~]$ ls -Z /home/student
            unconfined_u:object_r:user_home_t:s0 Containerfile              unconfined_u:object_r:user_home_t:s0 myfiles
            unconfined_u:object_r:user_home_t:s0 Desktop                    unconfined_u:object_r:user_home_t:s0 Pictures
            unconfined_u:object_r:user_home_t:s0 Documents                  unconfined_u:object_r:user_home_t:s0 Public
            unconfined_u:object_r:user_home_t:s0 Downloads                  unconfined_u:object_r:user_home_t:s0 rhcsa9
           unconfined_u:object_r:audio_home_t:s0 Music                      unconfined_u:object_r:user_home_t:s0 Templates
system_u:object_r:container_file_t:s0:c802,c1016 mydb                       unconfined_u:object_r:user_home_t:s0 Videos
```

### Show mariadb data files with the correct ownership.

```
[student@test ~]$ ll mydb
total 4
drwx------. 4 100026 100026 4096 Dec 24 10:01 data
srwxrwxrwx. 1 100026 100026    0 Dec 24 10:01 mysql.sock
```










