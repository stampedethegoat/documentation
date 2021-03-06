---
title: Log Collection & Integrations
kind: Documentation
description: "Configure your Datadog Agent to gather logs from your host, containers, and services."
aliases:
  - /logs/faq/how-to-send-logs-to-datadog-via-external-log-shippers
---

## Getting started with the Agent

Log collection requires an Agent version >= 6.0. Older versions of the Agent do not include the `Log collection` interface that is used for log collection.

If you are not using it already, please follow [the Agent installation instruction][1].

Collecting logs is **disabled** by default in the Datadog Agent, you need to enable it in `datadog.yaml`:

```
logs_enabled: true
```

The Datadog Agent sends its logs to Datadog over TLS-encrypted TCP. This requires outbound communication over port `10516`.

## Enabling log collection from integrations
To start collecting logs for a given integration, uncomment the logs section in that integration's yaml file and configure it for your environment.

<div class="alert alert-warning">
Not all integrations include out of the box log configurations.  <a href="https://docs.datadoghq.com/integrations/#cat-log-collection">Consult the current list of supported integrations available</a>.
</div>

If an integration does not support logs by default, use the custom file configuration below.

## Custom log collection

The Datadog Agent v6 can collect logs from files or the network (TCP or UDP) and forward them to Datadog. To configure this, create a new repository and yaml file named after your log source  in the Agent's **conf.d** directory ( `conf.d/python.d/conf.yaml` for python logs, ...) and set these options:

* `type` : (mandatory) type of log input source (**tcp** / **udp** / **file**)
* `port` / `path` : (mandatory) Set `port` if `type` is **tcp** or **udp**. Set `path` if `type` is **file**.
* `service` : (mandatory) name of the service owning the log
* `source` : (mandatory) attribute that defines which integration is sending the logs. "If the logs do not come from an existing integration then this field may include a custom source name. But we recommend matching this value to the namespace of any related [custom metrics][2] you are collecting, e.g, `myapp` from `myapp.request.count`)"
* `sourcecategory` : (optional) Multiple value attribute. Can be used to refine the source attribtue. Example: source:mongodb, sourcecategory:db_slow_logs
* `tags`: (optional) add tags to each log collected.

## Tail existing files
Set `type` to **file** then specify the absolute `path` to the log file you want to tail.

Example:
To gather python applications stored in **/var/log/myapp1.log** and **/var/log/python.log** create a `python.d/conf.yaml` file as follows:

```yaml
##Log section
logs:

  - type: file
    path: /var/log/myapp1.log
    service: myapp1
    source: python
    sourcecategory: sourcecode
    tags: env:prod

  - type: file
    path: /var/log/python.log
    service: myapplication
    source: python
    sourcecategory: sourcecode
```
* [Restart your Agent][3]


**Note**: If you are using the Windows 6 Agent and trailing files for logs - make sure that those files have a UTF8 encoding.

## Stream logs through TCP/UDP
Set `type` to **tcp** or **udp** depending on the protocol then specify the `port` of your incoming connection.

Example:
If your PHP application does not log to a file, but instead forwards its logs via TCP, create a configuration file that specifies the port to receive as in the example below:

```yaml

##Log section
logs:
  - type: tcp
    port: 10518
    service: webapp
    source: php
    sourcecategory: front

```
* [Restart your Agent][3]

The Agent supports raw string, JSON, and Syslog formated logs. If you are sending logs in batch, use break line characters to separate your logs.

## Advanced log collection functions

### Filter logs

All logs are not equal and you may want to send only a specific subset of logs to Datadog.
To achieve this use the `log_processing_rules` parameter in your configuration file with the **exclude_at_match** or **include_at_match** `type`.

* **exclude_at_match**: If the pattern is contained in the message the log is excluded, and not sent to Datadog.
  Example: Filtering out logs that contain a Datadog email

```yaml

logs:
  - type: file
    path: /my/test/file.log
    service: cardpayment
    source: java
    log_processing_rules:
    - type: exclude_at_match
      name: exclude_datadoghq_users
      ## Regexp can be anything
      pattern: \w+@datadoghq.com
```

