ansible-gitlab
==============

This is an [ansible](https://docs.ansible.com/) playbook that sets up [docker compose](https://docs.docker.com/compose/) for running [gitlab](https://about.gitlab.com/) and [traefik](https://traefik.io/traefik) with self-signed certificates from [cfssl](https://blog.cloudflare.com/introducing-cfssl/).

This is a work in progress, and not all services are complete.

Usage
-----

Start the stack:

```bash
ansible-playbook deploy/playbook-config.yml
```

For AWS:

```bash
ansible-playbook playbook-instance.yml
ansible-playbook -i hosts/aws_ec2.yml playbook-config.yml
```

After the stack has been created, if you do not specify a root password it can be found in /dkr/.env and that can be used to login.

For AWS:

```bash
aws ssm start-session \
   --target "${INSTANCE_ID}" \
   --document-name AWS-StartPortForwardingSession \
   --parameters "{\"portNumber\":[\"${LOCAL_PORT}\"],\"localPortNumber\":[\"${LOCAL_PORT}\"]}" \
   --region "${REGION}"
```

***replace $INSTANCE_ID with the id of the instance, $LOCAL_PORT of 8443, and $AWS_REGION of the deployed region (default us-west-2)***

Certificates
------------

Update `cfssl/config/config.json` with the domain name and generate the Root CA:

```bash
docker compose \
    run --rm \
    --entrypoint sh cfssl \
    -c "cfssl genkey -initca config/ca.json | cfssljson -bare ca"
```

Generate certificates:

```bash
# Run cfssl container to generate self-signed certificates
CONTAINERS=( gitlab traefik )
for C in "${CONTAINERS[@]}"; do
echo "Generating Certificate for ${C} container"
docker compose run --rm \
    --volume ./etc/cfssl:/etc/cfssl \
    --entrypoint sh cfssl \
    -c "cfssl gencert -ca ca.pem -ca-key ca-key.pem -config config/config.json -profile=server config/${C}.json | cfssljson -bare ${C}-server"
done

# Restart traefik
docker compose restart traefik
```

Testing
-------

This uses [molecule](https://ansible.readthedocs.io/projects/molecule/) and [docker](https://docker.io) to create a test instance for local development and updates.

```bash
cd deploy && molecule converge
```
