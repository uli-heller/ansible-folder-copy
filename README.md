Ansible: Verzeichnisbaum
==========================

Manchmal muß ich einen ganzen Verzeichnisbaum auf einen Rechner übertragen,
der mittels Ansible administriert wird. Leider gibt es keine Lösung
hierfür, die "supertoll" ist.

<!-- more -->

Aufgabenstellung
----------------

Ich habe eine Ansible-Rolle (role), die diese Dateien auf dem
Zielrechner ausrollen soll:

* files/rollout
    * apache2
        * conf.d/template.conf
        * vhosts.d/template.vhost
    * data/with/deep/paths
        * 001.txt
        * 002.txt
        * ...
        * 100.txt

Auf dem Zielrechner soll das so aussehen:

* /tmp/copied-by-ansible
    * apache2
        * conf.d/template.conf
        * vhosts.d/template.vhost
    * data/with/deep/paths
        * 001.txt
        * 002.txt
        * ...
        * 100.txt

Zielsetzung
-----------

* Einfache Anwendung
* Schnelle Ausführung
* Aussagekräftige Ausgabe bei `--check --diff`
* Keine irritierenden Zusatzausgaben

Lösungsalternativen
-------------------

* Dateien einzeln kopieren mit `copy`
* Dateien komplett kopieren mit `copy`
* Dateien komplett kopieren mit `synchronize`
* Dateien komplett kopieren mit`copy` und `filetree`
* Dateien komplett kopieren mit`file` und `filetree`

Dateien einzeln kopieren mit `copy`
-----------------------------------

### Erster Aufruf mit --check --diff

```
uli@ulinuc:~/tmp/ansible-test$ ansible-playbook copy-one-by-one.yml --check --diff
PLAY [all] *******************************************************************************************************************************************************************************************************************************

TASK [Gathering Facts] *******************************************************************************************************************************************************************************************************************
[WARNING]: Platform linux on host ansible-test is using the discovered Python interpreter at /usr/bin/python, but future installation of another Python interpreter could change this. See
https://docs.ansible.com/ansible/2.9/reference_appendices/interpreter_discovery.html for more information.
ok: [ansible-test]

TASK [copy-one-by-one-role : copy template.vhost] ********************************************************************************************************************************************************************************************
--- before
+++ after: /home/uli/tmp/ansible-test/roles/copy-one-by-one-role/files/rollout/apache2/vhosts.d/template.vhost
@@ -0,0 +1 @@
+Erste Version - template.vhost

changed: [ansible-test]

TASK [copy-one-by-one-role : copy template.conf] *********************************************************************************************************************************************************************************************
--- before
+++ after: /home/uli/tmp/ansible-test/roles/copy-one-by-one-role/files/rollout/apache2/conf.d/template.conf
@@ -0,0 +1 @@
+Erste Version - template.conf

changed: [ansible-test]

TASK [copy-one-by-one-role : copy 001.txt] ***************************************************************************************************************************************************************************************************
--- before
+++ after: /home/uli/tmp/ansible-test/roles/copy-one-by-one-role/files/rollout/data/with/deep/paths/001.txt
@@ -0,0 +1 @@
+Datei 001 - 1

changed: [ansible-test]

TASK [copy-one-by-one-role : copy 002.txt] ***************************************************************************************************************************************************************************************************
--- before
+++ after: /home/uli/tmp/ansible-test/roles/copy-one-by-one-role/files/rollout/data/with/deep/paths/002.txt
@@ -0,0 +1 @@
+Datei 002 - 2

...
```

Die Ausführung dauert sehr lange, leider nicht direkt abgestoppt.
Man sieht "schöne DIFFs".

Stoppuhr - `time ansible-playbook copy-one-by-one.yml --check --diff`
* real	6m16.366s, user	2m11.332s, sys	1m18.876s
* real	6m26.726s, user	2m10.996s, sys	1m24.316s
* real	4m39.737s, user	1m19.788s, sys	0m51.504s
* real	4m34.403s, user	1m15.944s, sys	0m49.900s
* real	4m34.527s, user	1m16.412s, sys	0m49.548s
* real	4m35.456s, user	1m17.816s, sys	0m51.064s

