# Troubleshooting 502 and 503

503 is no problem, it is only internally used by the readiness_check to signal the service is starting and not yet ready.

502 is caused by a request time out and is reported to the user. The application server is not able process the request in time and the user receives an error. 


## readiness_check
["If you do not specify a liveness check path or a readiness check path, by default split health checks only confirm that the VM instance and the Docker container are running."](https://cloud.google.com/appengine/docs/flexible/nodejs/migrating-to-split-health-checks)

Health checks are not configured for this app, so google thinks the service is ready to receive traffic before it actually is. Before the server is ready, it is unresponsive and the user will eventually receive a 502 when it times out.

[An example of readiness_check before the server is actually ready.](readiness-check-200-before-actually-ready.PNG).

Correctly configuring health checks will prevent the server from receiving traffic while unresponsive, and in turn fix the 502 problem.


## Development server
The app is currently deployed as a NodeJS application. Google runs "yarn start" automatically when it runs, but this starts the development server. This server is heavy on cpu and ram as it compiles the resources at runtime, does not compress the files and is a lot less performant than a proper production web server.

It takes 20 minutes for the development server to start, according to the logs, and during this time it is completely unresponsive. 
[An example of 502 while the server is starting](request-during-startup.PNG) and [an example of the 20 minute startup time](startup-time.PNG).

On May 27 the cpu and memory of the virtual machines were increased, [link to commit](https://github.com/Opplysningen1881/voice-app/commit/0afa3f8cdfe9e3cf9980355f0f818a85c68dda15). There has been no reported 502 since this time, so the higher specs likely enabled the server to start fast enough for users not to trigger a 502. Note it is still possible, just less likely than before.


## Custom runtime

The development server needs to be replaced with a production server. Since the artifacts are compiled html, js, etc. there is no need for a NodeJS runtime. I recommend changing the app engine runtime from "nodejs" to "custom" and provide a docker image that uses Nginx to serve the production files. Nginx is highly performant and widely used and a safe choice.



# Other considerations outside of the problem scope

### Slow deploy

The app is deployed with `gcloud app deploy` which uploads everything in the directory to google cloud to then be built. This includes `node_modules` which is 250 mb, and part of the reason why a deploy takes 20 min to several hours according to the readme. Adding a `.gcloudignore` file and putting `node_modules` in it would speed up the deploy process significantly. After this change a deploy takes 9 minutes my computer.

## Pipeline

Currently the developer checks out a branch on their machine and runs 'gcloud app deploy' to push to the test or production environment. This uploads whatever is in the developers directory and that might not be the same as what's in version control. Imagine accidentally uploading an unfinished feature branch to production for example. There is also no clear code-review -> test-environment -> prod-environment flow as anything can be uploaded anywhere at any time.

The alternative is to trigger builds whenever something changes in git, and automatically deploy to the respective environments. Dev branch to test environment and master to prod for example. This makes the build process more consistent and predictable.

## Traffic

What amount of trafic is expected? The current setup is limited to 2 instances. Will this always be sufficient? Consider configuring for automatic scaling.

## 1881searchapp
The app is fetched from /1881searchapp when used on the website. I don't know why this url is used, from what I can see it is not significant and returns the same as /jklasdfjkl would.

## node_modules
The node_modules folder is version controlled. This folder is 250 mb and compressed via git. It takes a really long time to clone the repo, and this would slow down any pipeline that has to build from the source code. Should consider rebasing it away or just creating a new git repo entirely and starting over.
