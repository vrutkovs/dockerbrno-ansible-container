% title: Ansible-container
% subtitle: Building a Docker image without Dockerfile and shell
% author: Vadim Rutkovsky
% author: Red Hat Czech, http://github.com/vrutkovs
% thankyou: Thanks! Questions?
% favicon: http://www.stanford.edu/favicon.ico

---
title: Typical case
build_lists: true

Grafana - an app to build dashboards and grafs from various sources

Default choice: 

Why I didn't like default Grafana-XXL image from dockerhub:

- Bloated: all the grafana plugins installed
- Bloated: uses gosu to run
- Has auto-upgrade on
- Doesn't have read-only anonymous access

---
title: Enter the Dockerfile madness

<pre class="prettyprint">
ENV GRAFANA_VERSION=3.1.1-1470047149 \
    GF_PLUGIN_DIR=/grafana-plugins \
    UPGRADEALL=true

COPY ./run.sh /run.sh

RUN apt-get update && \
  apt-get -y --force-yes --no-install-recommends 
      install libfontconfig curl ca-certificates git jq && \
  curl https://grafanarel.s3.amazonaws.com/builds/grafana_${GRAFANA_VERSION}_amd64.deb >
      /tmp/grafana.deb && \
  dpkg -i /tmp/grafana.deb && \
  rm /tmp/grafana.deb && \
  curl -L https://github.com/tianon/gosu/releases/download/1.7/gosu-amd64 
      > /usr/sbin/gosu && \
  chmod +x /usr/sbin/gosu && \
<b>for plugin in $(curl -s https://grafana.net/api/plugins?orderBy=name | 
      jq '.items[] | select(.internal=='false') | .slug' | tr -d '"'); do 
      grafana-cli --pluginsDir "${GF_PLUGIN_DIR}" plugins install $plugin; done</b>
</pre>

---
title: Ansible Container
build_lists: true

Ansible Container is a tool to build Docker images and orchestrate containers using only Ansible playbooks.

<pre class='prettyprint'>
https://github.com/ansible/ansible-container
sudo pip install ansible-container
</pre>

How it works:

 - `init` prepares directory structure
 - `build` runs Ansible playbooks to build an image
 - `run` launches the container
 - `push` pushes the built image to the registry
 - `shipit` deploy the container to a cloud provider

---
title: Directory structure after `init`

<pre>
.
└── ansible
    ├── container.yml
    ├── main.yml
    └── requirements.txt
</pre>

- `main.yml` - playbook to build the image
- `container.yml` - orchestration
- `requirements.txt` - requirements for ansible module


---
title: container.yml - orchestration

<pre class='prettyprint'>
version: "1"
services:
  grafana:
    image: centos:7
    ports:
      - "3000:3000"
    user: "grafana"
    command: ["/usr/sbin/grafana-server",
              "--homepath=/usr/share/grafana",
              "--config=/etc/grafana/grafana.ini",
              "cfg:default.paths.data=/var/lib/grafana",
              "cfg:default.paths.logs=/var/log/grafana",
              "cfg:default.paths.plugins=/grafana-plugins"]
<b>
registries:
  atomicregistry:
    url: atomic-registry.usersys.redhat.com:5000
    namespace: vrutkovs
</b>
</pre>

---
title: main.yml - playbook

<pre class='prettyprint fullslide'>
- hosts: grafana
  vars:
    rpm: https://grafanarel.s3.amazonaws.com/builds/grafana-3.1.1-1470047149.x86_64.rpm
  tasks:
    - name: Upgrade all packages
      yum: name=* state=latest

    - name: Install grafana
      yum: name="{{ rpm }}" state=present
    <b>
    - name: Install plugins
      command: grafana-cli --pluginsDir /grafana-plugins plugins install {{ item }}
      with_items:
        - grafana-piechart-panel
        - alexanderzobnin-zabbix-app
        - mtanda-histogram-panel
    </b>
    - name: Copy config file
      copy: src="grafana.ini" dest=/etc/grafana owner=grafana

    - name: Make sure grafana user is the owner of its dirs
      file: name={{ item }} state=directory owner=grafana recurse=true
      with_items:
        - /grafana-plugins
        - /var/lib/grafana
        - /var/log/grafana

    - name: Clean yum files
      command: yum clean all
</pre>

---
title: Build process

 - Creates `ansible_ansible-container` and `ansible_<service name>` containers
 - `ansible-container` connects to service containers
 - Runs the playbook 
 - Commits the resulting images
 - Can flatten the image in one layer

---
title: Build log

<pre class='fullslide prettyprint'>
Starting Docker Compose engine to build your images...
Attaching to ansible_ansible-container_1
Attaching to ansible_ansible-container_1, ansible_grafana_1
ansible-container_1  | 
ansible-container_1  | PLAY [grafana] ********************************************
ansible-container_1  | 
ansible-container_1  | TASK [setup] **********************************************
ansible-container_1  | ok: [grafana]
ansible-container_1  | 
ansible-container_1  | TASK [Upgrade all packages] *******************************
ansible-container_1  | ok: [grafana]
ansible-container_1  | 

[snip]

ansible-container_1  | PLAY RECAP ************************************************
ansible-container_1  | grafana: ok=7    changed=2    unreachable=0    failed=0   
ansible-container_1  | 
ansible_ansible-container_1 exited with code 0
Aborting on container exit...
Stopping ansible_grafana_1 ... 
Stopping ansible_grafana_1 ... done
Exporting built containers as images...
Committing image...
Exported grafana-xxl-grafana with image ID 
	sha256:c15c6a907eb24e0f2384e0bf7744986e793a699b5741a644fb64ab8613704cec
Cleaning up grafana build container...
Cleaning up Ansible Container builder...
</pre>

---
title: Running and shipping it

`run` starts services according to their description in `container.yml`

`push` pushes the resulting image to a specified registry

`shipit kubernetes` creates a playbook which deploys the image to a k8s cluster

`shipit openshift` same for Openshift

---
title: Future

A very young project, has lots of plans:

- Build caching
- Detached run, stop and restart
- Custom volumes and build variables
- rkt and OCI support

[Github](https://github.com/ansible/ansible-container)

[Docs](https://docs.ansible.com/ansible-container/)

 #ansible-container on Freenode
