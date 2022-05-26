[![Community Project header](https://github.com/newrelic/opensource-website/raw/master/src/images/categories/Community_Project.png)](https://opensource.newrelic.com/oss-category/#community-project)

# New Relic Pixie Integration

The Pixie integration connects to the Pixie API and enables the New Relic plugin in Pixie. The plugin then sends data generated by a set of PxL scripts to New Relic using the OpenTelemetry Line Protocol. The integration takes the preset script of the New Relic plugin and enables them for the provided cluster. The integration can be configured to register a set of custom PxL scripts on top of the preset scripts.

Once the New Relic plugin is successfully enabled and the PxL scripts are registered, the integration exits with error code 0.

## Getting Started

Make sure you have a Pixie account and a New Relic account set up, and have collected the following information:

 * [New Relic license key](https://docs.newrelic.com/docs/accounts/accounts-billing/account-setup/new-relic-license-key/)
 * [Pixie API key](https://docs.pixielabs.ai/using-pixie/api-quick-start/#get-an-api-token) - Using the px CLI: `px api-key create -s`
 * [Pixie Cluster ID](https://docs.pixielabs.ai/using-pixie/api-quick-start/#get-a-cluster-id) - Using the px CLI: `px get cluster --id`
 * Pixie endpoint - Using the px CLI: `px get cluster --cloud-addr`
 * Cluster name: the name of your Kubernetes cluster

## Building

```make build-container```

Docker is required to build the Pixie integration container image.

## Usage

```docker run --env-file ./env.list -it newrelic/newrelic-pixie-integration:latest```

Define the following environment variables in the `env.list` file:

```
NR_LICENSE_KEY=
PIXIE_API_KEY=
PIXIE_CLUSTER_ID=
PIXIE_ENDPOINT=
CLUSTER_NAME=
```

The following environment variables are optional. 

```
HTTP_SPAN_LIMIT=5000
DB_SPAN_LIMIT=1000
COLLECT_INTERVAL_SEC=10
HTTP_SPAN_COLLECT_INTERVAL_SEC=10
JVM_COLLECT_INTERVAL_SEC=10
MYSQL_COLLECT_INTERVAL_SEC=10
POSTGRES_COLLECT_INTERVAL_SEC=10
EXCLUDE_PODS_REGEX=
EXCLUDE_NAMESPACES_REGEX=
SCRIPT_DIR=/scripts
VERBOSE=true
```

The `*_LIMIT` and `*_INTERVAL_SEC` environment variables can be used to control the amount of data that is sent to New Relic. The `COLLECT_INTERVAL_SEC` environment variable sets the collection interval for all PxL scripts. The other `*_INTERVAL_SEC` environment variables are overrides for the individual scripts. Setting the interval to `-1` will disable sending that data to New Relic. The smallest valid interval is 2 seconds.

The `EXCLUDE_PODS_REGEX` and `EXCLUDE_NAMESPACES_REGEX` environment variables can be configured with [RE2 regular expressions](https://github.com/google/re2/wiki/Syntax) to not send observability data to New Relic for the matching pods and namespaces. When `EXCLUDE_NAMESPACES_REGEX` is provided, no data for the matching namespaces will be sent. When `EXCLUDE_PODS_REGEX` is provided, no data for the matching pods (independent of the namespace they are in) will be sent.

## Custom scripts

## Testing

### Unit tests

Run the unit tests locally:

```make test```

### End-to-end tests

After executing the command above Pixie data should be flowing into your New Relic account. Use the following NRQL queries to verify this:

**Metrics**
```
SELECT * FROM Metric WHERE instrumentation.provider='pixie'
```

**Spans**
```
SELECT * FROM Span WHERE instrumentation.provider='pixie'
```

## Custom scripts

The integration picks up custom scripts in the `SCRIPT_DIR` (set to `/scripts` by default). The custom scripts should be defined in a yaml file with the `.yaml` extension. The following fields are required:

 * name (string): the name of the script
 * description (string): description of the script
 * frequencyS (int): frequency to execute the script in seconds
 * scripts (string): the actual PxL script to execute
 * addExcludes (optional boolean, `false` by default): add pod and namespace excludes to the custom script

[This tutorial](https://docs.pixielabs.ai/tutorials/integrations/otel/#write-the-pxl-script) explains how to write custom PxL scripts. Example of a custom script, eg. `/scripts/custom1.yaml`:
```
name: "custom1"
description: "Custom script 1"
frequencyS: 60
script: |
    import px

    df = px.DataFrame(table='http_events', start_time=px.plugin.start_time)

    ns_prefix = df.ctx['namespace'] + '/'
    df.container = df.ctx['container_name']
    df.pod = px.strip_prefix(ns_prefix, df.ctx['pod'])
    df.service = px.strip_prefix(ns_prefix, df.ctx['service'])
    df.namespace = df.ctx['namespace']

    df.status_code = df.resp_status

    df = df.groupby(['status_code', 'pod', 'container','service', 'namespace']).agg(
        latency_min=('latency', px.min),
        latency_max=('latency', px.max),
        latency_sum=('latency', px.sum),
        latency_count=('latency', px.count),
        time_=('time_', px.max),
    )

    df.latency_min = df.latency_min / 1000000
    df.latency_max = df.latency_max / 1000000
    df.latency_sum = df.latency_sum / 1000000

    df.cluster_name = px.vizier_name()
    df.cluster_id = px.vizier_id()
    df.pixie = 'pixie'

    px.export(
    df, px.otel.Data(
        resource={
        'service.name': df.service,
        'k8s.container.name': df.container,
        'service.instance.id': df.pod,
        'k8s.pod.name': df.pod,
        'k8s.namespace.name': df.namespace,
        'px.cluster.id': df.cluster_id,
        'k8s.cluster.name': df.cluster_name,
        'instrumentation.provider': df.pixie,
        },
        data=[
        px.otel.metric.Summary(
            name='http.server.duration',
            description='measures the duration of the inbound HTTP request',
            # Unit is not supported yet
            # unit='ms',
            count=df.latency_count,
            sum=df.latency_sum,
            quantile_values={
            0.0: df.latency_min,
            1.0: df.latency_max,
            },
            attributes={
            'http.status_code': df.status_code,
            },
        )],
    ),
    )
```

The `addExcludes` adds a filtering section to the script right before the call to `px.export`. For example if `EXCLUDE_PODS_REGEX` is set to `team-1-.*`, the following line will be added to the PxL script:

```
df = df[not px.regex_match('team-1-.*', df.pod)]
```

 The filtering code requires your PxL script to:
   * Use `df` as the exported dataframe
   * The dataframe `df` to have a `pod` field (for `EXCLUDE_PODS_REGEX`)
   * The dataframe `df` to have a `namespace` field (for `EXCLUDE_NAMESPACE_REGEX`)

If the above requirements do not hold for your script, setting `addExcludes` to `true` will break the script.


## Script registration behaviour

The integration will consider scripts that start with `nri-` as managed by the integration. Scripts are registered per cluster and follow the `nri-<script name>-<cluster name>` pattern. The integration updates the scripts to bring them in-sync with the provided configuration. Scripts that are no longer present in the configuration are deleted. If you already have scripts that start with `nri-`, the integration will remove these if they are not specified in the integration configuration. 

A cluster name can only be used for a single cluster. Scripts that follow the `nri-<script name>-<cluster name>` pattern with the same `<cluster-name>` but that were registered from another cluster will be overwritten or deleted.

## Support

New Relic hosts and moderates an online forum where customers can interact with New Relic employees as well as other users to get help and share best practices. Like all official New Relic open source projects, there's a related Community topic in the New Relic Explorers Hub. You can find this project's topic/threads here: https://discuss.newrelic.com/t/new-relic-pixie-integration/143646

## Contribute

We encourage your contributions to improve the Pixie integration! Keep in mind that when you submit your pull request, you'll need to sign the CLA via the click-through using CLA-Assistant. You only have to sign the CLA once per project.

If you have any questions, or to execute our corporate CLA (which is required if your contribution is on behalf of a company), drop us an email at opensource@newrelic.com.

**A note about vulnerabilities**

As noted in our [security policy](../../security/policy), New Relic is committed to the privacy and security of our customers and their data. We believe that providing coordinated disclosure by security researchers and engaging with the security community are important means to achieve our security goals.

If you believe you have found a security vulnerability in this project or any of New Relic's products or websites, we welcome and greatly appreciate you reporting it to New Relic through [HackerOne](https://hackerone.com/newrelic).

If you would like to contribute to this project, review [these guidelines](./CONTRIBUTING.md).

To all contributors, we thank you!  Without your contribution, this project would not be what it is today.

## License
The New Relic Pixie integration is licensed under the [Apache 2.0](http://apache.org/licenses/LICENSE-2.0.txt) License.

The integration also uses source code from third-party libraries. You can find full details on which libraries are used and the terms under which they are licensed in the third-party notices document.
