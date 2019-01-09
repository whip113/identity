# Conjur Master Build on Ubuntu 16.04
# Pre-Req's
Ubuntu 16.04 w/ 2cpu and 4GB of memory
Docker CE
Conjur v5 EE appliance

You can use other distros, but the instructions here assume Ubuntu 16.04 LTS. 
These instructions should also work for 17.04 and 18.04 as well (or any Debian distro for that matter)
You could also use the OSS version of Conjur with these instructions, but the instructions here assume v5 EE.

First, we'll get some pre-req's knocked out. We won't use all of these today, but I've found them to be helpful.
Install pre-req packages used later
`$ sudo apt-get install -y curl \
    apt-transport-https \
    python-pip \
    ca-certificates \
    software-properties-common \
    openssh-server \
    git
`    
Now we'll prep for the Docker install. If you skip the next 2 steps, you'll install the wrong version of Docker. 
These steps are distro specific, so check out the Docker documentation if you're using a different distro.
# Add Docker Key
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

# Add Docker Repository
$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

# Update `apt`
$ sudo apt-get update

#Install Docker
$ sudo apt-get install -y docker-ce

# Lets use PIP to install Docker-Compose while we're at it
$ pip install -U docker-compose

# Create the `docker` group. The Docker install usually does this, but sometimes it doesn't.
$ sudo groupadd docker

# Add our user ID to the `docker` group so we don't have to prefix Docker commands with SUDO.
$ sudo usermod -aG docker $USER

# Refresh the groups. A logoff/logon would also work.
$ newgrp docker

# Test that docker is working and our user ID is in the `docker` group 
# If you get a weird deamon error then you probably don't exist in the `docker` group or forgot to refresh the groups
$ docker run hello-world

### OK, great, we have  Docker installed, making our VM our "Docker host". That's it for pre-requisites to install Conjur!
### Now we need to get the Docker image for the Conjur appliance copied over to our Docker host
### Go ahead and use WinSCP if you want, or if you're doing this from an SSH session already you can just SCP the file over
$ scp filename username@host:/tmp

# Load the appliance image into Docker
$ docker load -i conjur-appliance-5.1.2.tar.gz
$ docker load -i conjur_cli.tar.gz

# Verify the images are loaded and grab the name and tag of the image you want to launch
$ docker images

# Launch the Conjur appliance and expose the necessary ports
$ docker run --name conjur-master -d --restart=always --security-opt \
      seccomp:unconfined -p "443:443" -p "636:636" -p "5432:5432" -p "1999:1999" conjur-appliance_tag

# Confirm it is up and running
$ docker ps

# View a bunch of details about the container
$ docker inspect conjur-master

# `docker exec` into the  container
$ docker exec -it conjur-master bash

# Use `evoke` to configure the master
$ evoke configure master -h master.nate.lab -p Cyberark1 CYBR

# Note: If you use a name other than the docker host's hostname, you'll need to edit /etc/hosts so that the name you use resolves to
# the IP of your docker instance as shown in the `docker inspect conjur-master` commmand.

# Check the /health and /info routes from your docker host
$ curl -k https://master.nate.lab/health
$ curl -k https://master.nate.lab/info

# Let's make a working directory on the Docker host. We'll volume mount this into  the CLI container so our files persist.
$ mkdir ~/Documents/conjur

# Launch the CLI container with the working directory mounted. 
# Note, we're pulling this from a public GitHub project, so your docker host will need internet access to do this the first time.
$ docker run -v ~/Documents/conjur:/root --rm -it cyberark/conjur-cli:5

# Update hosts file in the container (if necessary)
$ vi /etc/hosts

# Initialize the CLI
$ conjur init -a CYBR -u https://master.nate.lab

# Login as `admin` using the CLI
$ conjur authn login -u admin -p Cyberark1

# Now create two policy files in your working directory
$ touch /root/users.yml && touch /root/variable.yml

# Check out https://cyberark.github.io/conjur-policy-generator/# for help creating policy files.
# For now, paste into users.yml the following

---
- !user alice
- !user bob
- !group secrets-users
- !group secrets-managers
- !grant
  role: !group secrets-users
  member: !user alice
- !grant
  role: !group secrets-managers
  member: !user bob
  
# Paste into variable.yml. Note that we're building a policy tree here. Root or `/` is the trunk of the policy tree
# and is where our users and groups are created. When we create the variable with a nested policy like this
# we need to be sure we reference the correct location for the groups. If we used secrets-users instead of /secrets-users
# we would need to have a group myapp/alfa/secrets-users instead of just secrets-users.

---
- !policy
  id: myapp
  body:
    - !policy
      id: alfa
      body:
        # Secret Declarations
        - &secrets_hydrogen
          - !variable hydrogen
        
        # User & Manager Groups
        - !permit
          role: !group /secrets-users
          privileges: [ read, execute ]
          resources: *secrets_hydrogen
        - !permit
          role: !group /secrets-managers
          privileges: [ read, execute, update ]
          resources: *secrets_hydrogen
    - !policy
      id: bravo
      body:
        # Secret Declarations
        - &secrets_lithium
          - !variable lithium
        
        # User & Manager Groups
        - !permit
          role: !group /secrets-users
          privileges: [ read, execute ]
          resources: *secrets_lithium
        - !permit
          role: !group /secrets-managers
          privileges: [ read, execute, update ]
          resources: *secrets_lithium
          
# Load the policies. Note the API keys for the users Alice and Bob.
$ conjur policy load root users.yml
$ conjur policy load root variable.yml
 
# Use the UI in a web browser, login with the CLI, or call the API directly and see the difference between what the user `alice` can do with the secrets vs. the user `bob`.
# To authenticate, use the API key generated when the user's were created as the password.
# Now, start experimenting!
