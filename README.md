# KontenaDemo
09/26/17 demo for running hyperspace on Kontena. 

## Install Kontena cluster
First, let's create the master of the cluster.

`kontena upcloud master create --username $UPCLOUD_API_USER --password $UPCLOUD_API_PASSWORD --zone nl-ams1 --plan 1xCPU-1GB --ssh-key kontena_rsa.pub`

Create a logical container, known as a 'grid' to house the worker nodes.

`kontena grid create --initial-size=2 demogrid`

Create worker nodes.

`kontena upcloud node create --grid demogrid --username $UPCLOUD_API_USER --password $UPCLOUD_API_PASSWORD  --zone nl-ams1 --plan 2xCPU-2GB --count 2 --ssh-key kontena_rsa.pub`

Install the application.

`kontena stack install <stackfile.yaml>`

Check that 'alles gut'.

`kontena service ls`

Checkout the external address of the load balancer.

`kontena service show hyperspace/loadbalancer `

Edit your etc host file to point the domain to the address or use a proper DNS record. Now, let's check that we can play our gaymmmm!! Should work - finger crossed :P

## Let's play around a little

If you take a peek at the hstack.yaml file, you'll notice that I used a variable section with a user prompt and a default value. The load balancer routes traffic based on the `app_domain` variable. So, you could have multiple deployments of the same app, for example, testing purposes, and route test traffic to those deployments by using different `app_domain` variable.  

One way to do the same thing as I did with the variables section, would be to use secrets in the cluster. Then the only thing changing between different clusters/deployments would be the secrets. You could copy the deployment to all clusters and be sure that correct configuration variables are used every time.

The app deployment in `hsteststack.yaml` uses Kontena vault to consume environment variables. More on how to use Kontena vault, check out the [docs](https://www.kontena.io/docs/using-kontena/vault.html).








