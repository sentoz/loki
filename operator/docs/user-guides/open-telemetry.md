---
title: "OpenTelemetry / OTLP"
description: ""
lead: ""
date: 2024-10-25T12:43:23+02:00
lastmod: 2024-10-25T12:43:23+02:00
draft: false
images: []
menu:
  docs:
    parent: "user-guides"
weight: 100
toc: true
---

## Introduction

Loki 3.0 introduced an API endpoint using the OpenTelemetry Protocol (OTLP) as a new way of ingesting log entries into Loki. This endpoint is an addition to the standard Push API that was available in Loki from the start.

Because OTLP is not specifically geared towards Loki but is a standard format, it needs additional configuration on Loki's side to map the OpenTelemetry data format to Loki's data model.

Specifically, OTLP has no concept of "stream labels" or "structured metadata". Instead, OTLP provides metadata about a log entry in _attributes_ that are grouped into three buckets (resource, scope and log), which allows setting metadata for many log entries at once or just on a single entry depending on what's needed.

## Prerequisites

Log ingestion using OTLP depends on structured metadata being available in Loki. This capability was introduced with schema version 13, which is available in Loki Operator when using `v13` in a schema configuration entry.

If you are creating a new `LokiStack`, make sure to set `version: v13` in the storage schema configuration.

If there is an existing schema configuration, a new schema version entry needs to be added, so that it becomes active in the future (see [Upgrading Schemas](loki-upgrading-schemas) in the Loki documentation).

```yaml
# [...]
spec:
  storage:
    schemas:
    - version: v13
      effectiveDate: 2024-10-25
```

Once the `effectiveDate` has passed your `LokiStack` will be using the new schema configuration and is ready to store structured metadata.

## Attribute Mapping

Loki splits the configuration for mapping OTLP attributes to stream labels and structured metadata into two places:

- `default_resource_attributes_as_index_labels` in the [`distributor` configuration](loki-docs-distributor-config)
- `otlp_config` in the `limits_config` (see [Loki documentation](loki-docs-limits-config))

By default, `default_resource_attributes_as_index_labels` provides a set of resource-attributes that are mapped to stream-labels on the Loki side.

As the field in the distributor configuration is limited to resource-level attributes and can only produce stream-labels as an output, the `otlp_config` needs to be used to map resource, scope or log level attributes to either be dropped or saved as structured metadata.

The Loki Operator does not use the same approach for configuring the attributes as Loki itself does. The most visible difference is that there is no distinction between the `distributor` and `limits` configuration in the Operator.

