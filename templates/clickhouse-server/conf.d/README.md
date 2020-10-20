### Files located in this directory can be included in the main config in the following way:

```
    <${config_option} incl="${file_name_without_xml}" />
```

### Example for including 'clickhouse_remote_servers.xml' in 'config.xml'

```
...
    <remote_servers incl="clickhouse_remote_servers" />
...
```
