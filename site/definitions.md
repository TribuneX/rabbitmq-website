<!--
Copyright (c) 2007-2022 VMware, Inc. or its affiliates.

All rights reserved. This program and the accompanying materials
are made available under the terms of the under the Apache License,
Version 2.0 (the "License”); you may not use this file except in compliance
with the License. You may obtain a copy of the License at

https://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

# Schema Definition Export and Import

## <a id="overview" class="anchor" href="#overview">Overview</a>

Nodes and clusters store information that can be thought of schema, metadata or topology.
Users, vhosts, queues, exchanges, bindings, runtime parameters all fall into this category.
This metadata is called **definitions** in RabbitMQ parlance.

Definitions can be [exported](#export) to a file and then [imported](#import) into another cluster or
used for schema [backup](backup.html) or data seeding.

Definitions are stored in an internal database and replicated across all cluster nodes.
Every node in a cluster has its own replica of all definitions. When a part of definitions changes,
the update is performed on all nodes in a single transaction. This means that
in practice, definitions can be exported from any cluster node with the same result.

[VMware Tanzu RabbitMQ](https://docs.vmware.com/en/VMware-Tanzu-RabbitMQ-for-Kubernetes/index.html) supports [continuous schema definition replication](definitions-standby.html) to a remote cluster,
which makes it easy to run a warm standby cluster for disaster recovery.

Definition import on node boot is the recommended way of [pre-configuring nodes at deployment time](#import-on-boot).

## <a id="export" class="anchor" href="#export">Definition Export</a>

Definitions are exported as a JSON file in a number of ways.

 * [`rabbitmqctl export_definitions`](cli.html) is the only option that does not require [management plugin](management.html) to be enabled
 * The `GET /api/definitions` API endpoint
 * [`rabbitmqadmin export`](management-cli.html) which uses the above HTTP API endpoint
 * There's a definitions pane on the Overview page

Definitions can be exported for a specific [virtual host](vhosts.html) or the entire cluster (all virtual host).
When definitions are exported for just one virtual host, some information (contents of the other
virtual hosts or users without any permissions to the target virtual host) will be
excluded from the exported file.

Exported user data contains password hashes as well as [password hashing function](passwords.html) information. While brute forcing passwords with hashing functions such as SHA-256 or SHA-512 is not a completely trivial task,
user records should be **considered sensitive information**.

To export definitions using [`rabbitmqctl`](cli.html), use `rabbitmqctl export_definitions`:

<pre class="lang-bash">
# Does not require management plugin to be enabled
rabbitmqctl export_definitions /path/to/definitions.file.json
</pre>

`rabbitmqadmin export` is very similar but uses the HTTP API and is compatible
with older versions:

<pre class="lang-bash">
# Requires management plugin to be enabled
rabbitmqadmin export /path/to/definitions.file.json
</pre>

In this example, the `GET /api/definitions` endpoint is used directly to export
definitions of all virtual hosts in a cluster:

<pre class="lang-bash">
# Requires management plugin to be enabled,
# placeholders are used for credentials and hostname.
# Use HTTPS when possible.
curl -u {username}:{password} -X GET http://{hostname}:15672/api/definitions
</pre>

The response from the above API endpoint can be piped to [`jq`](https://stedolan.github.io/jq/) and similar tools
for more human-friendly formatting:

<pre class="lang-bash">
# Requires management plugin to be enabled,
# placeholders are used for credentials and hostname.
# Use HTTPS when possible.
#
# jq is a 3rd party tool that must be available in PATH
curl -u {username}:{password} -X GET http://{hostname}:15672/api/definitions | jq
</pre>


## <a id="import" class="anchor" href="#import">Definition Import</a>

To import definitions using [`rabbitmqctl`](cli.html), use `rabbitmqctl import_definitions`:

<pre class="lang-ini">
# Does not require management plugin to be enabled
rabbitmqctl import_definitions /path/to/definitions.file.json
</pre>

`rabbitmqadmin import` is its HTTP API equivalent:

<pre class="lang-ini">
# Requires management plugin to be enabled
rabbitmqadmin import /path/to/definitions.file.json
</pre>

It is also possible to use the `POST /api/definitions` API endpoint directly:

<pre class="lang-bash">
# Requires management plugin to be enabled,
# placeholders are used for credentials and hostname.
# Use HTTPS when possible.
curl -u {username}:{password} -X POST -T /path/to/definitions.file.json http://{hostname}:15672/api/definitions
</pre>


## <a id="import-on-boot" class="anchor" href="#import-on-boot">Definition Import at Node Boot Time</a>

A definition file can be imported during or after node startup time. In a multi-node cluster, at-boot-time imports
can and in practice will result in repetitive work performed by the nodes on boot. This is of no concern with
smaller definition files but with larger files, [importing definitions after node boot](#import-after-boot) after
cluster deployment (formation) is recommended.

Modern releases support definition import directly in the core,
without the need to [preconfigure](plugins.html#enabled-plugins-file) the [management plugin](management.html).

To import definitions from a local file on node boot,
set the `load_definitions` config key to a path of a previously exported JSON file with definitions:

<pre class="lang-ini">
# Does not require management plugin to be enabled.
load_definitions = /path/to/definitions/file.json
</pre>

From RabbitMQ `3.9.4` you can import definitions from a URL accessible over HTTPS on node boot.
Set the `definitions.import_backend` and `definitions.https.url` config keys to https and a valid URL where a JSON definition is located.

<pre class="lang-ini">
# Does not require management plugin to be enabled.
definitions.import_backend = https
definitions.https.url = https://raw.githubusercontent.com/rabbitmq/sample-configs/main/queues/5k-queues.json
# client-side TLS options for definition import
definitions.tls.versions.1 = tlsv1.2
</pre>


As of RabbitMQ `3.8.6`, definition import happens after plugin activation.
This means that definitions related to plugins (e.g. dynamic Shovels, exchanges of a custom type, and so on)
can be imported at boot time.

The definitions in the file will not overwrite anything already in the broker.
However, if a blank (uninitialised) node imports a definition file, it will
not create the default virtual host and user.

### <a id="import-on-boot-skip-if-unchanged" class="anchor" href="#import-on-boot-skip-if-unchanged">Avoid Boot Time Import if Definition Contents Have Not Changed</a>

By default definitions are imported by every cluster node, unconditionally.
In many environments definition file rarely changes. In that case it makes
sense to only perform an import when definition file contents actually change.
Starting with RabbitMQ 3.10, this can be done by setting the `definitions.skip_if_unchanged` configuration key
to `true`:

<pre class="lang-ini">
# when set to true, definition import will only happen
# if definition file contents change
definitions.skip_if_unchanged = true

definitions.import_backend = local_filesystem
definitions.local.path = /path/to/definitions/defs.json
</pre>

This feature works for both individual files and directories:

<pre class="lang-ini">
# when set to true, definition import will only happen
# if definition file contents change
definitions.skip_if_unchanged = true

definitions.import_backend = local_filesystem
definitions.local.path = /path/to/definitions/conf.d/
</pre>

 It is also supported by the HTTPS endpoint import mechanism:

<pre class="lang-ini">
# when set to true, definition import will only happen
# if definition file contents change
definitions.skip_if_unchanged = true

definitions.import_backend = https
definitions.https.url = https://some.endpoint/path/to/rabbitmq.definitions.json

definitions.tls.verify     = verify_peer
definitions.tls.fail_if_no_peer_cert = true

definitions.tls.cacertfile = /path/to/ca_certificate.pem
definitions.tls.certfile   = /path/to/client_certificate.pem
definitions.tls.keyfile    = /path/to/client_key.pem
</pre>


## <a id="import-after-boot" class="anchor" href="#import-after-boot">Definition Import After Node Boot</a>

Installations that use earlier versions that do not provide the built-in definition import
can import definitions immediately after node boot using a combination of two CLI commands:

<pre class="lang-bash">
# await startup for up to 5 minutes
rabbitmqctl await_startup --timeout 300

# import definitions using rabbitmqctl
rabbitmqctl import_definitions /path/to/definitions.file.json

# OR, import using rabbitmqadmin
# Requires management plugin to be enabled
rabbitmqadmin import /path/to/definitions.file.json
</pre>