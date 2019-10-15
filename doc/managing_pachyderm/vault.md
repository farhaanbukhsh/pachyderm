# Vault Secret Engine

Pachyderm supports Vault integration by providing a Vault Secret Engine.


## Deployment

Vault instructions for the admin deploying/configuring/managing vault

1) Get plugin binary

- Navigate to the Pachyderm Repo [on github]()
    - Go to the latest release page
    - Download the `vault` asset

2) Download / Install that binary on your vault server instance

On your vault server:

```
# Assuming the binary was downloaded to /tmp/vault-plugins/pachyderm
export SHASUM=$(shasum -a 256 "/tmp/vault-plugins/pachyderm" | cut -d " " -f1)
echo $SHASUM
vault write sys/plugins/catalog/pachyderm sha_256="$SHASUM" command="pachyderm"
vault secrets enable -path=pachyderm -plugin-name=pachyderm plugin
```

**Note**: You may need to enable memory locking on the pachyderm plugin (see
[https://www.vaultproject.io/docs/configuration/#disable_mlock]). That will look
like:
```
sudo setcap cap_ipc_lock=+ep $(readlink -f /tmp/vault-plugins/pachyderm)
```

3) Configure the plugin

We'll need to gather and provide this information to the plugin for it to work:

- `admin_token` : is the (machine user) pachyderm token the plugin will use to cut new credentials on behalf of users
- `pachd_address` : is the URL where the pachyderm cluster can be accessed
- `ttl` : is the max TTL a token can be issued


#### Admin Token

To get a machine user `admin_token` from pachyderm:

###### If auth is not activated
(this activates auth with a robot user. It's also possible to activate auth with a github user. Also, the choice of `robot:admin` is arbitrary. You could name this admin `robot:<any string>`)
```
$ pachctl auth activate --initial-admin=robot:admin
Retrieving Pachyderm token...
WARNING: DO NOT LOSE THE ROBOT TOKEN BELOW WITHOUT ADDING OTHER ADMINS.
IF YOU DO, YOU WILL BE PERMANENTLY LOCKED OUT OF YOUR CLUSTER!
Pachyderm token for "robot:admin":
34cffc9254df40f0a277ee23e9fb005d

$ ADMIN_TOKEN=34cffc9254df40f0a277ee23e9fb005d
$ echo "${ADMIN_TOKEN}" | pachctl auth use-auth-token # authenticates you as the cluster admin
```

###### If auth *is* already activated
```
# Login as a cluster admin
$ pachctl auth login
... login as cluster admin ...

# Appoint a new robot user as the cluster admin (if needed)
$ pachctl auth modify-admins --add=robot:admin

# Get a token for that robot user admin
$ pachctl auth get-auth-token robot:admin
New credentials:
  Subject: robot:admin
  Token: 3090e53de6cb4108a2c6591f3cbd4680

$ ADMIN_TOKEN=3090e53de6cb4108a2c6591f3cbd4680
```

Pass the new admin token to Pachyderm:
```
vault write pachyderm/config \
    admin_token="${ADMIN_TOKEN}" \
    pachd_address="127.0.0.1:30650" \
    ttl=5m # optional
```
4) Test the plugin

```
vault read pachyderm/version

# If this fails, check if the problem is in the client (rather than the server):
vault read pachyderm/version/client-only
```

5) Manage user tokens with `revoke`

```
$ vault token revoke d2f1f95c-2445-65ab-6a8b-546825e4997a
Success! Revoked token (if it existed)
```

Which will revoke the vault token. But if you also want to manually revoke a pachyderm token, you can do so by issuing:

```
$vault write pachyderm/revoke user_token=xxx

```

## Usage

When your application needs to access pachyderm, you will first do the following:

1) Connect / login to vault

Depending on your language / deployment this can vary. [see the vault documentation]() for more details.

2) Anytime you are going to issue a request to a pachyderm cluster first:

- check to see if you have a valid pachyderm token
    - if you do not have a token, hit the `login` path as described below
    - if you have a token but it's TTL will expire soon (latter half of TTL is what's recommended), hit the `renew` path as described below
- then use the response token when constructing your client to talk to the pachyderm cluster

### Login

Again, your client could be in any language. But as an example using the vault CLI:

```
$ vault write -f pachyderm/login/robot:test
Key                Value
---                -----
lease_id           pachyderm/login/robot:test/e93d9420-7788-4846-7d1a-8ac4815e4274
lease_duration     768h
lease_renewable    true
pachd_address      192.168.99.100:30650
user_token         aa425375f03d4a5bb0f529379d82aa39
```

The response metadata contains the `user_token` that you need to use to connect to the pachyderm cluster,
    as well as the `pachd_address`.
Again, if you wanted to use this Pachyderm token on the command line:
```
$ echo "aa425375f03d4a5bb0f529379d82aa39" | pachctl auth use-auth-token
$ pachctl config update context `pachctl config get active-context` --pachd-address=127.0.0.1:30650
$ pachctl list repo
```

The TTL is tied to the vault lease in `lease_id`, which can be inspected or revoked
  using the vault lease API (documented here: https://www.vaultproject.io/api/system/leases.html):

```
$ vault write /sys/leases/lookup lease_id=pachyderm/login/robot:test/e93d9420-7788-4846-7d1a-8ac4815e4274
Key             Value
---             -----
expire_time     2018-06-17T23:32:23.317795215-07:00
id              pachyderm/login/robot:test/e93d9420-7788-4846-7d1a-8ac4815e4274
issue_time      2018-05-16T23:32:23.317794929-07:00
last_renewal    <nil>
renewable       true
ttl             2764665
```


### Renew

You should issue a `renew` request once the halfway mark of the TTL has elapsed.
Like revocation, renewal is handled using the vault lease API:
```
$ vault write /sys/leases/renew lease_id=pachyderm/login/robot:test/e93d9420-7788-4846-7d1a-8ac4815e4274 increment=3600
Key                Value
---                -----
lease_id           pachyderm/login/robot:test/e93d9420-7788-4846-7d1a-8ac4815e4274
lease_duration     2h
lease_renewable    true
```