Instead, the Loki Operator only uses the `limits` configuration for all its attributes. The structure of the `limits` configuration also differs from the structure in the Loki configuration file. See [Custom Attribute Mapping](#custom-attribute-mapping) below for an explanation of the configuration options available in the Operator.

### Picking Stream Labels and Structured Metadata

Whether you choose to map an attribute to a stream label or to structured metadata depends on what data is present in the attribute. Stream labels are used to identify a set of log entries "belonging together" in a stream of events. These labels are used for indexing and identifying the streams and so should not contain information that changes between different log entries of the same application. They should also not contain values that have a high number of different values ("cardinality").

Structured metadata on the other hand is just saved together with the log entries and only read when querying for logs, so it's more suitable to store "any data". Both stream labels and structured metadata can be used to filter log entries during a query.

If the OpenTelemetry data sent to Loki includes attributes that should not be stored, it is also possible to "drop" them from the entry before storing them into Loki. This is possible for all types of attributes (resource, scope and log).

**Note:** Attributes that are not explicitly set as stream labels or dropped from the entry are saved as structured metadata by default.

### Loki Operator Defaults

When using the Loki Operator the default attribute mappings depend on the [tenancy mode]({{< ref "api.md#loki-grafana-com-v1-ModeType" >}}) used for the `LokiStack`:

- `static` and `dynamic` use the Grafana defaults
- `openshift-logging` uses OpenShift defaults

### Custom Attribute Mapping

All tenancy modes support customization of the attribute mapping configuration. This can be done globally (for all tenants) or on a per-tenant basis. When a custom attribute mapping configuration is defined, then the Grafana defaults are not used. If the default labels are desired as well, they need to be added to the custom configuration. See also the section about [Customizing OpenShift Defaults](#customizing-openshift-defaults) below.

**Note:** A major difference between the Operator and Loki is how it handles inheritance. Loki by default only copies the attributes defined in the `default_resource_attributes_as_index_labels` setting to the tenants whereas the Operator will copy all global configuration into every tenant.

The attribute mapping configuration in `LokiStack` is done through the limits configuration:

```yaml
# [...]
spec:
  limits:
    global:
      otlp: {} # Global OTLP Attribute Configuration
    tenants:
      example-tenant:
        otlp: {} # OTLP Attribute Configuration for tenant "example-tenant"
```

Both global and per-tenant OTLP configurations can map attributes to stream-labels. At least _one stream-label_ is needed for successfully saving a log entry to Loki storage, so the configuration should account for that.

Stream labels can only be generated from resource-level attributes, which is mirrored in the data structure of the `LokiStack` resource:

```yaml
# [...]
spec:
  limits:
    global:
      otlp:
        streamLabels:
          resourceAttributes:
          - name: "k8s.namespace.name"
          - name: "k8s.pod.name"
          - name: "k8s.container.name"
```

The "drop" configuration for dropping attributes from the log entry is possible for all three types of attributes (resource, scope and log):

```yaml
# [...]
spec:
  limits:
    global:
      otlp:
        streamLabels:
          # [...]
        drop:
          resourceAttributes:
          - name: "process.command_line"
          - name: "k8s\\.pod\\.labels\\..+"
            regex: true
          scopeAttributes:
          - name: "service.name"
          logAttributes:
          - name: "http.route"
```

The previous example also shows that the attribute names can be expressed as _regular expressions_ by setting `regex: true`.

Using a regular expression makes sense when there are many attributes with similar names for which a configuration should apply. It is not recommended to use regular expressions for stream labels, as it can potentially apply to a lot of attributes and create lots of labels.

### Customizing OpenShift Defaults

The `openshift-logging` tenancy mode contains its own set of default attributes. Some of these attributes (called "required attributes") can not be removed by applying a custom configuration, because they are needed for other OpenShift components to function properly. Other attributes (called "recommended attributes") are provided but can be disabled in case they influence performance negatively. The complete set of attributes is documented in the [data model](rhobs-data-model) repository. The default configuration only lists the stream labels of the data model, because the mapping to structured metadata is Loki's default.

Because the OpenShift attribute configuration is applied based on the tenancy mode, the simplest configuration is to just set the tenancy mode and not apply any custom attributes. This will provide instant compatibility with the other OpenShift tools.

In case additional stream labels are needed, the normal custom attribute configuration mentioned above can be used. Attributes defined in the custom configuration will be **merged** with the default configuration. The same approach can be used to drop attributes, as long as they are not part of the set of required attributes.

#### Removing Recommended Attributes

In case of issues with the default set of attributes, there is a way to slim down the default set of attributes applied to a LokiStack operating in `openshift-logging` tenancy mode:

```yaml
# [...]
spec:
  tenants:
    mode: openshift-logging
    openshift:
      otlp:
        disableRecommendedAttributes: true # Set this to remove recommended attributes
```

Setting `disableRecommendedAttributes: true` reduces the set of default stream labels to only the "required attributes". This means the following "recommended attributes" are **not** automatically used as stream labels:

- `k8s.container.name`
- `k8s.cronjob.name`
- `k8s.daemonset.name`
- `k8s.deployment.name`
- `k8s.job.name`
- `k8s.node.name`
- `k8s.pod.name`
- `k8s.statefulset.name`
- `kubernetes.container_name`
- `kubernetes.host`
- `kubernetes.pod_name`
- `service.name`

Since these attributes are no longer stream labels by default, queries that previously relied on them **as stream labels** will need to be updated, and their performance might be negatively affected. It is recommended to combine this setting with a [custom attribute mapping](#custom-attribute-mapping) to selectively reintroduce any of these attributes (or other) as stream labels if they are important for your querying strategy.

## References

- [Loki Labels](loki-labels)
- [Structured Metadata](loki-structured-metadata)
- [OpenTelemetry Attribute](otel-attributes)
- [OpenShift Default Attributes](rhobs-data-model)

[loki-docs-distributor-config]: https://grafana.com/docs/loki/latest/configure/#distributor
[loki-docs-limits-config]: https://grafana.com/docs/loki/latest/configure/#limits_config
[loki-labels]: https://grafana.com/docs/loki/latest/get-started/labels/
[loki-structured-metadata]: https://grafana.com/docs/loki/latest/get-started/labels/structured-metadata/
[loki-upgrading-schemas]: https://grafana.com/docs/loki/latest/configure/storage/#upgrading-schemas
[otel-attributes]: https://opentelemetry.io/docs/specs/otel/common/#attribute
[rhobs-data-model]: https://github.com/rhobs/observability-data-model/blob/main/cluster-logging.md#attributes