### Aufruf ohne --check --diff mit Komplettabgleich

Aufruf bei leerem Zielrechner, es müssen alle Dateien abgeglichen werden:

```
uli@ulinuc:~/tmp/ansible-test$ ansible-playbook copy-one-by-one.yml
...
PLAY RECAP *******************************************************************************************************
ansible-test               : ok=106  changed=103  unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

Die Ausführung dauert sehr lange. Es werden alle Dateien hochgeladen.

Stoppuhr:
* real	4m56.506s, user	1m21.632s, sys	0m50.104s

### Aufruf ohne --check --diff ohne Abgleich

Aufruf bei bereits aktualisiertem Zielrechner, es müssen keinerlei Dateien abgeglichen werden:

```
uli@ulinuc:~/tmp/ansible-test$ ansible-playbook copy-one-by-one.yml
...
PLAY RECAP *******************************************************************************************************
ansible-test               : ok=106  changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

Die Ausführung dauert sehr lange. Es werden keinerlei Dateien hochgeladen.

Stoppuhr:
* real	5m4.665s, user	1m47.408s, sys	1m10.628s
* real	6m36.950s, user	2m11.236s, sys	1m19.168s
* real	4m27.372s, user	1m14.708s, sys	0m44.964s

### Abgleich löschen

```
uli@ulinuc:~/tmp/ansible-test$ ssh ansible-test rm -rf /tmp/copied-by-ansible
```

Dateien komplett kopieren mit `copy`
------------------------------------

### Erster Aufruf mit --check --diff

```
uli@ulinuc:~/tmp/ansible-test$ ansible-playbook copy-folder.yml --check --diff
PLAY [all] *******************************************************************************************************

TASK [Gathering Facts] *******************************************************************************************
ok: [ansible-test]

TASK [copy-folder-role : Copy complete folder] *******************************************************************
changed: [ansible-test]

PLAY RECAP *******************************************************************************************************
ansible-test               : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

Die Ausführung dauert relativ lange, die Ausgabe bleibt "hängen".
Man sieht keine DIFFs.

Stoppuhr - `time ansible-playbook copy-folder.yml --check --diff`
* real	3m40.986s, user	0m56.912s, sys	0m34.692s
* real	3m30.098s, user	0m57.872s, sys	0m35.396s
* real	3m12.622s, user	0m48.276s, sys	0m28.536s

### Aufruf ohne --check --diff mit Komplettabgleich

Aufruf bei leerem Zielrechner, es müssen alle Dateien abgeglichen werden:

```
uli@ulinuc:~/tmp/ansible-test$ ansible-playbook copy-folder.yml
...
PLAY RECAP *******************************************************************************************************
ansible-test               : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

Die Ausführung dauert relativ lange. Es werden alle Dateien hochgeladen.

Stoppuhr:
* real	4m23.504s, user	1m10.928s, sys	0m42.812s

### Aufruf ohne --check --diff ohne Abgleich

Aufruf bei bereits aktualisiertem Zielrechner, es müssen keinerlei Dateien abgeglichen werden:

```
uli@ulinuc:~/tmp/ansible-test$ ansible-playbook copy-folder.yml
...
PLAY RECAP *******************************************************************************************************
ansible-test               : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

Die Ausführung dauert sehr lange. Es werden keinerlei Dateien hochgeladen.

Stoppuhr:
* real	3m23.959s, user	0m51.380s, sys	0m28.668s
* real	3m17.905s, user	0m49.404s, sys	0m27.012s
* real	3m25.674s, user	0m53.100s, sys	0m29.912s

### Abgleich löschen

```
uli@ulinuc:~/tmp/ansible-test$ ssh ansible-test rm -rf /tmp/copied-by-ansible
```

Dateien komplett kopieren mit `synchronize`
-------------------------------------------

### Erster Aufruf mit --check --diff

```
uli@ulinuc:~/tmp/ansible-test$ ansible-playbook synchronize-folder.yml --check --diff
PLAY [all] *******************************************************************************************************************************************************************************************************************************

