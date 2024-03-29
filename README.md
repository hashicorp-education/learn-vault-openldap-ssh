# Docker LDAP Secrets Engine with SSH lab

You can use the information in this lab to build a demonstration for testing authentication of SSH connections using LDAP and PAM, with OpenLDAP credentials managed by Vault.

The infrastructure in this demonstration lab consists of the following:

- 1 Vault container
- 1 OpenLDAP container
- 1 Secure Shell Daemon (sshd) container

Once you establish initial SSH authentication through OpenLDAP, you can use Vault to manage your LDAP user credential with the [LDAP Secrets Engine](https://developer.hashicorp.com/vault/docs/secrets/ldap).

> **NOTE:** As this is purely an informative feature demonstration, the environment is not configured to use TLS for Vault and OpenLDAP.

Refer to the [LDAP Secrets Engine tutorial](https://developer.hashicorp.com/vault/tutorials/secrets-management/openldap) for additional detail on using the LDAP secrets engine.

## Prerequisites

You need the following to try this demonstration lab.

1. [Docker Desktop](https://www.docker.com/products/docker-desktop) installed and running.

1. [HashiCorp Education learn-vault-openldap-ssh](https://github.com/hashicorp-education/learn-vault-openldap-ssh) repository clone.

1. LDAP utilities installed and on your system `PATH`; some operating systems, such as macOS install these by default, or you can install them via package on others.
    - `ldapadd`
    - `ldappasswd`

This lab was last tested 9 Mar 2023 on a macOS 12.6.3 using the following versions.

```shell
docker version --format '{{.Server.Version}}'
20.10.23
```

## Setup the infrastructure

Once you meet all prerequisites, proceed with the setup to establish a dedicated Docker network and the necessary containers for your demonstration infrastructure.

### Docker network

Create a Docker network for your containers to keep them isolated from any existing containers.

Create a bridged network named learn-vault-ldap-ssh.

```shell
docker network create learn-vault-ldap-ssh
```

**Output example:**

```plaintext
cab57becef85420f00cb78a41eecd21a196e102a52f7843a3ddff5f529ff4527
```

### OpenLDAP container

1. Run the OpenLDAP container.

    ```shell
    docker run \
      --name=learn-ldap \
      --hostname=learn-ldap \
      --network=learn-vault-ldap-ssh \
      -p 389:389 \
      -e LDAP_ORGANISATION="Example" \
      -e LDAP_DOMAIN="example.com" \
      -e LDAP_ADMIN_PASSWORD="admin" \
      --detach \
      --rm \
      osixia/openldap:1.3.0
    ```

    The flags to `docker run` define a container name, network hostname, Docker network name, LDAP port number, organization name, organization domain, and administrator password.  The command also specifies that the container detach from the terminal and remove itself when it exits.

    **Output example:**

    ```plaintext
    54276896dfbd840dda7af7b3e5407260889794529d16212da461578667cea3f1
    ```

1. Check that the container is up and ready.

    ```
    docker ps -f name=learn-ldap --format "table {{.Names}}\t{{.Status}}"
    NAMES               STATUS
    learn-ldap          Up 5 seconds
    ```

Now that the OpenLDAP server is ready, you can configure it.

#### OpenLDAP configuration

The OpenLDAP server needs a bit of additional configuration to define some initial groups and a POSIX user account named _learner_. You'll use this account to authenticate to the sshd container later on. You can find this configuration in the files `config/base.ldif` and `config/learn.ldif`.

1. Add configuration from `configs.base.ldif` to your OpenLDAP server to add initial users and groups.

    ```shell
    ldapadd \
      -x \
      -w admin \
      -D "cn=admin,dc=example,dc=com" \
      -f ./config/base.ldif
    ```

    **Output example:**

    ```plaintext
    adding new entry "ou=users,dc=example,dc=com"

    adding new entry "ou=groups,dc=example,dc=com"
    ```

1. Next, add the POSIX user _learner_ with configuration from the file `config/learner.ldif`.

    ```shell
    ldapadd \
      -x \
      -w admin \
      -D "cn=admin,dc=example,dc=com" \
      -f ./config/learner.ldif
    ```

    **Output example:**

    ```plaintext
    adding new entry "uid=learner,ou=users,dc=example,dc=com"

    adding new entry "cn=learners,ou=groups,dc=example,dc=com"
    ```

1. Set _learner_'s password value to the literal string password.

    ```shell
    ldappasswd \
      -s password \
      -w admin \
      -D "cn=admin,dc=example,dc=com" \
      -x "uid=learner,ou=users,dc=example,dc=com"
    ```

    This command produces no output.

With your OpenLDAP container fully configured, you can build and run the SSH container.

### Secure Shell Daemon (sshd) container

The sshd container in this demonstration lab configures PAM for LDAP based logins. You must build the container image from the included `Dockerfile.centos7` file before running it.

1. Build the container.

    ```shell
    docker build . -f Dockerfile.centos7 -t sshtest:1.0.0
    ```

    **Output example:**

    The process requires around a minute the first time depending on your Docker host system. The output should conclude with successful built and tagged messages as in this example.

    ```plaintext
    ...snip...
    Successfully built a3400498ff9e
    Successfully tagged sshtest:1.0.0
    ```

1. Run the container image.

    ```shell
    docker run \
      --name=learn-sshd \
      --hostname=learn-sshd \
      --network=learn-vault-ldap-ssh \
      -e SSH_PASSWORD_AUTHENTICATION='true' \
      -p 2022:22 \
      --detach \
      --rm \
      sshtest:1.0.0
    ```

    The flags define container name, network hostname, Docker network, an environment variable to enable password authentication in sshd, the sshd port mapping.

    You also specify that the container detach from the terminal and remove itself when it exits.

    **Output example:**

    ```
    b7ed04c3a15a3d9d0317a549bd4541f92a69322b5bd0422f9d3a33b16c27a8c4
    ```

1. Check that the container is up and ready.

    ```
    docker ps -f name=learn-sshd --format "table {{.Names}}\t{{.Status}}"
    NAMES               STATUS
    learn-sshd          Up 6 seconds (healthy)
    ```

Now that the sshd server is ready, you can test it.

## Test SSH authentication

1. Try to authenticate to the sshd container with your LDAP user and its password.

    ```shell
    ssh -l learner 0.0.0.0 -p 2022
    ```

1. Since this is your first connection to the sshd container, it prompts you with some information about the host and asked whether you want to continue connecting.

    ```plaintext
    The authenticity of host '[0.0.0.0]:2022 ([0.0.0.0]:2022)' can't be established.
    ECDSA key fingerprint is SHA256:jiMaT0yey0MUFs7qKg+OTMGPBILuBaUKkATVjB02s4o.
    Are you sure you want to continue connecting (yes/no)? yes
    ```

    Enter `yes` then press ENTER, then enter `password` when prompted for password. If all goes well, you authenticate with the _learn-sshd_ container and in a shell session.

    ```plaintext
    Warning: your password will expire in 0 days
    Creating directory '/home/learner'.
    [learner@sshd ~]$
    ```

1. Enter `exit` to get out of the SSH container shell.

Now that you've authenticated with the sshd server, configure Vault to manage the LDAP credential for your _learner_ user, and then rotate it.

### Vault container

You can use a simple Vault development server container for this demonstration lab.

1. Run the Vault container.

    ```shell
    docker run \
      --name=learn-vault \
      --hostname=learn-vault \
      --network=learn-vault-ldap-ssh \
      --cap-add=IPC_LOCK \
      -e 'VAULT_DEV_ROOT_TOKEN_ID=c0ffee0ca7' \
      -e 'VAULT_DEV_LISTEN_ADDRESS=0.0.0.0:8200' \
      -e 'VAULT_ADDR=http://127.0.0.1:8200' \
      -p 8200:8200 \
      --detach \
      --rm \
      vault:1.13.0
    ```

    The flags define container name, network hostname, Docker network, IPC_LOCK capability, environment variables to set Vault root token and listen address along with port mapping.

    You also specify that the container detach from the terminal and remove itself when it exits.

    **Output example:**

    ```plaintext
    ...snip...
    Status: Downloaded newer image for vault:1.13.0
    089b6beed68bafd942ae771f444008fd51694960973c15ce0af9700874f835b3
    ```

    > **NOTE:** You started the Vault server in [development mode](https://www.vaultproject.io/docs/commands/server/#inlinecode--dev-1); this means Vault initializes, unseals itself, and sets the initial root token to _c0ffee0ca7_ for you. You can use any 1.4.0+ version of the container.

    Beware that in dev server mode all data persist to memory, but not durable storage. If you stop the container, you will lose any progress from this point onward.

1. Check that the Vault server container is up and ready.

    ```shell
    docker ps -f name=learn-vault --format "table {{.Names}}\t{{.Status}}"
    NAMES               STATUS
    learn-vault          Up 14 seconds
    ```

    From this point on, you will execute a shell in the Vault server container and carry out commands to configure the Vault LDAP secrets engine from within the container itself.

1. Use `docker exec` to open a shell in the Vault server container.

    ```shell
    docker exec -it learn-vault sh
    ```

1. Now that the Vault server is ready, check the Vault server status.

    ```plaintext
    vault status
    ```

    **Output example:**

    ```plaintext
    Key             Value
    ---             -----
    Seal Type       shamir
    Initialized     true
    Sealed          false
    Total Shares    1
    Threshold       1
    Version         1.13.0
    Build Date      2023-03-01T14:58:13Z
    Storage Type    inmem
    Cluster Name    vault-cluster-66c051bc
    Cluster ID      778dc1a7-d378-3e49-1fd1-63afe8931750
    HA Enabled      false
    ```

    Vault is ready to go, and you are now ready to configure the LDAP secrets engine.

1. First, authenticate with the root token value `c0ffee0ca7`.

    ```plaintext
    vault login c0ffee0ca7
    ```

    **Output example:**

    ```plaintext
    Success! You are now authenticated. The token information displayed below
    is already stored in the token helper. You do NOT need to run "vault login"
    again. Future Vault requests will automatically use this token.

    Key                  Value
    ---                  -----
    token                c0ffee0ca7
    token_accessor       LzZz2PbkEOmZCPBVcRmcNWTQ
    token_duration       ∞
    token_renewable      false
    token_policies       ["root"]
    identity_policies    []
    policies             ["root"]
    ```

Now that you authenticated with the root token, continue with secrets engine configuration.

> ***NOTE:** This lab uses the initial root token for all Vault operations solely for convenience and simplicity. In production use, Vault you should guard root tokens, and not generally use them. Refer to the [Root Tokens](https://master--vault-www.netlify.com/docs/concepts/tokens/#root-tokens) documentation to learn more.


#### Enable and configure LDAP Secrets Engine

1. Enable the LDAP secrets engine.

    ```shell
    vault secrets enable ldap
    ```

    **Output:**

    ```plaintext
    Success! Enabled the ldap secrets engine at: ldap/
    ```

1. Configure the secrets engine with an example name and admin password.

    ```shell
    vault write ldap/config \
        binddn=cn=admin,dc=example,dc=com \
        bindpass=admin \
        url=ldap://learn-ldap
    ```

    **Output:**

    ```plaintext
    Success! Data written to: ldap/config
    ```

1. Rotate the root credential so that Vault has sole control of it from this point forward.

    ```shell
    vault write -f ldap/rotate-root
    ```

    **Output:**

    ```plaintext
    Success! Data written to: ldap/rotate-root
    ```

1. Now create a static role to manage the _learner_ user credentials and while you are at it, you can define an automatic rotation period of 24 hours for example as well.

    ```shell
    vault write ldap/static-role/learner \
        dn='uid=learner,ou=users,dc=example,dc=com' \
        username='learner' \
        rotation_period="24h"
    ```

    **Output:**

    ```plaintext
    Success! Data written to: ldap/static-role/learner
    ```

1. Rotate the _learner_ password

    ```shell
    vault write -f /ldap/rotate-role/learner
    ```

    **Output:**

    ```plaintext
    Success! Data written to: ldap/rotate-role/learner
    ```

## Test SSH authentication again

Now if you try authenticate with the sshd server container, it should no longer be possible to use the initial value of `password` for the learner user password.

Open a new command terminal and try SSH using the password you specified.

```shell
$ ssh -l learner 0.0.0.0 -p 2022
learner@0.0.0.0's password:
Permission denied, please try again.
```

Press CTRL+C to stop the ssh session.

You can no longer access the server with that password, but you can now ask Vault for a new password.

## Back to Vault

Return to the terminal where you are running a shell within the Vault container. You can ask Vault for a new SSH credential by reading from the `ldap/static-cred/learner` path. If you use the `-field` flag, you can ask for just the raw password string.

```plaintext
vault read -field=password ldap/static-cred/learner
```

> **NOTE:** This operation produces sensitive output.

**Output example:**

```plaintext
ZdSuDuHEUeeLlNijYXF527RzYdiF34h2YmgAv0EhNhpLRhCUmmpkGzenTQHyTs1H
```

If you'd prefer to output all the metadata associated with the secret, just omit the `-field=password`.

```plaintext
vault read ldap/static-cred/learner
```

**Output example:**

```plaintext
Key                    Value
---                    -----
dn                     uid=learner,ou=users,dc=example,dc=com
last_vault_rotation    2020-03-10T17:42:07.6574783Z
password               ZdSuDuHEUeeLlNijYXF527RzYdiF34h2YmgAv0EhNhpLRhCUmmpkGzenTQHyTs1H
rotation_period        72h
ttl                    71h59m40s
username               learner
```

Now that you know the credential value that Vault has generated during the earlier rotate operation, try to use it with the sshd server container.

## Test SSH authentication once more

If you try to authenticate with the sshd server container using your updated password as read from Vault, you should observe success.

```shell
ssh -l learner 0.0.0.0 -p 2022
learner@0.0.0.0's password:
```

**Output example:**

```
Warning: your password will expire in 0 days
Last failed login: Thu Mar  9 16:02:50 UTC 2023 from 172.20.0.1 on ssh:notty
There was 1 failed login attempt since the last successful login.
Last login: Thu Mar  9 16:01:21 2023 from 172.20.0.1
```

Enter `exit` to exit out of the SSH shell.

## Cleanup

You can clean up the environment by stopping the Docker containers and network.

```shell
docker rm learn-vault --force && \
  docker rm learn-ldap --force && \
  docker rm learn-sshd --force && \
  docker network rm learn-vault-ldap-ssh
```

**Output:**

```plaintext
learn-vault
learn-ldap
learn-sshd
learn-vault-ldap-ssh
```

## Help and reference

1. [Docker Desktop](https://www.docker.com/products/docker-desktop)
1. [vault-guides repository](https://github.com/hashicorp/vault-guides)
1. [hashicorp/vault](https://hub.docker.com/_/vault)
1. [hashicorp/vault GitHub repository](https://github.com/hashicorp/docker-vault)
1. [Vault server development mode](https://www.vaultproject.io/docs/commands/server/#inlinecode--dev-1)
1. [LDAP Secrets Engine](https://developer.hashicorp.com/vault/docs/secrets/ldap)
1. [LDAP Secrets Engine API](https://developer.hashicorp.com/vault/api-docs/secret/ldap)
1. [OpenLDAP](https://www.openldap.org/)
1. [osixia/openldap](https://hub.docker.com/r/osixia/openldap)
1. [osixia/openldap  GitHub repository](https://github.com/osixia/docker-openldap)
1. [jdeathe/centos-ssh](https://hub.docker.com/r/jdeathe/centos-ssh/)
1. [jdeathe/centos-ssh GitHub repository](https://github.com/jdeathe/centos-ssh)
