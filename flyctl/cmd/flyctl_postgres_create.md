# _flyctl postgres create_

Create a postgres cluster

### About

Create a postgres cluster

### Usage
```
flyctl postgres create [flags]
```

### Options

```
  -h, --help                  help for create
      --name string           the name of the new app
      --organization string   the organization that will own the app
      --password string       the superuser password. one will be generated for you if you leave this blank
      --region string         the region to launch the new app in
      --vm-size string        the size of the VM
      --volume-size string    the size in GB for volumes
```

### Global Options

```
  -t, --access-token string   Fly API Access Token
  -j, --json                  json output
  -v, --verbose               verbose output
```

### See Also

* [flyctl postgres](/docs/flyctl/postgres/)	 - Manage postgres clusters

