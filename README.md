# KontenaDemo
09/26/17 demo for running hyperspace on Kontena.

Check out the slide stack [here](https://docs.google.com/presentation/d/1JkeIa-eTu_AiudtCpmY0QR8KPvnxBOrIgmraOyXYYmw/edit?usp=sharing).

## Development environment

As a first, we'll use Packer and the UpCloud provider to create a custom UpCloud VM template to provision a GitLab instance for our version control and CI/CD. 

First, create a sub user account and allow API access for that account in UpCloud user accounts settings. Source the username and password of the API as environment variables and use them in the Packer template. 

Now, we're ready to build the template.

`packer build gitlab.json`

The build will take some time, so let's continue with the actual workload cluster. 

## Install Kontena cluster
First, let's create the master of the cluster.

`kontena upcloud master create --username $UPCLOUD_API_USER --password $UPCLOUD_API_PASSWORD --zone nl-ams1 --plan 1xCPU-1GB --name kontena-master --use-kontena-cloud --ssh-key kontena_rsa.pub`

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

Let's remove the stack after playing a little :) 

`kontena stack rm hyperspace`

## Let's play around a little

If you take a peek at the hstack.yaml file, you'll notice that I used a variable section with a user prompt and a default value. The load balancer routes traffic based on the `app_domain` variable. So, you could have multiple deployments of the same app, for example, testing purposes, and route test traffic to those deployments by using different `app_domain` variable.  

One way to do the same thing as I did with the variables section, would be to use secrets in the cluster. Then the only thing changing between different clusters/deployments would be the secrets. You could copy the deployment to all clusters and be sure that correct configuration variables are used every time.

The app deployment in `hsteststack.yaml` uses Kontena vault to consume environment variables. More on how to use Kontena vault, check out the [docs](https://www.kontena.io/docs/using-kontena/vault.html).

So now, let's create the secrets:

`kontena vault write SSL_CERT "$(cat cert.pem)"`

`kontena vault write KONTENA_LB_VIRTUAL_HOSTS_HYPERSPACE hyperspace.kontenademo.io`

Add port and secret as an environment variable to the load balancer stack.

```
    ports:
      - 80:80
      - 443:443
    affinity:
      - node=={{ lb_node }}
    secrets:
      - secret: SSL_CERT
        name: SSL_CERTS
        type: env
```

Now upgrade the stack:

`kontena stack upgrade loadbalancer loadbalancer.yaml`

### Modify the APP stack

We'll modify the APP stack to consume the domain environment variable as a secret. The full stack looks like this:


```
stack: hyperspace
version: 0.1.1
description: Hyperspace Demo 
services:
  game:
    image: jatula/hyperspace:0.1
    instances: 1
    links:
      - loadbalancer/lb
    environment:
      - KONTENA_LB_INTERNAL_PORT=9393
    secrets:
      - secret: KONTENA_LB_VIRTUAL_HOSTS_HYPERSPACE
        name: KONTENA_LB_VIRTUAL_HOSTS
        type: env
    deploy:
      strategy: ha
      wait_for_port: 9393
    hooks:
      post_start:
        - name: sleep
          cmd: sleep 10
          instances: 1
```

Tail the load balancer log, to see that the backend comes live.

`kontena service logs -t loadbalancer/lb`

### Let's deploy another game

As the Hyperspace game doesn't work well with https, we'll deploy the 2048 game as a substitute. First, we'll create a new domain secret that it reflects the new game. 

`kontena vault write KONTENA_LB_VIRTUAL_HOSTS_2048 2048.kontenademo.io`

And update our hosts file for the appropriate domain.

`echo '<lb address> 2048.kontenademo.io' |sudo tee -a /etc/hosts`

The new stack file for the 2048 is mostly similar to the Hyperspace stack file, but with the custom load balancer settings environment variable that force HTTP traffic to HTTPS.

```
    environment:
      - KONTENA_LB_INTERNAL_PORT=80
      - KONTENA_LB_CUSTOM_SETTINGS=redirect scheme https if !{ ssl_fc }
```

Both games are up and our load balancer is routing traffic beatifully. <3

**Let's go a little bit further...**

One thing that might intress us at some point would be to login to HAProxy portal and check that backends are doing okay. Even though we see that from the logs also. 

The out-of-the-box VPN that comes with Kontena is meant for stuff like this. Connecting to internal services straight from your laptop. Let's give it a go.

Firstly, we'll provision the OpenVPN server `kontena vpn create`. Wait for a while and when it's finished, download the config file to you OpenVPN client of choice.

`kontena vpn config > kontena.ovpn`

Now use the generated configuration with your OpenVPN client and log in to HAProxy statistic portal by using its internal IP and port 1000. Default username and password are: `stats:secret`.

If you scale a service now and watch the stats portal, you'll see the the scaled backend expanding in numbers.

### GitLab

Before moving on to clean up, we'll provision our GitLab instance from the packer image template we created before. 

1. List the available templates.

`curl -XGET --user ${USERN} https://api.upcloud.com/1.2/storage/template`

2. Provision the server.

```
curl -XPOST --user ${USERN} -H "Content-Type:application/json" -d \
'{
  "server": {
    "zone": "nl-ams1",
    "title": "gitlab",
    "hostname": "gitlab",
    "plan": "4xCPU-4GB",
    "storage_devices": {
      "storage_device": [
        {
          "action": "clone",
          "storage": "<tempalate UUID>",
          "title": "gitlab--template-1505932125",
          "size": 30,
          "tier": "maxiops"
        }
      ]
    }
  }
}' \
https://api.upcloud.com/1.2/server
```
3. From the output, copy the public IP and edit your hosts file again. As I already had the URL for GitLab in my hosts file, I'll substitute the address to the current one.

`sudo sed -i -e 's/.*gitlab.kontenademo.io/<lb ip> gitlab.kontenademo.io/g' /etc/hosts`

4. Open up the URL in a browser and go through the initial GitLab installation.

### GitLab CI

The speed up the development process, where going to add a GitLab runner to act as a CI slave and push new commits as Docker images to Docker Hub and to our Kontena cluster. Lastly it'll check that the current version is running and exit.

To start, provision the runner instance the same way you did the GitLab instance *but* take notice of the variables: 

```
{
    "variables": {
      "UPCLOUD_USERNAME": "{{ env `UPCLOUD_API_USER` }}",
      "UPCLOUD_PASSWORD": "{{ env `UPCLOUD_API_PASSWORD` }}",
      "gitlab-address": "<fill in>",
      "gitlab-token": "<fill in>",
      "kontena-master-address": "<fill in>"

    },
```

When the GitLab runner is up and alive, you should register the runner in GitLabs settings and assign the tag "kontena" to it appropriately. I won't go too deep in how you setup the runner. Checkout GitLabs [docs](https://docs.gitlab.com/runner/) for more information.

Next, we'll create the CI file. GitLab looks a the root folder of every branch for a `gitlab-ci.yml` file, and if sees one, It'll use it to construct the CI/CD pipeline for that branch.

We'll be using only master branch, and our CI configuration won't be that fancy either. Something quick'n'dirty to make the job done! 

You can check it out under `hyperspace-app/gitlab-ci.yml`. 

Now, when ever a commit is pushed to the master branch:

1. A Docker image is build with an updated image tag and pushed to Docker Hub.
2. That image is the used to update the Kontena stack deployment of the Hyperspace app.
3. A small validation is runned against the live stack to verify that the image has updated. 



### CleanUP

When you're done playing. Let's remove demo environment.

`kontena stack rm --force <stack name>`
`kontena upcloud node terminate --force <node name>`
`kontena grid rm --force <grid name>`
`kontena master rm --force <master name>`

The UpCloud plugin doesn't have a command to remove the master instance VM, so we'll have to call UpCloud API directly. 

`curl -XGET --user ${USERN} https://api.upcloud.com/1.2/server`
`curl -XPOST --user ${USERN} https://api.upcloud.com/1.2/server/<server UUID>/stop`
`curl -XDELETE --user ${USERN} https://api.upcloud.com/1.2/server/<server UUID>`

Use the same approach to delete the GitLab instance. 

### Done

In this demo, we overlooked many things, like the out-of-the-box support for stateful apps that Kontena provides. You should check out my [blog](https://blog.mecloud.online/upcloud-contena-all-suomi-love-3/) for a more comprehensive tutorial.
















