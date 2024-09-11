====================================================
Lab 01 - Explore Red Hat Product Demo System cluster
====================================================

We will begin with a quick tour of the lab environment. This lab consists of a multi-node kubernetes cluster which has been deployed exclusively for you. This is a sandbox environment. Feel free to play around.

Red Hat Openshift Container Platform configuration review
---------------------------------------------------------

Let's start by getting the nodes in our current cluster:


.. code-block:: shell
    
    oc get nodes

This cluster has 3 control-plane nodes and 3 worker nodes.

We can check the openshift version we are running be running the following command:

.. code-block:: shell
    
    oc version


And we can check the status of the cluster by running:


.. code-block:: shell
    
    oc cluster-info


Portworx configuration review
-----------------------------

Portworx Enterprise is already installed and running on this cluster.  We will investigate the configuration in the next section:

What does Portworx need to be installed?

1. **Drives**: The drive /dev/nvme1n1 is available on each node which we will be using.
2. **Key Value Database (KVDB)**: Such as ETCD. We will be using the Portworx Built-in KVDB instead of deploying our own.
3. **Specification**: Portworx is defined by a spec file, we will create the Portworx cluster using the spec URL.


.. code-block:: shell

   oc get pods -o wide -n portworx -l name=portworx

Check the installation logs:

.. code-block:: shell

    oc -n portworx logs -f $(oc get pods -l name=portworx -n portworx -o jsonpath='{.items[0].metadata.name}')  -c portworx

Add PXCTL alias
---------------------

To make it easier to run pxctl commands, we will add an alias to the shell.

.. code-block:: shell
    
    alias pxctl="oc -n portworx exec $(oc get pods -l name=portworx -n portworx -o jsonpath='{.items[0].metadata.name}') -c portworx -it -- /opt/pwx/bin/pxctl"
    echo 'alias pxctl="oc -n portworx exec $(oc get pods -l name=portworx -n portworx -o jsonpath='{.items[0].metadata.name}') -c portworx -it -- /opt/pwx/bin/pxctl"' >> ~/.bashrc

Take a moment to review the portworx configuration by running the following command:

.. code-block:: shell

    pxctl status

Notice that our Portworx cluster is running on 3 nodes.