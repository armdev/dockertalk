Using Docker in production: Get started today!
==============================================

<h2 class="info-box exercise">Preamble</h2>
<ul>
<li>View the <a href="https://www.slideshare.net/clarenceb/using-docker-in-production-get-started-today" target="_blank">slides for the talk</a> (SlideShare)</li>
<li>You can view the instructions (<code>README.md</code>) in a Markdown viewer (e.g. on Github) or you can open the HTML version (<code>README.html</code>) which is more colourful :)</li>
</ul>
</h2>

<h2 class="info-box exercise">Demo script</h2>

*Tested with Docker Version: Docker CE for AWS (v17.06)*

*A talk by Clarence Bakirtzidis - Email: [clarence.bakirtzidis@elabor8.com.au](mailto:clarence.bakirtzidis@elabor8.com.au), Twitter: [@clarenceb_oz](https://twitter.com/clarenceb_oz)*

We will be using [Docker for AWS](https://docs.docker.com/docker-for-aws/) Community Edition (CE).

There are two ways to deploy Docker for AWS:

* With a pre-existing VPC
* With a new VPC created by Docker

You can use either the AWS Management Console (browser based) or use the AWS CLI (command-line based).

<div class="info-box notice">
<strong>Note</strong>: there will be some costs associated with running through this tutorial on AWS but these costs will be minimal
</div>

**In this tutorial we will use a new VPC created by Docker and the AWS CLI.**

<h2 class="info-box exercise">Prerequisites</h2>

* Access to an AWS account with permissions to use CloudFormation and to create the following objects (see [full set of required permissions](https://docs.docker.com/docker-for-aws/iam-permissions/)):
    * EC2 instances + Auto Scaling groups
    * IAM profiles
    * DynamoDB Tables
    * SQS Queue
    * VPC + subnets and security groups
    * ELB
    * CloudWatch Log Group
* SSH key in the AWS region where you want to deploy (required to access the completed Docker install)
* AWS account that supports EC2-VPC (See the [FAQ for details about EC2-Classic](https://docs.docker.com/docker-for-aws/faqs/))


<div class="info-box notice">
It is recommended that you do not use your AWS root account and instead create a new IAM user.  You can either grant them <a href="https://docs.docker.com/docker-for-aws/iam-permissions/" target="_blank">all these permissions</a> or make them an admin.  We will use a service role with CloudFormation that only has minimum required permissions.
</div>

<h2 class="info-box exercise">Install the AWS CLI and configure your AWS access keys</h2>

### Installation

Refer to the [AWS Command Line Reference](https://aws.amazon.com/cli/).

Simplest way to install:

### Mac / Linux:

```sh
# Requires Python 2.6.5 or higher. Install using pip.
pip install awscli
```

or on Mac, using [Homebrew](https://brew.sh/):

```sh
brew install awscli
```

### Windows:

Download and run the [64-bit](https://s3.amazonaws.com/aws-cli/AWSCLI64.msi) or [32-bit](https://s3.amazonaws.com/aws-cli/AWSCLI32.msi) Windows installer.

### Configuration

See the [Configuring the AWS CLI](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html) page.

For default profile:

```sh
aws configure
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]: ap-southeast-2
Default output format [None]: json
```

For custom profile:

```sh
aws configure --profile dockertutorial
AWS Access Key ID [None]: AKIAI44QH8DHBEXAMPLE
AWS Secret Access Key [None]: je7MtGbClwBF/2Zp9Utk/h3yCo8nvbEXAMPLEKEY
Default region name [None]: ap-southeast-2
Default output format [None]: json
```

<h2 class="info-box exercise">Exercise: Provision a Docker for AWS cluster</h2>

<div class="info-box information">
<strong>Note</strong>: You must have created an EC2 ssh key pair in the desired region before creating the cluster. the keypair name must be called <code>DockerTutorial</code>.
</div>

Make a copy of the `aws_vars.sh.template` file and called it `aws_vars.sh`
Set the required values environment variables:

* `AWS_PROFILE` or leave blank for default profile
* `AWS_REGION`
* `CF_ROLE_ARN` or leave blank to use your IAM user permissions

```sh
./create_stack.sh

# ==>
Creating the CloudFormation Stack...
{
    "StackId": "arn:aws:cloudformation:ap-southeast-2:XXXXXXXXXX:stack/DockerTutorial/7adc6f40-99d5-11e7-a360-50fa575f6862"
}
```

After some time, check the status:

```sh
./describe_stack.sh

# ==>
{
    "Stacks": [
        {
            "StackId": "arn:aws:cloudformation:ap-southeast-2:899051371423:stack/DockerTutorial/7adc6f40-99d5-11e7-a360-50fa575f6862",
            "Description": "Docker CE for AWS 17.06.1-ce (17.06.1-ce-aws1)",
            ...snip...
            "StackStatus": "CREATE_IN_PROGRESS",
            ...snip...

        }
    ]
}
```
<div class="info-box information">
When the stack is ready, the <code>StackStatus</code> will be <code>CREATE_COMPLETE</code>.
If the stack creation fails, the <code>StackStatus</code> will be <code>ROLLBACK_COMPLETE</code>.
</div>

<div class="info-box notice">
<strong>For the demo</strong>: Open AWS Console in a tab to show ELB, instances, etc.
</div>

<h2 class="info-box exercise">Exercise: Setup the local environment to talk to a Swarm Manager</h2>

Find a Docker Swarm Manager instance public IP:

```sh
./get_manager_ips.sh

# ==>
54.79.9.120
```

First, add your SSH keypair PEM file to your SSH agent or keychain:

```sh
# Use the path to your SSH keypair PEM file location:
ssh-add ~/.ssh/DockerTutorial.pem
```

You can SSH directly to the Docker manager like so:

```sh
ssh -A docker@54.79.9.120
docker ps
exit
```

But since we want to use scripts and config files locally, we will open an SSH tunnel instead.

Open an ssh tunnel to the manager so you can use `docker` commands directly with the Docker for AWS swarm.

```sh
./ssh_tunnel.sh 54.79.9.120

# ==>
Type: export DOCKER_HOST=localhost:2374
```

Set the `DOCKER_HOST` environment variable and test connection to the Docker swarm manager:

```sh
export DOCKER_HOST=localhost:2374
docker info

# ==>
...snip...
Server Version: 17.06.1-ce
...snip...
Labels:
 os=linux
 region=ap-southeast-2
 availability_zone=ap-southeast-2b
 instance_type=t2.small
 node_type=manager
...snip...
```

<div class="info-box information">
The Labels can be used with placement constraints or <a href="https://docs.docker.com/engine/swarm/services/#specify-service-placement-preferences-placement-pref" target="_blank">preferences</a> (e.g. to evenly distribute services across AZs, not just nodes), e.g.
<br/>
<pre class="sourceCode">
docker service create \
  --replicas 9 \
  --name redis_2 \
  --placement-pref 'spread=node.labels.availability_zone' \
  redis:3.0.6</pre></div>

<h2 class="info-box exercise">Exercise: Verify your swarm nodes are up and running</h2>

```sh
docker node ls

# ==>
ID                            HOSTNAME                                           STATUS              AVAILABILITY        MANAGER STATUS
6r7o5rv4x3pv3757ik2y5jm6j *   ip-172-31-21-75.ap-southeast-2.compute.internal    Ready               Active              Leader
a3o4uvlpdtrbcpxwu9hou4fsf     ip-172-31-10-93.ap-southeast-2.compute.internal    Ready               Active
i75ybtjkpdfq2dm8v0lojj7mo     ip-172-31-38-221.ap-southeast-2.compute.internal   Ready               Active
q3dmtefzvtd075rsfdgl8dxxp     ip-172-31-25-160.ap-southeast-2.compute.internal   Ready               Active
```

<h2 class="info-box exercise optional">(Optional) Exercise: Running the app locally</h2>

If you have Docker setup on your local machine (e.g. for Mac or Docker for Windows) then you can test out the app locally using `docker-compose`:

```sh
unset DOCKER_HOST
cd voting-app
docker-compose -f ./docker-stack.yml up -d
# This may take a couple of minutes...
docker-compose ps

# ==>
       Name                     Command               State            Ports
-------------------------------------------------------------------------------------
votingapp_db_1       docker-entrypoint.sh postgres    Up      5432/tcp
votingapp_redis_1    docker-entrypoint.sh redis ...   Up      0.0.0.0:32768->6379/tcp
votingapp_result_1   node server.js                   Up      0.0.0.0:5001->80/tcp
votingapp_vote_1     gunicorn app:app -b 0.0.0. ...   Up      0.0.0.0:5000->80/tcp
votingapp_worker_1   /bin/sh -c dotnet src/Work ...   Up
```

Browse to [`http://localhost:5000`](http://localhost:5000) for the Vote interface.
Browse to [`http://localhost:5001`](http://localhost:5001) for the Result interface.

Check the Swarm Visualiser - nothing?  Why?

Stop and remove the local application:

```sh
docker-compose down
```

Deploy the same Docker Compose project as a "stack" to the local host:

```sh
# Confirm the local instance is a single swarm node:
docker node ls
# If not a swarm, use `swarm init` first.
```

Deploy the stack:

```sh
docker stack deploy votingapp -c ./docker-stack.yml
docker stack ls
```

Now, check the Swarm Visualiser.

Remove the local stack:

```sh
docker stack rm votingapp
```

Reset the local env to point back to the Docker for AWS manager:

```sh
cd ..
export DOCKER_HOST=localhost:2374
docker node ls
```

<h2 class="info-box exercise">Exercise: Deploy voting app to Docker for AWS</h2>

Use `docker stack deploy` with the Docker Compose V3 format YAML file to deploy the app onto the swarm:

```sh
docker stack deploy votingapp --compose-file voting-app/docker-stack.yml

# ==>
Creating network votingapp_backend
Creating network votingapp_frontend
Creating network votingapp_default
Creating service votingapp_db
Creating service votingapp_vote
Creating service votingapp_result
Creating service votingapp_worker
Creating service votingapp_visualizer
Creating service votingapp_redis
```

```sh
docker stack ls

# ==>
NAME                SERVICES
votingapp           6
```

```sh
docker service ls

# ==>
ID                  NAME                   MODE                REPLICAS            IMAGE                                          PORTS
f0t0bqff6opc        votingapp_redis        replicated          1/1                 redis:alpine                                   *:0->6379/tcp
ge4vah4nc0mp        votingapp_db           replicated          1/1                 postgres:9.4
ndm6ytoqvaz4        votingapp_visualizer   replicated          1/1                 dockersamples/visualizer:stable                *:8080->8080/tcp
uwk4s3y27pud        votingapp_worker       replicated          1/1                 dockersamples/examplevotingapp_worker:latest
vbhjil2sk8z1        votingapp_result       replicated          1/1                 dockersamples/examplevotingapp_result:before   *:5001->80/tcp
w5rt9x4bt6sw        votingapp_vote         replicated          2/2                 dockersamples/examplevotingapp_vote:before     *:5000->80/tcp

# You can also see the tasks for a service:
docker service ps votingapp_vote
```

To access the voting app that is deployed to the swarm, you will need the ELB name:

```sh
./get_elb_names.sh

# ==>
"DNSName": "DockerTut-External-P2N78KBI3BAI-1241970304.ap-southeast-2.elb.amazonaws.com",
```

You can now access each service which have published ports at:

`http://DockerTut-External-P2N78KBI3BAI-1241970304.ap-southeast-2.elb.amazonaws.com:<published_port>`

Try accessing the exposed services:

* Vote : `http://<elb_dns_name>:5000`
* Result : `http://<elb_dns_name>:5001`
* Visualizer : `http://<elb_dns_name>:8080`

<div class="info-box information">
<strong>Note</strong>: These are exposed as HTTP by default since the ELB is not terminating TLS.  It is acting as a Layer 4 LB and reverse proxy.  This feature is setup automatically as part of Docker for AWS.
</div>

Later we will expose services using TLS termination on the ELB.

<h2 class="info-box exercise">Exercise: Check the Swarm Visualiser (via ELB DNS NAME)</h2>

Browse to the Swarm Visualizer : `http://<elb_dns_name>:8080`

Undeploy the stack so we can use a custom reverse proxy as the entrypoint to the swarm:

```sh
docker stack rm votingapp
```

<h2 class="info-box exercise">Exercise: Deploy Traefik reverse proxy</h2>

The Docker for AWS setup currently only creates an ELB (Layer 4) not an ALB (Layer 7).  This means we have to use a custom Layer 7 reverse proxy to route to our services in the swarm over a single port (443 or 80).

We will use a Traefik service with a published port on port 443 to handle internal routing on the swarm ingress load balancer (routing mesh).

### With a custom domain name, ACM certificate and Route 53 DNS setup

We will use a custom domain name and ACM wildcard certificate to get TLS termination working on the ELB and then use Traefik to route to services internally in the swarm via a host header.

#### Prerequisites

You will need to have registered a custom domain name (e.g. `mydomain.com`)

In Route 53, create a Hosted Zone for your domain.

In your DNS registrar, point it to at least two of the DNS servers assigned by AWS (in your NS record).

Create a host wildcard record in your hosted zone (A record of type alias):

```
        Name: *.mydomain.com
        Type: A - IPv4 address
       Alias: Yes
Alias Target: <Select your ELB DNS NAme from the list>
```

Leave the remaining fields as is.
Create the record.

Go to Amazon Certificate Manager and create a free wildcard certificate for your domain: `*.mydomain.com`

Approve the certificate request in your email.

Record the ARN for the ACM certificate, e.g: `arn:aws:acm:ap-southeast-2:XXXXXXXXXXXX:certificate/e807ff24-1988-428e-a326-8ed928d535b6`

This will be needed for Traefik later.

Copy the file `prod.env.template` to `prod.env` and update the environment variables with your domain name and ACM certificate ARN:

```sh
DOMAIN_NAME=<your_domain>
ACM_CERT_ARN=<your_acm_cert_arn>
```

#### Start exercise

Create the overlay network for Traefik and other services to communicate within the swarm:

```sh
docker network create -d overlay traefik
```

Deploy the Traefik stack to the swarm:

```sh
./deploy_stack.sh voting-app/docker-stack-traefik.yml prod.env
```

Browse to the Traefik dashboard: http://traefik.dockertutorial.technology:8000/
(No services displayed yet.)

Deploy the Voting App stack to the swarm:

```sh
./deploy_stack.sh voting-app/docker-stack-votingapp.yml prod.env
```

Let's check our stacks:

```sh
docker stack ls

# ==>
NAME                SERVICES
traefik             1
votingapp           5
```

Thanks to Traefik, we can now browse to the vote and result apps:

Browse to: https://vote.dockertutorial.technology
Browse to: https://result.dockertutorial.technology

TLS is [setup automatically for us in the ELB](https://docs.docker.com/docker-for-aws/load-balancer/) by the `com.docker.aws.lb.arn` label:

```yaml
traefik:
  deploy:
    ...snip...
    labels:
      ...snip...
      - "com.docker.aws.lb.arn=${ACM_CERT_ARN}"
```

### Alternative: no custom domain or TLS certificate

There will be no TLS termination on the ELB using this method (Traefik does supports TLS itself but that is beyond the scope of this tutorial).

Your options here are:

1. Don't use Traefik at all - just add published ports to the services you want to expose via the ELB and use the ELB DNS Name to access them
2. Use Traefik with [Path Routing](https://docs.traefik.io/basics/) instead of Host Header routing, e.g. `traefik.frontend.rule=Path:/vote`

Through this tutorial, when you see a URL like: `https://vote.dockertutorial.technology`, change it to `http(s)://<ELB_DNS_Name>(:port|/path)` depending on your configuration choice above.

Copy the file `prod.env.template` to `prod.env` and update the environment variables with empty values (since they won't be required):

```sh
DOMAIN_NAME=
ACM_CERT_ARN=
```

<h2 class="info-box exercise">Exercise: Deploy the Swarm Visualizer (using Traefik reverse proxy)</h2>

```sh
./deploy_stack.sh voting-app/docker-stack-visualizer.yml prod.env
```

Browse to: https://visualizer.dockertutorial.technology

<h2 class="info-box exercise">Exercise: Get audience to vote via the custom domain</h2>

Which the most popular pet?  Cats or Dogs?

Browse to: https://vote.dockertutorial.technology on your mobiles.
Check results at: https://result.dockertutorial.technology

<h2 class="info-box exercise">Exercise: Deploy Portainer</h2>

```sh
./deploy_stack.sh voting-app/docker-stack-portainer.yml prod.env
```

Browse to: https://portainer.dockertutorial.technology

**Note**: Normally, you would use a [persistent volume for Portainer](https://portainer.readthedocs.io/en/stable/deployment.html#persist-portainer-data) to retain state.  In this demo we will not use a volume.

Scale up the vote service to 2 containers then try voting and see that different containers process the vote.

In Portainer, check the logs (e.g. traefik) and console (e.g. db) for a running container.

**Note:** Exposing Portainer is a security risk - do not expose it publicly!

<h2 class="info-box exercise">Exercise: Deploy Logging</h2>

Containers can use one of [several drivers](https://docs.docker.com/engine/admin/logging/overview/) to ship logs.

`docker logs` / `docker-compose logs` / `docker service logs` only work with the default `json-file` logging driver.

The containers in the Docker for AWS cluster are already configured to send all their logs to AWS CloudWatch Logs.  This is not the easily interface to use in the AWS Console.  You can ship elsewhere from here.  For the demo, we'll use EFK stack to ship to a private Elasticsearch server.

<div class="info-box information">
<strong>Note</strong>: In production, you’ll need to consider HA, scaling, data backup and retention, security, disaster recovery, etc.
<br/><br/>
<strong>Note</strong>: Fluentd is exposed publicly due to limitations of the Docker for AWS setup.  You can create additional security groups or setup NACLs to block access.  In Docker EE you can configure UCP logging to ship logs to Elasticsearch.
</div>

#### Start exercise

Create the logging network:

```sh
docker network create -d overlay logging
```

Deploy the logging stack:

```sh
./deploy_stack.sh voting-app/docker-stack-logging.yml prod.env
```

Check that the logging stack and services are up and running:

```sh
docker stack ls
docker service ls --filter name=logging
```

Check the Kibana dashboard:

Browse to: https://kibana.dockertutorial.technology

No logs?

Update the `votingapp` stack to connect to the `logging` network and add the fluentd logging options, then re-deploy.

```sh
./deploy_stack.sh voting-app/docker-stack-votingapp.yml prod.env
```

Only services that have changed will be restarted.

In Kibana, click <strong>refresh fields</strong> to configure the index pattern then click Create.

Go to the <strong>Discover</strong> view - logs should now appear.

Select the container_name, log, and message fields.
Auto-refresh every 10 secs.

Try voting a few times then check the logs (try query: `Processing vote for '?'`)

<h2 class="info-box exercise optional">(Optional) Exercise: Create a visualisation in Kibana</h2>

Create a visualisation for a tally of all votes for Cats and Dogs:

* Click **Visualize**
* Click **Create Visualization**
* Select `Metric` as the visualization type
* Select the `logstash-*` index
* Select `count` for Metric
* Select `Filters` under Split group / Aggregation
* Filter 1: "Processing vote for 'a'"
* Label 1: Cats
* Click **Add Filter**
* Filter 2: "Processing vote for 'b'"
* Label 2: Dogs
* Click **Save**
* Enter "Vote Tally"

<div class="info-box notice">
<strong>Note</strong>: Make sure no filter is added to the query at the top.
</div>

Create a dashboard which includes the "Vote Tally" visualisation:

* Click **Dashboard**
* Click **Create a dashboard**
* Click **Add**
* Select 'Vote Tally'
* Click **Save**
* Maximise the dashboard

Try voting a few times then check the dashboard.

Any data that is logged can be searched and made into a dashboard for key performance or business metrics.

<h2 class="info-box exercise optional">(Optional) Exercise: Volumes - Handling application state in the cluster</h2>

In a cluster, application state should not be kept inside containers.  However, if we use host volumes, then the data is not portable -- we need to use constraints on services to limit where they can run which is not very practical.

Various Docker Volume plugins exist to help solve this problem.  With Docker for AWS, the [Cloudstor plugin](https://docs.docker.com/docker-for-aws/persistent-data-volumes) is setup which supports EBS and EFS based backing stores.  This driver enables data to travel with the service tasks (containers) even when they get rescheduled.

### Without portable volumes

If we move the DB container to another host then we'll loose state in the DB.

Check the Results: https://result.dockertutorial.technology/
There should be some votes.

Check the Visualizer: https://visualizer.dockertutorial.technology/
The voting app database should be running on the manager node.

Update the `db` service contraint in `voting-app/docker-stack-votingapp.yml` to be:

`constraints: [node.role != manager]`

Re-deploy the votingapp:

```sh
./deploy_stack.sh voting-app/docker-stack-votingapp.yml prod.env
```

Make sure db has moved to a different host.

Check the Results: https://result.dockertutorial.technology/
There should no votes -- the state was lost since the host volume was left on the manager node and a new one was created.

If result is now showing anything (not votes in the bottom right corner), you might need to scale result to 0 then back to 1:

```sh
docker service scale votingapp_result=0
# wait a bit...
docker service scale votingapp_result=1
```

### With Volumes (using Cloudstor volume plugin)

Let's create a portal volume using EFS and the CloudStor volume plugin.

```sh
docker volume create -d "cloudstor:aws" --opt backing=shared db-votes
```

```sh
docker volume ls
DRIVER              VOLUME NAME
...snip...
cloudstor:aws       db-votes
...snip...
```

This will use EFS (for DBs with a lot of writes, you should use EBS but moving EBS volumes between hosts is a slower process so for the demo we use EFS).

Update the voting app stack to use the external `db-votes` volume:

```yaml
volumes:
  db-data:
  db-votes:
    external: true
```

Update the `db` service to use the new volume:

```yaml
db:
  image: postgres:9.4
  volumes:
    - db-votes:/var/lib/postgresql/data
```

You can remove the placement constraint too:

```yaml
# deploy:
  # placement:
    # constraints: [node.role != manager]
```

Re-deploy the voting app:

```sh
./deploy_stack.sh voting-app/docker-stack-votingapp.yml prod.env
```

Create some votes: https://vote.dockertutorial.technology/

Check the Results: https://result.dockertutorial.technology/

If you need to move the `db` service (without draining a whole node) try:

```sh
docker service scale votingapp_db=0
# wait a bit...
docker service scale votingapp_db=1
```

<h2 class="info-box exercise">Exercise: Scaling up / down services</h2>

Scale up some of the services and check Kibana logs:

```sh
docker service scale votingapp_worker=3
docker service scale votingapp_vote=3
```

Reload the vote page: https://vote.dockertutorial.technology/

Try voting a few times - notice that a different container processes the vote each time (this is load-balancing provided by swarm mode).

<h2 class="info-box exercise">Exercise: Drain node</h2>

Pick a node (not master) and not the one with the `db` on it.

Use `docker node ls` to find the right one.

Set it to `drain` to move containers:

```sh
docker node update --availability=drain a3o4uvlpdtrbcpxwu9hou4fsf
```

Check the Visualizer: https://visualizer.dockertutorial.technology/

All containers have moved off this node.  Now you can perform any maintenance or leave it as is (common for manager nodes).

Let's set it back to `active`:

```sh
docker node update --availability=active a3o4uvlpdtrbcpxwu9hou4fsf
```

<h2 class="info-box exercise">Exercise: Swarm Service Rolling updates</h2>

When deploying updates to services (e.g. new image or config/secrets/env vars), swarm uses a rolling update strategy to achieve zero-downtime deployment (you can combine this with built-in [healthchecks](https://docs.docker.com/compose/compose-file/#healthcheck)).

See the Compose file:

```
deploy:
  replicas: 1
  update_config:
    parallelism: 2
    delay: 10s
```

This means 2 containers will be updated at a time with a 10 sec delay between groups of containers.  you can also set a `failure_action` (e.g. to `rollback` or `continue` or `pause`).

Let's make a config change using environment variables and roll out the change to the `vote` instances.

In the `vote` service, add:

```yaml
environment:
  OPTION_A: Pigs
  OPTION_B: Hamsters
```

Also set replicas to `3` for the `vote` service:

```yaml
deploy:
  replicas: 3
```

Deploy the `votingapp` stack again to perform a rolling update:

```sh
./deploy_stack.sh voting-app/docker-stack-votingapp.yml prod.env
```

And watch the Visualizer: https://visualizer.dockertutorial.technology/

<h2 class="info-box exercise">Exercise: Deploy Monitoring</h2>

For this demo we'll deploy a monitoring stack consisting off: cAdvisor (host/container meterics) + Prometheus (server + alarm manager) + Grafana (dashboard)

Alternatives:

* Sysdig, Datadog, CloudWatch, APM tools (Dynatrace, AppDynamics, etc.)

UCP provides some monitoring too: https://docs.docker.com/datacenter/ucp/1.1/monitor/monitor-ucp/

We'll use this setup as it works: https://github.com/bvis/docker-prometheus-swarm and its dashboard: https://grafana.com/dashboards/609

### Create config for Prometheus

```sh
docker config create prometheus.yml docker-prometheus-swarm/rootfs/etc/prometheus_custom/prometheus.yml
docker config create alert.rules_services docker-prometheus-swarm/rootfs/etc/prometheus_custom/alert.rules_services

# Check config exists
docker config ls
```

**Note**: Swarm supports secrets too, there are encrypted.

### Deploy the monitoring stack

```sh
./deploy_stack.sh voting-app/docker-stack-monitoring.yml prod.env
```

Check the monitoring services:

Browse to: https://alertmanager.dockertutorial.technology
Browse to: https://prometheus.dockertutorial.technology
Browse to: https://alertmanager.dockertutorial.technology
Browse to: https://grafana.dockertutorial.technology

Import the JSON dashboard: `docker-prometheus-swarm/dashboards/docker-swarm-container-overview_rev23_no_logging.json`

Inspect some of the metrics available.

<h2 class="info-box exercise">Exercise: Teardown</h2>

```sh
docker stack rm logging monitoring portainer traefik votingapp visualizer
docker network prune -f
docker volume prune -f
docker container prune -f
# If you want to remove all unused images and reclaim space
#docker image prune -f
docker config rm prometheus.yml alert.rules_services
```

<h2 class="info-box exercise">Delete the CloudFormation stack</h2>

Clean up all AWS resources.

```sh
./delete_stack.sh
```

<h2 class="info-box exercise">Credits</h2>

The voting app was copied from Docker Samples:
* https://github.com/dockersamples/example-voting-app

Monitoring dashboard and Docker Compose file - [Basilio Vera](https://github.com/bvis):

* https://github.com/bvis/docker-prometheus-swarm

<h2 class="info-box exercise">License</h2>

<pre>
Copyright (c) 2017 Elabor8 Pty Ltd

MIT License

Permission is hereby granted, free of charge, to any person obtaining
a copy of this software and associated documentation files (the
"Software"), to deal in the Software without restriction, including
without limitation the rights to use, copy, modify, merge, publish,
distribute, sublicense, and/or sell copies of the Software, and to
permit persons to whom the Software is furnished to do so, subject to
the following conditions:

The above copyright notice and this permission notice shall be
included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND
NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE
LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION
OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION
WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
</pre>