* **include_at_match**: Only log with a message that includes the pattern are sent to Datadog.
  Example: Sending only logs that contain a Datadog email

```yaml

logs:
  - type: file
    path: /my/test/file.log
    service: cardpayment
    source: java
    log_processing_rules:
    - type: include_at_match
      name: include_datadoghq_users
      ## Regexp can be anything
      pattern: \w+@datadoghq.com
```

**Note**: If you set up multiple processing rules, they will be applied sequentially.
Each rule will be applied on the result of the previous one.

### Scrub sensitive data in your logs

If your logs contain sensitive information that you wish to redact, configure the Datadog Agent to scrub sensitive sequences by using the `log_processing_rules` parameter in your configuration file with the **mask_sequences** `type`.

This replaces all matched groups with `replace_placeholder` parameter value.
Example: Redact credit card numbers

```yaml

logs:
 - type: file
   path: /my/test/file.log
   service: cardpayment
   source: java
   log_processing_rules:
      - type: mask_sequences
        name: mask_credit_cards
        replace_placeholder: "[masked_credit_card]"
        ##One pattern that contains capture groups
        pattern: (?:4[0-9]{12}(?:[0-9]{3})?|[25][1-7][0-9]{14}|6(?:011|5[0-9][0-9])[0-9]{12}|3[47][0-9]{13}|3(?:0[0-5]|[68][0-9])[0-9]{11}|(?:2131|1800|35\d{3})\d{11})
```

### Multi-line aggregation

If your logs are not sent in JSON and you want to aggregate several lines into one single entry, configure the Datadog Agent to detect a new log using a specific regex pattern instead of having one log per line.

This is accomplished by using the `log_processing_rules` parameter in your configuration file with the **multi_line** `type`.

This aggregates all lines into one single entry until the given pattern is detected again. This is especially useful for database logs and stack traces.
Example: Every java log line starts with a timestamp with `yyyy-dd-mm` format. The below lines including a stack trace would be sent as two logs.

```
2018-01-03T09:24:24.983Z UTC Exception in thread "main" java.lang.NullPointerException
        at com.example.myproject.Book.getTitle(Book.java:16)
        at com.example.myproject.Author.getBookTitles(Author.java:25)
        at com.example.myproject.Bootstrap.main(Bootstrap.java:14)
2018-01-03T09:26:24.365Z UTC starting upload of /my/file.gz

```

To achieve this, you need to use the following `log_processing_rules`:

```yaml

logs:
 - type: file
   path: /var/log/pg_log.log
   service: database
   source: postgresql
   log_processing_rules:
      - type: multi_line
        name: new_log_start_with_date
        pattern: \d{4}\-(0?[1-9]|1[012])\-(0?[1-9]|[12][0-9]|3[01])
```

More examples:

| **Raw string**           | **Pattern**                                |
| :---                     | :----                                      |
| 14:20:15                 | `\d{2}:\d{2}:\d{2}`                        |
| 11/10/2014               | `\d{2}\/\d{2}\/\d{4}`                      |
| Thu Jun 16 08:29:03 2016 | `\w{3}\s+\w{3}\s+\d{2}\s\d{2}:\d{2}:\d{2}` |
| 20180228                 | `\d{8}`                                    |

### Tail multiple directories or whole directories by using wildcards

If your log files are labeled by date or all stored in the same directory, configure your Datadog Agent to monitor them all and automatically detect new ones by using wildcards in the `path` attribute.

* Using `path: /var/log/myapp/*.log`:
  * Matches all `.log` file contained in the `/var/log/myapp/` directory.
  * Doesn't match `/var/log/myapp/myapp.conf`.

* Using `path: /var/log/myapp/*/*.log`:
  * Matches `/var/log/myapp/log/myfile.log`.
  * Matches `/var/log/myapp/errorLog/myerrorfile.log`
  * Doesn't match `/var/log/myapp/mylogfile.log`.

Configuration example:

```yaml

logs:
 - type: file
   path: /var/log/myapp/*.log
   service: mywebapp
   source: go
```

**Note**: The Agent requires read and execute permissions (5) on the directory to be able to list all the available files in it.

### Using a Proxy for Logs

Logs make use of a different set of proxy settings than other data types forwarded by the Datadog Agent. This is due to logs presently being transported over TCP/SSL, while other features submit data via on HTTPS.

To configure your Datadog Agent to forward logs through a proxy server, add these settings to the `datadog.yaml` configuration file:"

```
logs_config:
  dd_url: <MY_PROXY_URL>
  dd_port: <MY_PROXY_PORT>
  dev_mode_no_ssl: true
```

Then configure your proxy to forward logs to the endpoint `agent-intake.logs.datadoghq.com` on port `10516` with SSL activated. 

[Refer to our Agent proxy documentation page to learn how to forward your metrics with a proxy][8].

### The Advantage of Collecting JSON-formatted logs

Datadog automatically parses JSON-formatted logs. For this reason, when you have control over the log format you send to Datadog, we encourage you to format them as JSON to avoid the need for custom parsing rules.

## Reserved attributes

If your logs are formatted as JSON, be aware that some attributes are reserved for use by Datadog:

### *date* attribute

By default Datadog generates a timestamp and appends it in a date attribute when logs are received.
However, if a JSON formatted log file includes one of the following attributes, Datadog interprets its value as the the log’s official date:

* `@timestamp`
* `timestamp`
* `_timestamp`
* `Timestamp`
* `eventTime`
* `date`
* `published_date`
* `syslog.timestamp`

You can also specify alternate attributes to use as the source of a log's date by setting a [log date remapper processor][4]

**Note**: Datadog rejects a log entry if its official date is older than 6 hours in the past.

<div class="alert alert-info">
The recognized date formats are: <a href="https://www.iso.org/iso-8601-date-and-time-format.html">ISO8601</a>, <a href="https://en.wikipedia.org/wiki/Unix_time">UNIX (the milliseconds EPOCH format)</a>, and <a href="https://www.ietf.org/rfc/rfc3164.txt">RFC3164</a>.
</div>

### *message* attribute

By default, Datadog ingests the value of message as the body of the log entry. That value is then highlighted and displayed in the [logstream][5], where it is indexed for [full text search][6].

### *status* attribute

Each log entry may specify a status level which is made available for faceted search within Datadog. However, if a JSON formatted log file includes one of the following attributes, Datadog interprets its value as the the log’s official status:

* `severity`
* `level`
* `syslog.severity`

If you would like to remap some status existing in the `status` attribute, you can do so with the [log status remapper][7]

### *host* attribute

Using the Datadog Agent or the RFC5424 format automatically sets the host value on your logs. However, if a JSON formatted log file includes the following attribute, Datadog interprets its value as the the log’s host:

* `syslog.hostname`

### *service* attribute

Using the Datadog Agent or the RFC5424 format automatically sets the service value on your logs. However, if a JSON formatted log file includes the following attribute, Datadog interprets its value as the the log’s service:

* `syslog.appname`

### Edit reserved attributes

You can now control the global hostname, service, timestamp, and status main mapping that are applied before the processing pipelines. This is particularly helpful if logs are sent in JSON or from an external Agent.

{{< img src="logs/log_collection/reserved_attribute.png" alt="Reserved Attribute" responsive="true" popup="true" style="width:80%;">}}

To change the default values for each of the reserved attributes, go to the pipeline page and edit the `Reserved Attribute mapping`:

{{< img src="logs/log_collection/reserved_attribute_tile.png" alt="Reserved Attribute Tile" responsive="true" popup="true" style="width:80%;">}}

[1]: /agent
[2]: /getting_started/custom_metrics/
[3]: /agent/faq/agent-commands/#start-stop-restart-the-agent
[4]: /logs/processing/#log-date-remapper
[5]: /logs/explore/#logstream
[6]: /logs/explore/#search-bar
[7]: /logs/processing/#log-status-remapper
[8]: /agent/proxy