TASK [Gathering Facts] *******************************************************************************************************************************************************************************************************************
[WARNING]: Platform linux on host ansible-test is using the discovered Python interpreter at /usr/bin/python, but future installation of another Python interpreter could change this. See
https://docs.ansible.com/ansible/2.9/reference_appendices/interpreter_discovery.html for more information.
ok: [ansible-test]

TASK [synchronize-folder-role : Synchronize complete folder] *****************************************************************************************************************************************************************************
cd+++++++++ rollout/
cd+++++++++ rollout/apache2/
cd+++++++++ rollout/apache2/conf.d/
<f+++++++++ rollout/apache2/conf.d/template.conf
cd+++++++++ rollout/apache2/vhosts.d/
<f+++++++++ rollout/apache2/vhosts.d/template.vhost
...
```

Die Ausführung läuft sehr schnell durch.
Man sieht "kryptische DIFFs".

Stoppuhr - `time ansible-playbook synchronize-folder.yml --check --diff`
* real	0m10.178s, user	0m4.208s, sys	0m0.664s
* real	0m7.990s, user	0m3.952s, sys	0m0.912s
* real	0m7.629s, user	0m3.748s, sys	0m0.724s

### Aufruf ohne --check --diff mit Komplettabgleich

Aufruf bei leerem Zielrechner, es müssen alle Dateien abgeglichen werden:

```
uli@ulinuc:~/tmp/ansible-test$ ansible-playbook synchronize-folder.yml
...
PLAY RECAP *******************************************************************************************************
ansible-test               : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

Die Ausführung geht sehr schnell. Es werden alle Dateien hochgeladen.

Stoppuhr:
* real	0m11.110s, user	0m4.572s, sys	0m0.836s

### Aufruf ohne --check --diff ohne Abgleich

Aufruf bei bereits aktualisiertem Zielrechner, es müssen keinerlei Dateien abgeglichen werden:

```
uli@ulinuc:~/tmp/ansible-test$ ansible-playbook synchronize-folder.yml
...
PLAY RECAP *******************************************************************************************************
ansible-test               : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

Die Ausführung geht sehr schnell. Es werden keinerlei Dateien hochgeladen.

Stoppuhr:
* real	0m8.789s, user	0m4.544s, sys	0m0.776s
* real	0m8.449s, user	0m4.312s, sys	0m0.732s
* real	0m8.281s, user	0m4.164s, sys	0m0.772s

### Abgleich löschen

```
uli@ulinuc:~/tmp/ansible-test$ ssh ansible-test rm -rf /tmp/copied-by-ansible
```

Dateien komplett kopieren mit `copy` und `filetree`
---------------------------------------------------

### Erster Aufruf mit --check --diff

```
uli@ulinuc:~/tmp/ansible-test$ ansible-playbook copy-filetree.yml --check --diff
TASK [copy-filetree-role : Copy complete folder using copy and filetree] *****************************************************************************************************************************************************************
skipping: [ansible-test] => (item={'group': u'uli', 'uid': 1000, 'state': 'directory', 'gid': 1000, 'mode': '0775', 'mtime': 1588249802.2199366, 'owner': 'uli', 'path': u'apache2', 'size': 28, 'root': u'/home/uli/tmp/ansible-test/files/rollout/', 'ctime': 1588249802.2199366}) 
skipping: [ansible-test] => (item={'group': u'uli', 'uid': 1000, 'state': 'directory', 'gid': 1000, 'mode': '0775', 'mtime': 1588249820.3719356, 'owner': 'uli', 'path': u'data', 'size': 8, 'root': u'/home/uli/tmp/ansible-test/files/rollout/', 'ctime': 1588249820.3719356}) 
skipping: [ansible-test] => (item={'group': u'uli', 'uid': 1000, 'state': 'directory', 'gid': 1000, 'mode': '0775', 'mtime': 1588249859.3399334, 'owner': 'uli', 'path': u'apache2/conf.d', 'size': 26, 'root': u'/home/uli/tmp/ansible-test/files/rollout/', 'ctime': 1588249859.3399334}) 
skipping: [ansible-test] => (item={'group': u'uli', 'uid': 1000, 'state': 'directory', 'gid': 1000, 'mode': '0775', 'mtime': 1588249879.5759323, 'owner': 'uli', 'path': u'apache2/vhosts.d', 'size': 28, 'root': u'/home/uli/tmp/ansible-test/files/rollout/', 'ctime': 1588249879.5759323}) 
--- before
+++ after: /home/uli/tmp/ansible-test/files/rollout/apache2/conf.d/template.conf
@@ -0,0 +1 @@
+Erste Version - template.conf

