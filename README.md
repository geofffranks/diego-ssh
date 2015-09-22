# diego-ssh

Diego-ssh is an implementation of an ssh proxy server, a lightweight ssh
daemon, and a Cloud Foundry cli plugin. When deployed and configured
correctly, they provide a simple and scalable way to access containers
associated with Diego long running processes.

## Proxy

The ssh proxy hosts the user-accessible ssh endpoint and is responsible for
authentication, policy enforcement, and access controls in the context of
Cloud Foundry. After a user has successfully authenticated with the proxy, the
proxy will attempt to locate the target container and create an ssh session to
a daemon running inside the container. After both sessions have been
established, the proxy will manage the communication between the user's ssh
client and the container's ssh daemon.

### Proxy Authentication

Clients authenticate with the proxy using a specially formed user name that
describes the authentication domain and target container and a password that
contains the appropriate credentials for the domain.

The proxy currently supports authentication against a `diego` domain and a
`cf` domain. Each authentication domain can be enabled independently via
command line arguments.

#### Diego via custom credentials

For Diego, the user is of the form `diego:`_process-guid_/_index_ and the
password must hold the configured credentials.

Client example:
```
$ ssh -p 2222 'diego:my-process-guid/1'@ssh.bosh-lite.com
$ scp -P 2222 -oUser='diego:ssh-process-guid/0' my-local-file.json ssh.bosh-lite.com:my-remote-file.json
```

The credentials checked by the proxy are configurable via the
`--diegoCredentials` flag.  The password provided by the client to the proxy
must match what is present in the flag for successful authentication.

This support is enabled with the `--enableDiegoAuth` flag.

#### Cloud Foundry via Cloud Controller and UAA

For Cloud Foundry, the user is of the form `cf:`_app-guid_/_instance_ and the
password must be either a valid OAuth 2 bearer token that represents the end
user or an authorization code that the ssh proxy server can exchange for an
authorization token.

In some environments, the oauth token length exceeds the limits of standard
ssh clients. When this happens, a one-time authorization code can be used as
an alternative. If the authorization code flow is used, the SSH proxy must be
configured with an OAuth client id and secret. The client id used by the proxy
must be advertised in the `/v2/info` endpoint under the `app_ssh_oauth_client`
key.  Please see the [UAA][non-standard-oauth-auth-code] documentation for
details on how to allocate an authorization code.

The proxy will contact the Cloud Controller as the user to determine if the
policy allows the user to access application containers via SSH.

Client example:
```
$ cf oauth-token | tail -1 | pbcopy # paste oauth token when prompted for password
```
or
```
$ curl -k -v -H "Authorization: $(cf oauth-token | tail -1)" \
    https://uaa.bosh-lite.com/oauth/authorize \
    --data-urlencode  "client_id=$(cf curl /v2/info | jq -r .app_ssh_oauth_client)" \
    --data-urlencode 'response_type=code' 2>&1 | \
    grep Location: | \
    cut -f2 -d'?' | \
    cut -f2 -d'=' | \
    pbcopy # paste oauth token when prompted for password
```

Use the token or one-time authorization code as the password:

```
$ ssh -p 2222 cf:$(cf app app-name --guid)/0@ssh.bosh-lite.com
$ scp -P 2222 -oUser=cf:$(cf app app-name --guid)/0 my-local-file.json ssh.bosh-lite.com:my-remote-file.json
```

Cloud Foundry `cf` client example that uses the [ssh][ssh-plugin] plugin:
```
$ cf install-plugin https://github.com/cloudfoundry-incubator/diego-ssh/releases/download/${version}/ssh-plugin-${platform}-${arch}
$ cf ssh app-name
```

This support is enabled with the `--enableCFAuth` flag.

### Daemon discovery

To be accessible via the SSH proxy, containers must host an ssh daemon, expose
it via a mapped port, and advertise the port in a `diego-ssh` route. The proxy
will fail end user authentication if the target LRP or a route is not found.

```json
  "routes": {
    "diego-ssh": { "container_port": 2222 }
  }
```

