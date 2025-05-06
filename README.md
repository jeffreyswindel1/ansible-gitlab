ansible-gitlab
==============

This is a `docker compose` setup for running gitlab and traefik with self-signed certificates from `cfssl`. This is a work in progress, and not all services are complete. Local development setup can be tested with `molecule` or the `ansible` playbook.

Usage
-----

Start the stack:

```bash
ansible-playbook deploy/playbook.yml
```

Certificates
------------

Update `etc/cfssl/config/config.json` with the domain name and generate the Root CA:

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