changed: [ansible-test] => (item={'src': u'/home/uli/tmp/ansible-test/files/rollout/apache2/conf.d/template.conf', 'group': u'uli', 'uid': 1000, 'state': 'file', 'gid': 1000, 'mode': '0664', 'mtime': 1588249859.3399334, 'owner': 'uli', 'path': u'apache2/conf.d/template.conf', 'size': 30, 'root': u'/home/uli/tmp/ansible-test/files/rollout/', 'ctime': 1588249859.3399334})
--- before
+++ after: /home/uli/tmp/ansible-test/files/rollout/apache2/vhosts.d/template.vhost
@@ -0,0 +1 @@
+Erste Version - template.vhost

changed: [ansible-test] => (item={'src': u'/home/uli/tmp/ansible-test/files/rollout/apache2/vhosts.d/template.vhost', 'group': u'uli', 'uid': 1000, 'state': 'file', 'gid': 1000, 'mode': '0664', 'mtime': 1588249879.5759323, 'owner': 'uli', 'path': u'apache2/vhosts.d/template.vhost', 'size': 31, 'root': u'/home/uli/tmp/ansible-test/files/rollout/', 'ctime': 1588249879.5759323})
skipping: [ansible-test] => (item={'group': u'uli', 'uid': 1000, 'state': 'directory', 'gid': 1000, 'mode': '0775', 'mtime': 1588249820.3719356, 'owner': 'uli', 'path': u'data/with', 'size': 8, 'root': u'/home/uli/tmp/ansible-test/files/rollout/', 'ctime': 1588249820.3719356}) 
skipping: [ansible-test] => (item={'group': u'uli', 'uid': 1000, 'state': 'directory', 'gid': 1000, 'mode': '0775', 'mtime': 1588249820.3719356, 'owner': 'uli', 'path': u'data/with/deep', 'size': 10, 'root': u'/home/uli/tmp/ansible-test/files/rollout/', 'ctime': 1588249820.3719356}) 
skipping: [ansible-test] => (item={'group': u'uli', 'uid': 1000, 'state': 'directory', 'gid': 1000, 'mode': '0775', 'mtime': 1588250159.7319174, 'owner': 'uli', 'path': u'data/with/deep/paths', 'size': 1400, 'root': u'/home/uli/tmp/ansible-test/files/rollout/', 'ctime': 1588250159.7319174}) 
--- before
+++ after: /home/uli/tmp/ansible-test/files/rollout/data/with/deep/paths/001.txt
@@ -0,0 +1 @@
+Datei 001 - 1

changed: [ansible-test] => (item={'src': u'/home/uli/tmp/ansible-test/files/rollout/data/with/deep/paths/001.txt', 'group': u'uli', 'uid': 1000, 'state': 'file', 'gid': 1000, 'mode': '0664', 'mtime': 1588250159.5719173, 'owner': 'uli', 'path': u'data/with/deep/paths/001.txt', 'size': 14, 'root': u'/home/uli/tmp/ansible-test/files/rollout/', 'ctime': 1588250159.5719173})
--- before
+++ after: /home/uli/tmp/ansible-test/files/rollout/data/with/deep/paths/002.txt
@@ -0,0 +1 @@
+Datei 002 - 2

