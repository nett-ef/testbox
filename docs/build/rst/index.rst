
testbox
*******

*testbox* is an Ansible playbook that deploys a minimalistic
single-node Kubernetes cluster. It is smaller than `Kubespray
<https://kubespray.io/>`_ having fewer components and options. Most of
the Kubernetes components were switched off, allowing it to run on
low-end computers like Raspberry Pi. It can be used for development
when a quick microk8s environment is needed.

Supported OS: Ubuntu 22.04 LTS.


A note on reliability and security
**********************************

With the playbook, you will get a cloud-native environment that can
come in handy. The environment has zero reliability, and almost no
security measures are applied. Study the topics to meet your SLOs and
develop your security policy.


Requirements
************

The following system packages must be installed to follow the README:

*  `Ansible >= 2.13.2
   <https://docs.ansible.com/ansible/latest/installation_guide/index.html>`_

*  `Helm >= 3.9.3 <https://helm.sh/docs/intro/install/>`_

*  `kubectl >= 1.24.2 <https://kubernetes.io/docs/tasks/tools/>`_


Usage
*****

1. Configure the Ansible and create the inventory.

   Example:

      *  *environments/dev.example/ansible/hosts.yaml*

      *  *environments/dev.example/ansible/secrets.yaml*

      *  *environments/dev.example/ansible/group_vars/all.yaml*

      *  *environments/dev.example/ansible/group_vars/example_group.yaml*

   The *secrets.yaml* file contains *ansible_become_password* for the
   hosts. The key is encrypted with the `Ansible Vault
   <https://docs.ansible.com/ansible/latest/user_guide/vault.html>`_.

2. Run the play:

   ::

      % ansible-playbook testbox.yaml \
          -i environments/dev.example/ansible/hosts.yaml \
          -i environments/dev.example/ansible/secrets.yaml \
          --user username \
          --private-key ~/.ssh/username_ed25519 \
          --vault-id testbox@~/.vault/testbox


Tags
****

*  *common* - common administration tasks: create groups, add users to
   the groups, etc.

*  *hardening* - basic security tasks.

*  *unattended_upgrades* - enable unattended-upgrades.

*  *nftables* - enable nftables.

*  *nft_rules* - apply nft rules.

*  *troubleshooting* - install troubleshooting tools.

*  *microk8s* - install microk8s.


How to change the sshd port
***************************

Run the *hardening* tasks with *ansible_port* redefined:

::

   % ansible-playbook testbox.yaml \
       -i environments/dev.example/ansible/hosts.yaml \
       -i environments/dev.example/ansible/secrets.yaml \
       --user username \
       --private-key ~/.ssh/username_ed25519 \
       --vault-id testbox@~/.vault/testbox \
       --tags hardening \
       -e ansible_port 22


How to open firewall ports
**************************

1. Add the port number to the *nft_accept_tcp_ports* or
   *nft_accept_udp_ports* list in the variables file:
   *environments/dev.example/ansible/group_vars/example_group.yaml*.

2. Run the play with ``--tags nft_rules`` to apply the change.


How to create an application account
************************************

1. Add the user name to the *daemon_users* list in the variables file:
   *environments/dev.example/ansible/group_vars/example_group.yaml*.

2. Run the play with ``--tags common`` to apply the change and get the
   UID.


How to connect with kubectl
***************************

1. Get the kubeconfig file from the server:

   ``% microk8s config view``

2. Add *proxy-url:* key to the *-cluster* section of the
   *~/.kube/config* file, for example:

   ::

      apiVersion: v1
      clusters:
      - cluster:
          certificate-authority-data: <CA data>
          server: https://<host address>:16443
          proxy-url: socks5://localhost:1081

   Documentation: `Use a SOCKS5 Proxy to Access the Kubernetes API
   <https://kubernetes.io/docs/tasks/extend-kubernetes/socks5-proxy-access-api/>`_

3. Use SSH dynamic port forwarding, for example:

   ``% ssh -D 1081 -q -N <hostname> &``


How to run workloads
********************

1. Deploy the application:

   ``% helm install <name> <chart>``.

2. Change the service type to *NodePort*, making it accessible outside
   the cluster.

   Documentation: `Type NodePort
   <https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport>`_

3. Optionally change the service *externalTrafficPolicy* to *Local* to
   disable SNAT on the cluster network.

   Documentation: `Preserving the client source IP
   <https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/#preserving-the-client-source-ip>`_


Deployment example
******************

`Gitea <https://gitea.io/en-us/>`_ will be installed from the `Helm
chart <https://artifacthub.io/packages/helm/gitea/gitea>`_.


Installation
============

1. Add the *gitea-charts* repo:

   ::

      % helm repo add gitea-charts https://dl.gitea.io/charts/
      % helm repo update

2. Get *values.yaml* from the `helm-chart repo
   <https://gitea.com/gitea/helm-chart/src/branch/main/values.yaml>`_.

3. Configure the application, for example:
   *environments/dev.example/k8s/gitea/values.yaml*.

4. Install the application:

   ``% helm install gitea gitea-charts/gitea -f values.yaml``

5. Check the application status:

   ``% kubectl get all -l app=gitea``


Security configuration
======================

1. Create *gitea-admin-secret* as stated in the *values.yaml*:

   ::

      kubectl create secret generic gitea-admin-secret \
        --from-file=username=./username.txt \
        --from-file=password=./password.txt

   Documentation: `Managing Secrets using kubectl
   <https://kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-kubectl/>`_

2. Create the application account and get the UID (*1001*).

3. Set *podSecurityContext* and *containerSecurityContext*:

   ::

      podSecurityContext:
        fsGroup: 1001
      containerSecurityContext:
         allowPrivilegeEscalation: false
         capabilities:
           drop:
             - ALL
           add:
             - SYS_CHROOT
         privileged: false
         readOnlyRootFilesystem: true
         runAsGroup: 1001
         runAsNonRoot: true
         runAsUser: 1001

   Documentation: `Configure a Security Context for a Pod or Container
   <https://kubernetes.io/docs/tasks/configure-pod-container/security-context/>`_


Updating
========

1. Get updated *values.yaml* from the `helm-chart repo
   <https://gitea.com/gitea/helm-chart/src/branch/main/values.yaml>`_.

2. Merge the configuration.

3. Apply the update:

   ::

      % helm repo update
      % helm upgrade gitea gitea-charts/gitea -f values.yaml
