# continuwuity-docker
Setup for Continuwuity via docker with caddy



## Requirements:
Must have a domain name.
Must be able to configure CNAME, A, or AAAA records for your domain.
Ability to forward ports to to server, ports 80 and 443 required.
Docker, Docker-compose and sufficient permissions to use those tools.
Linux, may be compatible with windows but tested for linux.

Instructions were tested on an amazon EC2 t2.mini instance, 2vcpu 1gb memory, 10gb of volume, unsure the practical minimums but it will start up with those resources. The EC2 was using an ubuntu image, also tested on ubuntu on a real system.

## Process:
### Configuration and prep steps
1. Retrieve docker compose yml file. Either from continuwuity.org/deploying/docker.html#with-caddy-docker-proxy or using the one in the project file here. The official one has some strange configuration choices like a relative path for certificates which requires building the docker containers in a specific directory each time for consistent results. They also do not specify container names, defaults to docker_caddy_1 and docker_homeserver_1, the docker file here uses matrix-caddy and matrix-srv for container names.
2. Assign a dns record for the domain or subdomain for your matrix server. Update your nameservers CNAME/A/AAAA record accordingly to point at the system you are hosting matrix on. Wait usually 5 to 60 minutes depending on nameservers. Until an nslookup returns the correct information.
3. Update the docker compose yml file. If using the one attached check if you want to use those directories, /srv/continuwuity/ or change those lines. Global replace matrix.example.com with the FQDN you are going to host. 
4. Update port forwards on firewall, point 80 and 443 at the server.
5. Create file paths for the db and data directories configured in the yml. Check permissions should be accessable to docker group.

### Compose and load steps
1. Create caddy network, this is not done in the compose file. Then compose the file up. Use sudo if needed.
    docker network create caddy
    docker-compose -f matrix-better.yml up -d
2. Check matrix-caddy logs to look for acme cert success. If connections failing debug for external http calls reaching this system. The caddy server uses an HTTP-01 cert challenge, does not require a dns wildcard or any additional setup.
    docker container logs matrix-caddy
3. Once certs are successful use a federation checker, https://federationtester.mtrnord.blog/ . Expect it to prompt for allow access to local network devices, blocking this causes false cors errors that can make debugging hard.
3.1 If all green you are ready to create first user. Check matrix-srv log for the token for first user create, open a matrix client login to your matrix server subdomain name enter username and password next step prompts for token. After first user is created the token specified in the matrix-better.yml file is valid for more new users.
    docker container logs matrix-srv
4. Join the admin room of your server, check connectivity with '!admin debug ping matrix.org', if you get a result then server is ready. Note: This can take up to 10 minutes to fail if it is not working, there should be a result fairly quickly if not wait 10 minutes for the fail message. Error I had the most trouble with was dns resolution issues, I had to comment out the resolve.conf volume mount in the yml on ubuntu, try commenting it in if dns resolution is failing.


### Extra notes
- Specify ip's with ports if you have port conflicts and can use another static ip. Update that in the caddy section, leave the condinuwuity address at 0.0.0.0.
- If you blocked the federation checker access to local devices try your endpoints manually, the matrix url /.well-known/matrix/client and /.well-known/matrix/server if those load your api endpoints should be fine. It can take a day or more for the checker to re-prompt to access local devices again.
- In some instances I have seen the server seemingly expecting to be on port 8008 or 8448, these do not seem to be required ports.
- Trying to map ports around is likely to cause problems, the federation mechanism relies on https on 443 if you change ports other servers will have difficulty finding the correct port to get to the server file that is supposed to tell it to use that port.
- Add CONTINUWUITY_ALLOW_REGISTRATION: 'false' to turn off new user registration even with the token.