...
```

Die Ausführung dauert relativ lange.
Man sieht "schöne DIFFs". Leider sieht man auch blaue Warnmeldungen mit "skipping: ..."

Stoppuhr - `time ansible-playbook copy-filetree.yml --check --diff`
* real	4m21.437s, user	1m9.868s, sys	0m47.288s
* real	4m26.132s, user	1m10.852s, sys	0m49.024s
* real	5m46.500s, user	1m45.084s, sys	1m7.800s

### Aufruf ohne --check --diff mit Komplettabgleich

Aufruf bei leerem Zielrechner, es müssen alle Dateien abgeglichen werden:

```
uli@ulinuc:~/tmp/ansible-test$ ansible-playbook copy-filetree.yml
...
PLAY RECAP *******************************************************************************************************
ansible-test               : ok=5    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

Die Ausführung dauert relativ lange. Es werden alle Dateien hochgeladen.

Stoppuhr:
* real	5m13.661s, user	1m28.992s, sys	0m58.184s

### Aufruf ohne --check --diff ohne Abgleich

Aufruf bei bereits aktualisiertem Zielrechner, es müssen keinerlei Dateien abgeglichen werden:

```
uli@ulinuc:~/tmp/ansible-test$ ansible-playbook copy-filetree.yml
...
PLAY RECAP *******************************************************************************************************
ansible-test               : ok=5    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

Die Ausführung dauert sehr lange. Es werden keinerlei Dateien hochgeladen.

Stoppuhr:
* real	4m10.495s, user	1m7.376s, sys	0m42.172s
* real	4m24.945s, user	1m7.496s, sys	0m45.588s
* real	4m52.096s, user	1m23.396s, sys	0m56.076s

### Abgleich löschen

```
uli@ulinuc:~/tmp/ansible-test$ ssh ansible-test rm -rf /tmp/copied-by-ansible
```

Dateien komplett kopieren mit `file` und `filetree`
--------------------------------------------------

Das funktioniert prinzipbedingt nicht!
"file" kann zwar Verzeichnisse anlegen, aber keine
Dateien kopieren!


Grundaufbau
-----------

### ansible.cfg

```
[defaults]
inventory       = inventory.yml
```

### inventory.yml

```
all:
  hosts:
    ansible-test
```

### copy-one-by-one.yml

```
---
# file: copy-one-by-one.yml
- hosts: all
  roles:
    - copy-one-by-one-role
```

### copy-folder.yml

```
---
# file: copy-folder.yml
- hosts: all
  roles:
    - copy-folder-role
```

### synchronize-folder.yml

```
---
# file: synchronize-folder.yml
- hosts: all
  roles:
    - synchronize-folder-role
```

### copy-filetree.yml

```
---
# file: copy-filetree.yml
- hosts: all
  roles:
    - copy-filetree-role
```

### file-filetree.yml

```
---
# file: file-filetree.yml
- hosts: all
  roles:
    - file-filetree-role
```

### files/rollout/apache2/conf.d/template.conf

```
Erste Version - template.conf
```

### files/rollout/apache2/vhosts.d/template.vhost

```
Erste Version - template.vhost
```

### files/rollout/data/with/deep/paths

Dateien erzeugen mit:

```
for i in $(seq 001 100); do \
  ii=$(printf "%03d" $i);
  echo "Datei $ii - $i" >files/rollout/data/with/deep/paths/$ii.txt;
done
```

### roles/copy-one-by-one-role/tasks/main.yml

```
---
# tasks file for copy-one-by-one-role
- include: copy-one-by-one.yml
```

### roles/copy-one-by-one-role/tasks/copy-one-by-one.yml

```
---
# tasks/copy-one-by-one.yml
- name: create folder vhosts.d
  file: path=/tmp/copied-by-ansible/apache2/vhosts.d state=directory

- name: create folder conf.d
  file: path=/tmp/copied-by-ansible/apache2/conf.d state=directory  

- name: create folder deep paths
  file: path=/tmp/copied-by-ansible/data/with/deep/paths state=directory

- name: copy template.vhost
  copy: src=rollout/apache2/vhosts.d/template.vhost dest=/tmp/copied-by-ansible/apache2/vhosts.d/template.vhost

- name: copy template.conf
  copy: src=rollout/apache2/conf.d/template.conf dest=/tmp/copied-by-ansible/apache2/conf.d/template.conf

