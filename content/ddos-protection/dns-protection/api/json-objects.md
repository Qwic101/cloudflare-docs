---
title: JSON object
pcx_content_type: reference
weight: 3
meta:
  title: Advanced DNS Protection API - JSON object
---

# JSON object

This page contains an example of the DNS protection rule JSON object used in the API.

```json
{
  "id": "31c70c65-9f81-4669-94ed-1e1e041e7b06",
  "scope": "region",
  "name": "WEUR",
  "mode": "monitoring",
  "profile_sensitivity": "medium",
  "rate_sensitivity": "medium",
  "burst_sensitivity": "medium",
  "created_on": "2023-10-01T13:10:38.762503+01:00",
  "modified_on": "2023-10-01T13:10:38.762503+01:00"
}
```

The `scope` field value must be one of `global`, `region`, or `datacenter`. You must provide a region code (or data center code) in the `name` field when specifying a `region` (or `datacenter`) scope.

The `mode` value must be one of `enabled`, `disabled`, or `monitoring`.

The `profile_sensitivity` field value must be one of `low` (default), `medium`, `high`, or `very_high`.

The `rate_sensitivity` and `burst_sensitivity` field values must be one of `low`, `medium`, or `high`.

For more information on the rule settings, refer to [Rule settings](/ddos-protection/dns-protection/rule-settings/).