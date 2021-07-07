# SPIRE CockroachDB Integration

With the merging of [join token node joining](https://github.com/cockroachdb/cockroach/pull/63492), Cockroach's TLS bootstrapping story has been greatly simplified. However, there is still no good story around client certs - operators have to use `cockroach cert create-client`.

While the cli-based solution works fine for quickly generating user certs, it has some issues for long-lived services:

- Rotation: There is currently no story around client-side certificate rotation, meaning that compromised certificates can live forever.
- Bootstrapping: The `create-client` command requires a high level of privilege and requires a manual action per user. It also then requires the manual step of loading the certificate into your secret manager of choice. Ideally, we would like a system that can automatically manage minting & distribution of these certificates.

## SPIFFE/SPIRE

These problems are not unique to Cockroach, so open source projects already exist to address them:

- [SPIFFE](https://spiffe.io/docs/latest/spiffe-about/overview/) is an open standard for production identity based on x509 (and later JWT). It defines a spec for x509 certificates that revolves around a [SPIFFE ID](https://spiffe.io/docs/latest/spiffe-about/spiffe-concepts/#spiffe-id), a universal identifier for a workload.
- [SPIRE](https://spiffe.io/docs/latest/spire-about/) is an implementation of SPIFFE that allows workloads to "attest" to their identity using metadata like docker labels, kubernetes pod info, AWS metadata, etc. Once attestation has succeeded, SPIRE mints SPIFFE-compliant certificates and distributes them to the workloads.

Using SPIFFE/SPIRE, we can cut out all `cockroach cert` commands and operate the cluster using only the identities (SPIFFE x509 certificates) issued to us via SPIRE. This is a drastic decrease in operator overhead as all certificate minting and distribution is now automated.

## Demo

In this demo, we have 4 containers:

- `root-server`: In SPIRE deployments, the SPIRE server acts as the central registrar for workloads, both storing workload selector metadata and minting certificates for workloads. `root-server` is our SPIRE server for this deployment.
- `root-agent`: In SPIRE deployments, the SPIRE agent is what actually interacts with the workloads, verifying that they match the selectors in SPIRE server and passing certificates from the SPIRE server to the workload.
  This server/agent pattern seems overcomplicated in the single-node case that we have here; In production, however, you'll have a single SPIRE server to act as a source of truth and one SPIRE agent per node to relay the server's instructions.
- `roachfirst`: This is our CockroachDB instance.
- `clientfirst`: This is our client that will connect to Cockroach.

The goal of this demo is to use SPIRE to issue SPIFFE certificates to both Cockroach and the client and to then use those certificates to connect from the client to Cockroach.

### 1. Add SPIRE registration entries

Before spinning everything up, we need to tell SPIRE server how to identify our workloads. We're running in Docker Compose, so we'll use the [docker workload attestor](https://github.com/spiffe/spire/blob/v0.12.3/doc/plugin_agent_workloadattestor_docker.md). This attestor uses metadata about the containers to attest workloads. Specifically, we'll be using the docker labels `app:roachfirst` and `app:clientfirst` to identify our workloads.

Start just the server up and add in the registration entries:

```
$ docker-compose up -d root-server         
Starting spire-roach_root-server_1 ... done

$ docker-compose exec root-server /bin/sh  
/opt/spire # /opt/spire/bin/spire-server entry create \
      -parentID spiffe://example.org/spire/agent/x509pop/4b45c7c788f2ed572fa5aef7c39cbe2fd0523d78 \
      -spiffeID spiffe://example.org/workload/roachfirst \
      -dns roachfirst \
      -dns roachfirst.example.org \
      -selector docker:label:app:roachfirst
Entry ID         : fb8ebd54-b51b-448a-9aa6-80968f7579b7
SPIFFE ID        : spiffe://example.org/workload/roachfirst
Parent ID        : spiffe://example.org/spire/agent/x509pop/4b45c7c788f2ed572fa5aef7c39cbe2fd0523d78
Revision         : 0
TTL              : default
Selector         : docker:label:app:roachfirst
DNS name         : roachfirst
DNS name         : roachfirst.example.org

/opt/spire # /opt/spire/bin/spire-server entry create \
      -parentID spiffe://example.org/spire/agent/x509pop/4b45c7c788f2ed572fa5aef7c39cbe2fd0523d78 \
      -spiffeID spiffe://example.org/workload/clientfirst \
      -dns root \
      -selector docker:label:app:clientfirst
Entry ID         : 1b50b0be-8cfe-4aa4-872d-42a3b597263f
SPIFFE ID        : spiffe://example.org/workload/clientfirst
Parent ID        : spiffe://example.org/spire/agent/x509pop/4b45c7c788f2ed572fa5aef7c39cbe2fd0523d78
Revision         : 0
TTL              : default
Selector         : docker:label:app:clientfirst
DNS name         : root

/opt/spire # exit

$ docker-compose down
Removing spire-roach_clientfirst_1 ... done
Removing spire-roach_roachfirst_1  ... done
Removing spire-roach_root-agent_1  ... done
Removing spire-roach_root-server_1 ... done
Removing network spire-roach_default
```

In these commands, the `parentID` specifies what agent is allowed to attest the specified workload. That x509 SPIFFE ID was created because we used the [x509pop node attestor](https://github.com/spiffe/spire/blob/main/doc/plugin_server_nodeattestor_x509pop.md) to attest our agent. In a production setting, all agent attestation would happen via selectors that look more meaninful (and don't require extra certificate minting) like AWS IIDs, K8s SATs, etc.

Now that we've got our registration entries in, we can start up the whole system:

```
$ docker-compose up  
Creating network "spire-roach_default" with the default driver
Creating spire-roach_root-server_1 ... done
Creating spire-roach_root-agent_1  ... done
Creating spire-roach_roachfirst_1  ... done
Creating spire-roach_clientfirst_1 ... done
Attaching to spire-roach_root-server_1, spire-roach_root-agent_1, spire-roach_roachfirst_1, spire-roach_clientfirst_1
#### Lots of logs that hopefully don't show error messages #####
```

In another terminal, exec into our client and connect to the database:

```
$ docker-compose exec clientfirst /bin/bash
root@clientfirst:/app# cockroach sql --certs-dir=certs --user=root --url='postgres://roachfirst:26257/'
#
# Welcome to the CockroachDB SQL shell.
# All statements must be terminated by a semicolon.
# To exit, type: \q.
#
# Server version: CockroachDB CCL v21.1.3 (x86_64-unknown-linux-gnu, built 2021/06/21 19:57:59, go1.15.11) (same version as client)
# Cluster ID: 97d5d1ac-ba78-4eaf-b040-93c16b81bec2
#
# Enter \? for a brief introduction.
#
root@roachfirst:26257/defaultdb> show users;
  username | options | member_of
-----------+---------+------------
  admin    |         | {}
  root     |         | {admin}
(2 rows)

Time: 5ms total (execution 5ms / network 0ms)

root@roachfirst:26257/defaultdb> 
```

With that, we have a client connecting to Cockroach with credentials minted and distributed by SPIRE! As you might have noticed, we didn't have to touch certificates a single time during that process - thanks to the mechanisms in SPIRE, all certificates have been distributed and will even be rotated when their time is up.