#- name: copy $ii.txt
#  copy: src=rollout/data/with/deep/paths/$ii.txt dest=/tmp/copied-by-ansible/data/with/deep/paths/$ii.txt

- name: copy 001.txt
  copy: src=rollout/data/with/deep/paths/001.txt dest=/tmp/copied-by-ansible/data/with/deep/paths/001.txt

- name: copy 002.txt
  copy: src=rollout/data/with/deep/paths/002.txt dest=/tmp/copied-by-ansible/data/with/deep/paths/002.txt
...

- name: copy 100.txt
  copy: src=rollout/data/with/deep/paths/100.txt dest=/tmp/copied-by-ansible/data/with/deep/paths/100.txt
```

### roles/copy-folder-role/tasks/main.yml

```
---
# tasks file for copy-folder-role
- include: copy-folder.yml
```

### roles/copy-folder-role/tasks/copy-folder.yml

```
---
# tasks/copy-folder.yml
- name: Copy complete folder
  copy: src=rollout dest=/tmp/copied-by-ansible
```

### roles/synchronize-folder-role/tasks/main.yml

```
---
# tasks file for synchronize-folder-role
- include: synchronize-folder.yml
```

### roles/synchronize-folder-role/tasks/synchronize-folder.yml

```
---
# tasks/synchronize-folder
- name: Synchronize complete folder
  synchronize: src=rollout dest=/tmp/copied-by-ansible times=no checksum=yes delete=yes
```

### roles/copy-filetree-role/tasks/main.yml

```
---
# tasks file for copy-filetree-role
- include: copy-filetree.yml
```

### roles/copy-filetree-role/tasks/copy-filetree.yml

```
---
# tasks/copy-filetree.yml
- name: create folder vhosts.d
  file: path=/tmp/copied-by-ansible/apache2/vhosts.d state=directory

- name: create folder conf.d
  file: path=/tmp/copied-by-ansible/apache2/conf.d state=directory

- name: create folder deep paths
  file: path=/tmp/copied-by-ansible/data/with/deep/paths state=directory

- name: Copy complete folder using copy and filetree
  copy:
    src: "{{ item.src }}"         
    dest: "/tmp/copied-by-ansible/{{ item.path }}"
  with_filetree: rollout/
  when: item.state == 'file'
```

### roles/file-filetree-role/tasks/main.yml

```
---
# tasks file for file-filetree-role
- include: file-filetree.yml
```

### roles/file-filetree-role/tasks/file-filetree.yml

```
---
# tasks/file-filetree.yml    
- name: create folder vhosts.d
  file: path=/tmp/copied-by-ansible/apache2/vhosts.d state=directory

- name: create folder conf.d
  file: path=/tmp/copied-by-ansible/apache2/conf.d state=directory

- name: create folder deep paths
  file: path=/tmp/copied-by-ansible/data/with/deep/paths state=directory

- name: Copy complete folder using file and filetree    
  file:
    path: /tmp/copied-by-ansible/{{item.path}}
  with_filetree: rollout/
  when: item.state == 'file'
```

### Test

```
uli@desktop:~/ansible-test$ ansible all -m ping
ansible-test | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "ping": "pong"
}
```

Bewertung
---------

| Kriterium                              | Gewichtung | copy-einzeln  | copy-komplett | synchronize-komplett | copy-filetree | file-filetree |
| -------------------------------------- | ---------- | ------------- | ------------- | -------------------- | ------------- | ------------- |
| Einfache Anwendung                     |            | nein          | ja            | ja                   | ja            | NA            |
| Schnelle Ausführung - Zeit in Sekunden |            | 240 - 400s    | 200 - 300s    | 10s                  | 240 - 400s    | NA            |
| Schnelle Ausführung - Wertung          |            | sehr schlecht | schlecht      | gut                  | sehr schlecht | NA            |
| Ausgabe `--check --diff`               |            | super         | sehr schlecht | mittel               | super         | NA            |
| Irritierende Zusatzausgaben            |            | super         | super         | super                | schlecht      | NA            |

Links
-----

Änderungen
----------

* 2020-04-30: Erste Version
