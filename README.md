## quick start playground for vault

run server with docker-compose

```bash
docker-compose up -d
```

use interactive shell inside container

```
docker exec -ti vault sh
```

run vault command outside container

```
docker exec vault vault ...
```

access web UI though `http://localhost:8200`

note that for vault running in docker, there are two pre enabled secrets engine.

```
Path          Type         Accessor              Description
----          ----         --------              -----------
cubbyhole/    cubbyhole    cubbyhole_94b4eecc    per-token private secret storage
identity/     identity     identity_2381f067     identity store
secret/       kv           kv_eb71f033           key/value secret storage
sys/          system       system_43ed7278       system endpoints used for control, policy and debugging
```

- more detailed officail quick start guide. https://learn.hashicorp.com/collections/vault/getting-started
- vault command reference https://www.vaultproject.io/docs/commands
- entity and group example https://learn.hashicorp.com/tutorials/vault/identity

## some examples

In this example, we're going to

- enable `userpass` and `github` auth method.
- create user `john` under userpass
- create entity with alias for `john`
- create a policy allowing access to `secret/team-dev` path
- create an internal group named `dev` with `john` *entity* as member
- test whether permission works as expected

find initial root token

```
docker logs vault

# such as
# Unseal Key: cTHUoQCN4+ahUep79UcvhLI0ozZ/PJETF1l/NBFp+Ok=
# Root Token: s.9zrVFzFMVRTyOZxDWfPIxGR4
```

login with root token

```
# note -ti here, its required to input token interactively
docker exec -ti vault vault login
# or
docker exec -e VAULT_TOKEN=$token vault vault login
```

### enable userpass auth method

```
# enable
docker exec vault vault auth enable userpass

# create user
docker exec vault vault write auth/userpass/users/john 'password=s3cret'

# login with new user
docker exec -ti vault vault login -method=userpass username=john
```

### enable github auth method

```
# enable
docker exec vault vault auth enable github

# configure
docker exec vault vault write auth/github/config organization=wiredcraft

# map team to policy
docker exec vault vault write auth/github/map/teams/dev value=dev-policy

# login, get personal token here https://github.com/settings/tokens
docker exec -ti vault vault login -method=github
```

### create policy

```
cat dev.hcl | docker exec -i vault vault policy write dev -
```

### create entity and alias

*note*: this is not required if you are using web UI. login with web UI will create entity with alias automatically.

get entity id

```
docker exec vault vault write identity/entity name=john -format=json | jq '.data.id' > entity_id.txt
```

get auth accessor

```
docker exec vault vault auth list -format=json | jq '.["userpass/"].accessor' > accessor.txt
```

create entity alias

```
docker exec vault vault write identity/entity-alias name=john canonical_id=$entity_id mount_accessor=$accessor_id
```

### create internal groups

```
# create groups
docker exec vault vault write identity/group name=admin
docker exec vault vault write identity/group name=dev policies=dev member_entity_ids=$entity_id

# list groups
docker exec vault vault list identity/group/name

# show group info
docker exec vault vault read identity/group/name/dev
```

### check token capabilities

```
# grab token
docker exec vault vault login -method=userpass username=john password=

# this should give "create, delete, read, update"
docker exec -e VAULT_TOKEN=$token vault vault token capabilities secret/data/team-dev
```

test permission

```
# write
docker exec -e VAULT_TOKEN=$token vault vault kv put secret/team-dev token=123

# read
docker exec -e VAULT_TOKEN=$token vault vault kv get secret/team-dev
```


try removing entity from group

```
# we now remove entity from group
docker exec vault vault write identity/group name=dev policies=dev member_entity_ids=$entity_id

# this should now give "deny"
docker exec -e VAULT_TOKEN=$token vault vault token capabilities secret/data/team-dev
```
