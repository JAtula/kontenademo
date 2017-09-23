# KontenaDemo
09/26/17 demo for running hyperspace on Kontena. 

## Install Kontena cluster
First, let's create the master of the cluster.

`kontena upcloud master create --username myapiuser --password myapiuserpwd`

Create a logical container, known as a 'grid' to house the worker nodes.

`kontena grid create --initial-size=3 demogrid`

Create worker nodes.

`kontena upcloud node create --grid demogrid --username $UPCLOUD_API_USER --password $UPCLOUD_API_PASSWORD  --zone nl-ams1 --plan 2xCPU-2GB --count 3`

Install the application.

`kontena stack install <stackfile.yaml>`

Check that 'alles gut'.

`kontena service ls`

Checkout the external address of the load balancer.

`kontena service show hyperspace/loadbalancer `

Edit your etc host file to point the domain to the address or use a proper DNS record. Now, let's check that we can play our gaymmmm!! Should work - finger crossed :P








