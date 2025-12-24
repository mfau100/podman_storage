### Providing persistent storage using a bind mount or hostpath.



[student@test ~]$ mkdir mydb

[student@test ~]$ podman inspect registry.redhat.io/rhel9/mariadb-105 | grep -i uid
                    "created_by": "/bin/sh -c . /cachi2/cachi2.env &&     INSTALL_PKGS=\"policycoreutils rsync tar gettext hostname groff-base ${NAME}-server\" &&     yum install -y --setopt=tsflags=nodocs ${INSTALL_PKGS} &&     rpm -V ${INSTALL_PKGS} &&     /usr/libexec/mysqld -V | grep -qe \"${MYSQL_VERSION}\\.\" && echo \"Found VERSION ${MYSQL_VERSION}\" &&     yum -y clean all --enablerepo='*' &&     mkdir -p ${HOME}/data && chown -R mysql.0 ${HOME} &&     test \"$(id mysql)\" = \"uid=27(mysql) gid=27(mysql) groups=27(mysql)\"",
[student@test ~]

$ podman unshare chown 27:27 mydb

[student@test ~]$ podman unshare cat /proc/self/uid_map
         0       1000          1
         1     100000      65536

[student@test ~]$ ls -ld mydb/
drwxr-xr-x. 2 100026 100026 6 Dec 24 08:58 mydb/