The [CC-Bridge][bridge] components of Diego will generate the appropriate LRP
definitions for Cloud Foundry applications which reflect the policies that are
in effect.

### Proxy to Container Authentication

When the proxy attempts to handshake with the SSH daemon inside the target
container, it will use the information associated with the `diego-ssh` key in
the LRP routes.

#### `container_port` [required]
`container_port` indicates which port inside the container the ssh daemon is
listening on. The proxy will attempt to connect to host side mapping of this
port after authenticating the client.

#### `host_fingerprint` [optional]
When present, `host_fingerprint` declares the expected fingerprint of the SSH
daemon's host public key. When the fingerprint of the actual target's host key
does not match the expected fingerprint, the connection is terminated. The
fingerprint should only contain the hex string generated by `ssh-keygen -l`.

#### `user` [optional]
`user` declares the user ID to use during authentication with the container's
SSH daemon. While it's not a required part of the routing data, it is required
for password authentication and may be required for public key authentication.

#### `password` [optional]
`password` declares the password to use during password authentication with
the container's ssh daemon.

#### `private_key` [optional]
`private_key` declares the private key to use when authenticating with the
container's SSH daemon. If present, the key must be a PEM encoded RSA or DSA
public key.

##### Example LRP
```json
{
  "process_guid": "ssh-process-guid",
  "domain": "ssh-experiments",
  "rootfs": "preloaded:cflinuxfs2",
  "instances": 1,
  "start_timeout": 30,
  "setup": {
    "download": {
      "artifact": "diego-sshd",
      "from": "http://file-server.service.cf.internal:8080/v1/static/diego-sshd/diego-sshd.tgz",
      "to": "/tmp",
      "cache_key": "diego-sshd"
    }
  },
  "action": {
    "run": {
      "path": "/tmp/diego-sshd",
      "args": [
          "-address=0.0.0.0:2222",
          "-authorizedKey=ssh-rsa ..."
      ],
      "env": [],
      "resource_limits": {}
    }
  },
  "ports": [ 2222 ],
  "routes": {
    "diego-ssh": {
      "container_port": 2222,
      "private_key": "PEM encoded PKCS#1 private key"
    }
  }
}
```

## SSH Daemon

The ssh daemon is a lightweight implementation that is built around go's ssh
library. It supports command execution, interactive shells, local port
forwarding, and scp. The daemon is self-contained and has no dependencies on the
container root file system.

The daemon is focused on delivering basic access to application instances in
Cloud Foundry. It is intended to run as an unprivileged process and
interactive shells and commands will run as the daemon user. The daemon only
supports one authorized key is not intended to support multiple users.

The daemon can be made available on a file server and Diego LRPs that
want to use it can include a download action to acquire the binary and a run
action to start it. Cloud Foundry applications will download the daemon as
part of the lifecycle bundle.

[bridge]: https://github.com/cloudfoundry-incubator/diego-design-notes#cc-bridge-components
[cflinuxfs2]: https://github.com/cloudfoundry/stacks/tree/master/cflinuxfs2
[ssh-plugin]: https://github.com/cloudfoundry-incubator/diego-ssh/releases

## CF CLI Plugin

The [cf CLI](https://github.com/cloudfoundry/cli) plugin adds the following commands:
- **ssh** - ssh to an application container instance
- **enable-ssh** - enable ssh for the application
- **disable-ssh** - disable ssh for the application
- **ssh-enabled** - reports whether SSH is enabled on an application container instance
- **allow-space-ssh** - allow SSH access for the space
- **disallow-space-ssh** - disallow SSH access for the space
- **space-ssh-allowed** - reports whether SSH is allowed in a space

To build and install the plugin:
```
cd ~/workspace/diego-release/src/github.com/cloudfoundry-incubator/diego-ssh/cf-plugin
go build
cf uninstall-plugin Diego-SSH
cf install-plugin cf-plugin
```

[non-standard-oauth-auth-code]: https://github.com/cloudfoundry/uaa/blob/master/docs/UAA-APIs.rst#api-authorization-requests-code-get-oauth-authorize-non-standard-oauth-authorize
