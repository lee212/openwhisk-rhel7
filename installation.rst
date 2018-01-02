OpenWhisk using Kubernetes and Docker
===============================================================================

OpenWhisk components are delivered as a Kubernetes pod by deploying Docker
containers. The base images are pulled from `OpenWhisk repository on Docker Hub
<https://hub.docker.com/u/openwhisk/>`_ to build clusters and some tips and
issues that we experienced are provided in this guide to help other users.

Docker 1.12, not 17.09 ce
-------------------------------------------------------------------------------

Kubernetes needs a container engine, e.g. Docker but the latest one may not
work as expected. We find two reasons why Docker 1.12 is chosen, because 1)
`Kubernetes recommends v1.12
<https://kubernetes.io/docs/setup/independent/install-kubeadm/#installing-docker>`_
(also `here <https://github.com/kubernetes/kubernetes/issues/44140>`_) and 2)
`Docker CE 17.09 does not support
RHEL7<https://docs.docker.com/engine/installation/linux/docker-ee/rhel/>`_

.. note:: Docker package is also available in a RHEL extras repository which is
        recommended to receive support by Red Hat.

Manual Installation
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This section provides a list of commands to install Docker manually, instead of
using package pamangers i.e. yum to use an exact version of Docker.

Download Docker `rpm` package:

::
       
        wget https://packages.docker.com/1.12/yum/repo/main/centos/7/Packages/docker-engine-1.12.6.cs8-1.el7.centos.x86_64.rpm

.. note:: If this link is broken, please look up substitute the repository.

The following command to install:

::

        sudo yum -y install docker-engine-1.12.6.cs8-1.el7.centos.x86_64.rpm
        
.. note:: --installroot=/opt/docker may used to separate installed location, set variable by `export DOCKER_ROOT=/opt/docker`


Service file from the Moby Project:

::

        wget https://raw.githubusercontent.com/moby/moby/master/contrib/init/systemd/docker.service -P $DOCKER_ROOT/etc/systemd/system
        wget https://raw.githubusercontent.com/moby/moby/master/contrib/init/systemd/docker.socket -p $DOCKER_ROOT/etc/systemd/system

(Optional) for /opt/docker:

::

        export DOCKER_ROOT_ESCAPED=`echo $DOCKER_ROOT|sed -e 's/\//\\\\\//g'`
        sed -ie "s/ExecStart=/Environment=\"LD_LIBRARY_PATH=$DOCKER_ROOT_ESCAPED\/usr\/lib64\"\nEnvironment=\"PATH=\$PATH:$DOCKER_ROOT_ESCAPED\/bin:$DOCKER_ROOT_ESCAPED\/sbin:\/sbin\"\nExecStart=$DOCKER_ROOT_ESCAPED/" $DOCKER_ROOT/etc/systemd/system/docker.service 
        sudo mkdir /var/lib/docker
        sudo mkdir /etc/docker
        sudo ln -s /opt/docker/etc/systemd/system/docker.s* /etc/systemd/system

Devicemapper as storage driver        

::

        cat << EOF > /etc/docker/daemon.json
        {
          "storage-driver": "devicemapper"
        }
        EOF

Service reload/restart:

::

        
        sudo systemctl daemon-reload
        sudo systemctl start docker

Kubernetes (WIP)
-------------------------------------------------------------------------------

::

        cat <<EOF > /etc/yum.repos.d/kubernetes.repo
        [kubernetes]
        name=Kubernetes
        baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
        enabled=1
        gpgcheck=1
        repo_gpgcheck=1
        gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
        EOF

        setenforce 0
        yum install -y kubelet kubeadm kubectl

        sudo sed -i "s/ExecStart=\/usr\/bin/ExecStart=$DOCKER_ROOT_ESCAPED\/bin\/usr\/bin/" $DOCKER_ROOT/etc/systemd/system/kubelet.service
        sudo sed -i "s/ExecStart=\/usr\/bin/ExecStart=$DOCKER_ROOT_ESCAPED\/bin\/usr\/bin/" $DOCKER_ROOT/etc/systemd/system/kubelet.service.d/10-kubeadm.conf
        sudo sed -i "s/cgroup-driver=systemd/cgroup-driver=cgroupfs/" $DOCKER_ROOT/etc/systemd/system/kubelet.service.d/10-kubeadm.conf
        sudo sed -i "s/KUBELET_EXTRA_ARGS/KUBELET_EXTRA_ARGS --fail-swap-on=false/" $DOCKER_ROOT/etc/systemd/system/kubelet.service.d/10-kubeadm.conf

        # Weave net CNI addon
        export kubever=$(kubectl version | base64 | tr -d '\n')
        kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$kubever"


        export K8S_ROOT=/opt/kubernetes

        # Master to check token
        sudo $K8S_ROOT/bin/kubeadm token list

        # Node to join
        sudo $K8S_ROOT/bin/kubeadm join --token <TOKEN> <MASTER_IP>:<MASTER_PORT i.e. 6443> --skip-preflight-checks --discovery-token-unsafe-skip-ca-verification

        # Label for nodes
        $K8S_ROOT/bin/kubectl label node  <NODE_HOSTNAME> node-role.kubernetes.io/node=

