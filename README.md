# Orchestrator

Central command for the paz platform, providing cluster management as well as health and stats at the cluster, machine and service level. 

API documentation can be generated by running the following command:

```
$ npm run docs
```

And these docs can be served locally by running:
```
$ npm run docs-server
```

...and going to `http://localhost:9002` in the browser to see them.

The API documentation is generated from `docs/api-blueprint.md`.

## Developing against the Orchestrator

### Using `fig`

Ensure you have fig installed (http://www.fig.sh/install.html). And boot2docker if you're on a mac (http://docs.docker.com/installation/mac/). Install `etcdctl` and `fleetctl` using brew (we won't use the latter now but you'll need it).

Check if boot2docker is running:
```
$ boot2docker status
```

If it's not up, bring it up:
```
$ boot2docker up
```

It should stay up until you poweroff. Sometimes you'll lose it after a sleep. If fig gives you errors about "Connection to 192.168.59.103 timed out" then you know boot2docker is down.

For each new terminal session type the following (so fig and Docker know where to find boot2docker):
```
$ export DOCKER_HOST=tcp://192.168.59.103:2375
```

If you forget this, you'll get the following error message:
```
2014/09/02 17:15:06 Get http:///var/run/docker.sock/v1.13/containers/json: dial unix /var/run/docker.sock: no such file or directory
```

Now to bring up the Orchestrator so you can test against it, grab the repo from Github, `cd` to root dir, and run:
```
$ fig up -d
Recreating orchestrator_etcd_1...
Recreating orchestrator_svcdir_1...
Recreating orchestrator_scheduler_1...
Recreating orchestrator_orchestrator_1...
```

The `-d` means "run in the background" so you get your cmd-prompt back. As you can see, it has brought up all its dependencies. To check if it's running, and what ports it's bound to:
```
$ fig ps
           Name                 Command      State               Ports
-----------------------------------------------------------------------------------
orchestrator_etcd_1           /bin/bash      Up      7001->7001/tcp, 4001->4001/tcp
orchestrator_svcdir_1         ./bin/server   Up      9001->9001/tcp
orchestrator_scheduler_1      ./bin/server   Up      9002->9002/tcp
orchestrator_orchestrator_1   ./bin/server   Up      9000->9000/tcp
```

This is telling up that these services are "Up" and their ports.

Here is a little script to set up some useful environment variables to aid some tasks with curl I'll introduce later:
```
$ cat > paz-envvars.sh
#!/bin/bash
export ORCHESTRATOR_URL=192.168.59.103:9000
export SCHEDULER_URL=192.168.59.103:9002
export SVCDOC='{"name":"demo-api","description":"Very simple HTTP Hello World server","dockerRepository":"lukebond/demo-api","loadBalanced":false,"publicFacing":false}'
export DEPLOY_DOC="{\"serviceName\":\"demo-api\",\"dockerRepository\":\"lukebond/demo-api\",\"pushedAt\":`date +%s`}"
^D
$ chmod +x paz-envvars.sh
```

The above is just creating a text file. If it doesn't work just use vim or whatever you like to create the file.

Run the above script with this special syntax so that it exports the env vars:
```
$ . paz-envvars.sh
```

To ensure the export worked, try:
```
$ echo $ORCHESTRATOR_URL
172.17.8.101:49164
```

If you get any errors, ensure `fig ps` above is working as per the example.

To add a service via the orchestrator, and test that it worked:
```
$ curl -i -XPOST -H "Content-Type: application/json" -d "$SVCDOC" $ORCHESTRATOR_URL/services
$ curl -i $ORCHESTRATOR_URL/services
```
Look out for non-20X response codes.

To delete a service:
```
$ curl -i -XDELETE $ORCHESTRATOR_URL/services/demo-api
```

To patch a service (in this case to set an environment var- demo-api can optionally use a "MESSAGE" envvar):
```
$ curl -i -XPATCH -H "Content-Type: application/json" -d '{"env": {"MESSAGE": "so platform; many service; wow"}}' $ORCHESTRATOR_URL/services/demo-api/config/next
```

Now to deploy that service via the scheduler:
```
$ curl -i -XPOST -H "Content-Type: application/json" -d "$DEPLOY_DOC" $SCHEDULER_URL/hooks/deploy
```

Refer to the Orchestrator API docs for all the things you can do but this should get you started.

Now let's try running a service on the platform. We'll use a basic "hello world" web server. On your local machine, create a file called `contrived-service-1.json` with the following contents:
```
{
  "name": "contrived-service-1",
  "description": "Test service",
  "dockerImage": "lukebond/contrived-service-1",
  "ports": [
    {
      "container": 9000,
      "host": 80
    }
  ],
  "autoDeploy": "always",
  "numInstances": 1,
  "conflictsWith": "contrived-service-*"
}
```

Find out which host and port the service-directory is running on:
```
$ etcdctl --peers=172.17.8.101:4001 get /paz/services/paz-service-directory
172.17.8.103:49153
```

Now post the above JSON file to the service directory API:
```
$ curl -XPOST -H "Content-Type: application/json" -d @contrived-service-1.json 172.17.8.103:49153/services
{"meta":{"statusCode":201,"ok":true,"uuid":"8n373x","name":"contrived-service-1"}}
```

Now get the host and port of the scheduler:
```
$ etcdctl --peers=172.17.8.101:4001 get /paz/services/paz-scheduler
172.17.8.103:49154
```

Now post to the scheduler to trigger the deployment of this service:
```
$ curl -XPOST -H "Content-Type: application/json" -d '{"serviceName": "contrived-service-1", "dockerRepository": "lukebond/contrived-service-1:0.0.1", "pushedAt": 0}' 172.17.8.103:49154/hooks/deploy
{"statusCode":200}
```

Watch the logs of this service and wait until you see that it is up (it's a Harp web server):
```
$ fleetctl journal -f contrived-service-1-1.service
-- Logs begin at Tue 2014-07-15 10:30:37 UTC. --
Jul 15 12:00:15 core-03 systemd[1]: Starting Test service...
Jul 15 12:00:15 core-03 etcdctl[5131]: running
Jul 15 12:00:15 core-03 docker[5130]: Unable to find image 'lukebond/contrived-service-1' locally
Jul 15 12:00:15 core-03 systemd[1]: Started Test service.
Jul 15 12:00:15 core-03 docker[5130]: Pulling repository lukebond/contrived-service-1
Jul 15 12:01:59 core-03 docker[5130]: ------------
Jul 15 12:01:59 core-03 docker[5130]: Harp v0.12.1 – Chloi Inc. 2012–2014
Jul 15 12:01:59 core-03 docker[5130]: Your server is listening at http://localhost:9000/
Jul 15 12:01:59 core-03 docker[5130]: Press Ctl+C to stop the server
Jul 15 12:01:59 core-03 docker[5130]: ------------
```

Hit up port 80 on the box containing this new service to see if it works.
```
$ fleetctl list-units
UNIT                                    MACHINE                   ACTIVE  SUB
paz-orchestrator-announce.service       7c83517a.../172.17.8.101  active  running
paz-orchestrator.service                7c83517a.../172.17.8.101  active  running
paz-scheduler-announce.service          83fe3a48.../172.17.8.102  active  running
paz-scheduler.service                   83fe3a48.../172.17.8.102  active  running
paz-service-directory-announce.service  5c3e57b1.../172.17.8.103  active  running
paz-service-directory.service           5c3e57b1.../172.17.8.103  active  running
paz-web-announce.service                5c3e57b1.../172.17.8.103  active  running
paz-web.service                         5c3e57b1.../172.17.8.103  active  running
contrived-service-1-1.service           5c3e57b1.../172.17.8.103  active  running
```

As you can see here, it's on 172.17.8.103, so paste that into your browser and you should see "Hello World 1".