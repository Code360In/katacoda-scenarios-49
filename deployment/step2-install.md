The project provides a script that will install Sedna. 

To remove the taint added of master node by the command above, you can run: 
`kubectl taint node $HOSTNAME node-role.kubernetes.io/master-`{{execute}}

Install Sedna by the command above, you can run:

`curl https://raw.githubusercontent.com/kubeedge/sedna/main/scripts/installation/install.sh | SEDNA_GM_NODE=$HOSTNAME SEDNA_ACTION=create bash -`{{execute}}

Verify Sedna has started its control plane by the command above, you can run:

`kubectl get deployments,pods,services -n sedna`{{execute}}

Optimally, the Pod status should say _Running_ in about ~15 seconds.
