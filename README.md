# Step Certificates

An online certificate authority and related tools for secure automated
certificate management, so you can use TLS everywhere.

[![GitHub stars](https://img.shields.io/github/stars/smallstep/certificates.svg?style=social)](https://github.com/smallstep/certificates/stargazers)
[![Twitter followers](https://img.shields.io/twitter/follow/smallsteplabs.svg?label=Follow&style=social)](https://twitter.com/intent/follow?screen_name=smallsteplabs)

For more information and docs see [the Step website](https://smallstep.com/cli/)
and the [blog post](https://smallstep.com/blog/step-certificates.html)
announcing Step Certificate Authority.

![Animated terminal showing step certificates in practice](https://github.com/smallstep/certificates/raw/master/images/step-ca.gif)

## Why?

Managing your own *public key infrastructure* (PKI) can be tedious and error
prone. Good security hygiene is hard. Setting up simple PKI is out of reach for
many small teams, and following best practices like proper certificate revocation
and rolling is challenging even for experts.

Amongst numerous use cases, proper PKI makes it easy to use mTLS (mutual TLS) to improve security and to make it possible to connect services across the public internet. Unlike VPNs & SDNs, deploying and scaling mTLS is pretty easy. You're (hopefully) already using TLS, and your existing tools and standard libraries will provide most of what you need. If you know how to operate DNS and reverse proxies, you know how to operate mTLS infrastructure.

![Connect it all with mTLS](https://raw.githubusercontent.com/smallstep/certificates/master/images/connect-with-mtls-2.png)

There's just one problem: **you need certificates issued by your own certificate authority (CA)**. Building and operating a CA, issuing certificates, and making sure they're renewed before they expire is tricky. This project provides the infratructure, automations, and workflows you'll need.


This project is part of smallstep's broader security architecture, which makes
it much easier to implement good security practices early, and incrementally
improve them as your system matures.

> ## 🆕 Autocert
> <a href="autocert/README.md"><img width="50%" src="https://raw.githubusercontent.com/smallstep/certificates/autocert/autocert/autocert-logo.png"></a>
>
> If you're using Kubernetes, make sure you [check out autocert](autocert/README.md): a kubernetes add-on that builds on `step certificates` to automatically injects TLS/HTTPS certificates into your containers.

### Table of Contents

- [Installing](#installing)
- [Documentation](#documentation)
- [Terminology](#terminology)
- [Getting Started](#getting-started)
- [Commonly Asked Questions](docs/common-questions.md)
- [Recommended Defaults](docs/recommendations.md)
- [How To Create A New Release](docs/distribution.md)
- [Versioning](#versioning)
- [LICENSE](./LICENSE)
- [CHANGELOG](./CHANGELOG.md)


## Installing

These instructions will install an OS specific version of the `step` binary on
your local machine.

### Mac OS

Install `step` via [Homebrew](https://brew.sh/):

```
brew install smallstep/smallstep/step
```

### Linux

Download the latest Debian package from [releases](https://github.com/smallstep/certificates/releases):

```
wget https://github.com/smallstep/certificates/releases/download/X.Y.Z/step-certificates_X.Y.Z_amd64.deb
```

Install the Debian package:

```
sudo dpkg -i step-certificates_X.Y.Z_amd64.deb
```

## Documentation

Documentation can be found in three places:

1. On the command line with `step ca help xxx` where `xxx` is the subcommand you are interested in. Ex: `step help ca provisioners list`

2. On the web at https://smallstep.com/docs/certificates

3. In your browser with `step ca help --http :8080` and visiting http://localhost:8080

## Terminology

### PKI - Public Key Infrastructure

A set of roles, policies, and procedures needed to create, manage, distribute,
use, store, and revoke digital certificates and manage public-key encryption.
The purpose of a PKI is to facilitate the secure electronic transfer of
information for a range of network activities.

### Provisioners

Provisioners are people or code that are registered with the CA and authorized
to issue "provisioning tokens". Provisioning tokens are single use tokens that
can be used to authenticate with the CA and get a certificate.

## Getting Started

Demonstrates setting up your own PKI and certificate authority using `step ca`
and getting certificates using the `step` command line tool and SDK.

![Animated terminal showing step ca init in practice](https://smallstep.com/images/blog/2018-12-04-unfurl.gif)

### Prerequisites

1. [Step CLI](https://github.com/smallstep/cli/blob/master/README.md#installing)

2. [Step CA](#installing)

### Initializing PKI and configuring the Certificate Authority

To initialize a PKI and configure the Step Certificate Authority run:

```
step ca init
```

You'll be asked for a name for your PKI. This name will appear in your CA
certificates. It doesn't really matter what you choose. The name of your
organization or your project will suffice.

If you run:

```
tree $(step path)
```

You should see:

```
.
├── certs
│   ├── intermediate_ca.crt
│   └── root_ca.crt
├── config
│   ├── ca.json
│   └── defaults.json
└── secrets
    ├── intermediate_ca_key
    └── root_ca_key
```

The files created include:

* `root_ca.crt` and `root_ca_key`: the root certificate and private key for
  your PKI
* `intermediate_ca.crt` and `intermediate_ca_key`: the intermediate certificate
  and private key that will be used to sign leaf certificates
* `ca.json`: the configuration file necessary for running the Step CA.
* `defaults.json`: file containing default parameters for the `step` CA cli
interface. You can override these values with the appropriate flags or
environment variables.

All of the files endinging in `_key` are password protected using the password
you chose during PKI initialization. We advise you to change these passwords
(using the `step crypto change-pass` utility) if you plan to run your CA in a
non-development environment.

### What's Inside `ca.json`?

`ca.json` is responsible for configuring communication, authorization, and
default new certificate values for the Step CA. Below is a short list of
definitions and descriptions of available configuration attributes.

* `root`: location of the root certificate on the filesystem. The root certificate
is used to mutually authenticate all api clients of the CA.

* `crt`: location of the intermediate certificate on the filesystem. The
intermediate certificate is returned alongside each new certificate,
allowing the client to complete the certificate chain.

* `key`: location of the intermediate private key on the filesystem. The
intermediate key signs all new certificates generated by the CA.

* `password`: optionally store the password for decrypting the intermediate private
key (this should be the same password you chose during PKI initialization). If
the value is not stored in configuration then you will be prompted for it when
starting the CA.

* `address`: e.g. `127.0.0.1:8080` - address and port on which the CA will bind
and respond to requests.

* `dnsNames`: comma separated list of DNS Name(s) for the CA.

* `logger`: the default logging format for the CA is `text`. The other options
is `json`.

* `tls`: settings for negotiating communication with the CA; includes acceptable
ciphersuites, min/max TLS version, etc.

* `authority`: controls the request authorization and signature processes.

    - `template`: default ASN1DN values for new certificates.

    - `claims`: default validation for requested attributes in the certificate request.
    Can be overriden by similar claims objects defined by individual provisioners.

        * `minTLSCertDuration`: do not allow certificates with a duration less
        than this value.

        * `maxTLSCertDuration`: do not allow certificates with a duration greater
        than this value.

        * `defaultTLSCertDuration`: if no certificate validity period is specified,
        use this value.

        * `disableIssuedAtCheck`: disable a check verifying that provisioning
        tokens must be issued after the CA has booted. This is one prevention
        against token reuse. The default value is `false`. Do not change this
        unless you know what you are doing.

    - `provisioners`: list of provisioners. Each provisioner has a `name`,
    associated public/private keys, and an optional `claims` attribute that will
    override any values set in the global `claims` directly underneath `authority`.


`step ca init` will generate one provisioner. New provisioners can be added by
running `step ca provisioner add`.

### Running the CA

To start the CA run:

```
export STEPPATH=$(step path)
step-ca $STEPPATH/config/ca.json
```

### Configure Your Environment

**Note**: Configuring your environment is only necessary for remote servers
(not the server on which the `step ca init` command was originally run).

Many of the cli utilities under `step ca [sub-command]` interface directly with
a running instance of the Step CA. The CA exposes an HTTP API and clients are
required to connect using HTTP over TLS (aka HTTPS). As part of bootstraping the
Step CA, a certificate was generated using the root of trust that was
created when you initilialized your PKI. In order to properly validate this
certificate clients need access to the public root of trust, aka the public
root certificate. If you are using the Step CLI on the same host where you
initialized your PKI (the `root_ca.crt` is stored on disk locally), then you
can continue to setting up a `default.json`, otherwise we will show you
how to easily download your root certificate in the following step.

#### Download the Root Certificate

The next few steps are a guide for downloading the root certificate of your PKI
from a running instance of the CA. First we'll define two servers:

* **remote server**: This is the server where the Step CA is running. This may
also be the server where you initialized your PKI, but for security reasons
you may have done that offline.

* **local server**: This is the server that wants access to the `step ca [sub-command]`

* **ca-url**: This is the url at which the CA is listening for requests. This
should be a combination of the DNS name and port entered during PKI initialization.
In the examples below we will use `https://ca.smallstep.com:8080`.

1. Get the Fingerprint.

    From the **remote server**:

    ```
    $ FP=$(step certificate fingerprint $(step path)/certs/root_ca.crt)
    ```

2. Bootstrap your environment.

    From the **local server**:

    ```
    $ step ca bootstrap --fingerprint $FP --ca-url "https://ca.smallstep.com:8080"
    $ cat $(step path)/config/defaults.json
    ```

3. Test.

    ```
    * step ca health
    ```

#### Setting up Environment Defaults
This is optional, but we recommend you populate a `defaults.json` file with a
few variables that will make your command line experience much more pleasant.

You can do this manually or with the step command `step ca bootstrap`:

```
$ step ca bootstrap \
  --ca-url https://ca.smallstep.com:8080 \
  --fingerprint 0d7d3834cf187726cf331c40a31aa7ef6b29ba4df601416c9788f6ee01058cf3
# Let's see what we got...
$ cat $STEPPATH/config/defaults.json
{
  "ca-url": "https://ca.smallstep.com:8080",
  "fingerprint": "628cfc85090ca65bb246d224f1217445be155cfc6167db4ed8f1b0e3de1447c5",
  "root": "/Users/<you>/src/github.com/smallstep/step/.step/certs/root_ca.crt"
}
# Test it out
$ step ca health
```

* **ca-url** is the DNS name and port that you used when initializing the CA.

* **root** is the path to the root certificate on the file system.

* **fingerprint** is the root certificate fingerprint (SHA256).

You can always override these values with command-line flags or environment
variables.

To manage the CA provisioners you can also add the property **ca-config** with
the path to the CA configuration file, with that property you won't need to add
it in commands like `step ca provisioners [add|remove]`.
**Note**: to manage provisioners you must be on the host on which the CA is
running. You need direct access to the `ca.json` file.

### Hot Reload

It is important that the CA be able to handle configuration changes with no downtime.
Our CA has a built in `reload` function allowing it to:

1. Finish processing existing connections while blocking new ones.
2. Parse the configuration file and re-initialize the API.
3. Begin accepting blocked and new connections.

`reload` is triggered by sending a SIGHUP to the PID (see `man kill`
for your OS) of the Step CA process. A few important details to note when using `reload`:

* The location of the modified configuration must be in the same location as it
was in the original invocation of `step-ca`. So, if the original command was

```
$ step-ca ./.step/config/ca.json
```

then, upon `reload`, the Step CA will read it's new configuration from the same
configuration file.

* Step CA requires the password to decrypt the intermediate certificate, again,
upon `reload`. You can automate this in one of two ways:

    * Use the `--password-file` flag in the original invocation.
    * Use the top level `password` attribute in the `ca.json` configuration file.

### Let's issue a certificate!

There are two steps to issuing a certificate at the command line:

1. Generate a provisioning token using your provisioning credentials.
2. Generate a CSR and exchange it, along with the provisioning token, for a certificate.

If you would like to generate a certificate from the command line, the Step CLI
provides a single command that will prompt you to select and decrypt an
authorized provisioner and then request a new certificate.

```
$ step ca certificate "foo.example.com" foo.crt foo.key
```

If you would like to generate certificates on demand from an automated
configuration management solution (no user input) you would split the above flow
into two commands.

```
$ TOKEN=$(step ca token foo.example.com \
    --kid 4vn46fbZT68Uxfs9LBwHkTvrjEvxQqx-W8nnE-qDjts \
    --ca-url https://ca.example.com \
    --root /path/to/root_ca.crt  --password-file /path/to/provisioner/password)

$ step ca certificate "foo.example.com" foo.crt foo.key --token "$TOKEN"
```

You can take a closer look at the contents of the certificate using `step certificate inspect`:

```
$ step certificate inspect foo.crt
```

### List|Add|Remove Provisioners

The Step CA configuration is initialized with one provisioner; one entity
that is authorized by the CA to generate provisioning tokens for new certificates.
We encourage you to have many provisioners - ideally one for each entity in your
infrastructure.

**Why should I be using multiple provisioners?**

* Each certificate generated by the Step CA contains the ID of the provisioner
that issued the *provisioning token* authorizing the creation of the cert. This
ID is stored in the X.509 ExtraExtensions of the certificate under
`OID: 1.3.6.1.4.1.37476.9000.64.1` and can be inspected by running `step
certificate inspect foo.crt`. These IDs can and should be used to debug and
gather information about the origin of a certificate. If every member of your
ops team and the configuration management tools all use the same provisioner
to authorize new certificates you lose valuable visibility into the workings
of your PKI.
* Each provisioner should require a **unique** password to decrypt it's private key
-- we can generate unique passwords for you but we can't force you to use them.
If you only have one provisioner then every entity in the infrastructure will
need access to that one password. Jim from your dev ops team should not be using
the same provisioner/password combo to authorize certificates for debugging as
Chef is for your CICD - no matter how trustworthy Jim says he is.

Let's begin by listing the existing provisioners:

```
$ bin/step ca provisioner list
```

Now let's add a provisioner for Jim.

```
$ bin/step ca provisioner add jim@smallstep.com --create
```

**NOTE**: This change will not affect the Step CA until a `reload` is forced by
sending a SIGHUP signal to the process.

List the provisioners again and you will see that nothing has changed.

```
$ bin/step ca provisioner list
```

Now let's `reload` the CA. You will need to re-enter your intermediate
password unless it's in your `ca.json` or your are using `--password-file`.

```
$ ps aux | grep step-ca   # to get the PID
$ kill -1 <pid>
```

Once the CA is running again, list the provisioners, again.

```
$ bin/step ca provisioner list
```

Boom! Magic.
Now suppose Jim forgets his password ('come on Jim!'), and he'd like to remove
his old provisioner. Get the `kid` (Key ID) of Jim's provisioner by listing
the provisioners and finding the appropriate one. Then run:

```
$ bin/step ca provisioner remove jim@smallstep.com --kid <kid>
```

Then `reload` the CA and verify that Jim's provisioner is no longer returned
in the provisioner list.

We can also remove all of Jim's provisioners, supposing Jim forgot all the passwords
('really Jim?'), by running the following:

```
$ bin/step ca provisioner remove jim@smallstep.com --all
```

The same entity may have multiple provisioners for authorizing different
types of certs. Each of these provisioners must have unique keys.

## Notes on Securing the Step CA and your PKI.

In this section we recommend a few best practices when it comes to
running, deploying, and managing your own online CA and PKI. Security is a moving
target and we expect out recommendations to change and evolve as well.

### Initializing your PKI

When you initialize your PKI two private keys are generated; one intermediate
private key and one root private key. It is very important that these private keys
are kept secret. The root private key should be moved around as little as possible,
preferably not all - meaning it never leaves the server on which it was created.

### Passwords

When you intialize your PKI (`step ca init`) the root and intermediate
private keys will be encrypted with the same password. We recommend that you
change the password with which the intermediate was encrypted at your earliest
convenience.

```
$ step crypto change-pass $STEPPATH/secrets/intermediate_ca_key
```

Once you've changed the intermediate private key password you should never have
to use the root private key password again.

We encourage users to always use a password manager to generate random passwords
or let Step CLI generate passwords for you.

The next important matter is how your passwords are stored. We recommend using a
[password manager](https://en.wikipedia.org/wiki/List_of_password_managers).
There are many to choose from and the choice will depend on the risk & security
profile of your organization.

In addition to using a password manager to store all passwords (private key,
provisioner, etc.) we recommend using a threshold cryptography algorithm like
[Shamir's Secret Sharing](https://en.wikipedia.org/wiki/Shamir's_Secret_Sharing)
to divide the root private key across a handful of trusted parties.

### Provisioners

When you intialize your PKI (`step ca init`) a default provisioner will be created
and it's private key will be encrypted using the same password used to encrypt
the root private key. Before deploying the Step CA you should remove this
provisioner and add new ones that are encrypted with new, secure, random passwords.
See the section on [managing provisioners](#listaddremove-provisioners).

### Deploying

* Refrain from entering passwords for private keys or provisioners on the command line.
Use the `--password-file` flag whenever possible.
* Run the Step CA as a new user and make sure that the config files, private keys,
and passwords used by the CA are stored in such a way that only this new user
has permissions to read and write them.
* Use short lived certificates. Our default validity period for new certificates
is 24 hours. You can configure this value in the `ca.json` file. Shorter is
better - less time to form an attack.
* Short lived certificates are **not** a replacement for CRL and OCSP. CRL and OCSP
are features that we plan to implement, but are not yet available. In the mean
time short lived certificates are a decent alternative.
* Keep your hosts secure by enforcing AuthN and AuthZ for every connection. SSH
access is a big one.

## The Future

We plan to build more tools that facilitate the use and management of zero trust
networks.

* Tell us what you like and don't like about managing your PKI - we're eager to
help solve problems in this space.
* Tell us what features you'd like to see - open issues or hit us on
[Twitter](https://twitter.com/smallsteplabs).

## Versioning

We use [SemVer](http://semver.org/) for versioning. For the versions available,
see the [tags on this repository](https://github.com/smallstep/cli).

## License

This project is licensed under the Apache 2.0 License - see the
[LICENSE](./LICENSE) file for details

### Individual Contributor License

[![CLA assistant](https://cla-assistant.io/readme/badge/smallstep/certificates)](https://cla-assistant.io/smallstep/certificates)
