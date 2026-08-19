I had not worked with any of these technologies before (other than Docker). I started working on this from a Windows machine but quickly decided I am more comfortable working from Linux and switched. 

## Commands Run

Setup commands:
```
sudo apt install dotnet-sdk-8.0
sudo apt install gh
sudo apt-get install wget gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
```

Project setup commands:
```
mkdir ThreatLockerAssessment
cd ThreatLockerAssessment
dotnet new webapi -n OrderService --use-controllers false
```

## Q&A

Why do we care if Dockerfile runs as root?

    If a bad actor is able to get access into the container, running as root would allow them to run privilege escalation exploits like installing malware or stealing credentials.

What did you have to change to make APPUSER actually work?

    RUN adduser --disabled-password --gecos "" appuser creates the user while USER appuser tells the application what user to use at runtime.

Risk of docker run with no restart policy or health-based rollback:

    If the container is unhealthy, it will remain running even if the application is broken. If there is no restart policy, someone will have to manually intervene because default restart policy is "no". 

Smallest change:

        docker run -d --name orderservice-prod \
          --restart-policy=unless-stopped \
          -p 8080:8080 \
          -e APP_VERSION=$GITHUB_SHA \
          $REGISTRY/$IMAGE_NAME:$GITHUB_SHA

Format check, test, security scan parallel vs sequentially:

    I think I would run it parallel because these do not depend on one another's output. Parallel saves time but needs more VMs. Sequentially might save on resources.

Where do secrets live and blast radius:

    I used Settings > Secrets and Variables > Actions > Repository Secrets. In the event that somehow a secret value did leak into the pipeline, since it's a public repo, I suppose anyone could take a look at the logs. How severe of an impact it would leave depends on what the secret was and where the logs went.

Trivy gate gap and how to fix it:

    The pipeline only runs on push/pull so unless the pipeline is run again it won't catch it. There should probably be some sort of scheduled pipeline that regularly runs on a set interval. I've used Renovate.

3 replicas behind a load balancer -- smallest, realistic next step?

    ...is setting up a Kubernetes cluster considered a small step? Haha. Some sort of container orchestration system.

And what not to do with Dockerfile:

    Don't make the Dockerfile run the application 3 times.


## Anyting worth calling out:

1. Since I'm running it through Github Actions, the "docker stop" and "docker rm" command wasn't really necessary since each time it runs its on a new runner, but I kept "- run: docker rm orderservice-prod 2>/dev/null || true" in there anyway just for posterity.
1. In the Deploy step, I needed to authenticate to Docker again to pull the image so I added that at the beginning.
1. "curl --fail http://localhost:8080/version" was happening before the container was ready so I had to add a wait to actually get confirmation that it was able to get the sha from the env variable.