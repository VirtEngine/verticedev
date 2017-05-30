# Writing an API

The REST api needs to be changed for 2.0 since we'll no longer use the HMAC based approach. We'll rather use BasicAuth (username/password) or AuthenticationToken.

# Solution Overview

We deal with the changes needed for the API to work with 2.0. We are go over one api `marketplaces` here.

We will use the rubygem [kubeclient](https://github.com/abonas/kubeclient). Please remove **megam_api** and include `kubeclient` in the Gemfile of 2.0 code.

### nilavu.conf

The `http` url will have the value `http://localhost:8080`.

## Initialize the client:

```
client = Kubeclient::Client.new('http://localhost:8080/api/', "v1")

```

Or without specifying version (it will be set by default to "v1")

```
client = Kubeclient::Client.new('http://localhost:8080/api/')

```
            *------------------------*
            | models/api/marketplaces|
            *------------------------*
                      |
                      V
                      *------------------------*
                      | models/api_dispatcher  |
                      *------------------------*

## ApiDispatcher

We'll not use `Kubeclient` directly but rather change   `ApiDispatcher` and the layer(`models/api/marketplaces`) that calls the `ApiDispatcher` with the correct method names eg: `get_pods`.

There apis have two prefixes which we need to support

`/api`
`/oapi`

The apis that need to do that will be `templates`

- By default `ApiDispatcher` will use `/api`  prefix.

- `ApiDispatcher` will be modified to use `Kubeclient`

- The layer(`models/api/marketplaces`) will send the prefix `oapi` as per the api requirement.

- The layer (`models/api/marketplaces`) will pass back to the `scrubber`.

- The `honeypot` methods that load the data needs to be modified.

```ruby
# groups with cattypes (Collaboration...)
  def self.cached_marketplace_groups(params)
    Rails.cache.fetch("honeypot", race_condition_ttl: 20) do
      Api::Marketplaces.instance.list(params).marketplace_groups
    end
  end

  # groups with settings_name (vertice, bitnami, containership...)
  # race_condition_ttl is needed  (RTFM - https://github.com/megamsys/nilavu/issues/943)
  def self.cached_marketplace_bifurs(params)
    Rails.cache.fetch("honeypot_bifurs", race_condition_ttl: 20) do
      Api::Marketplaces.instance.list(params).marketplace_bifurs
    end
  end
```

The `.marketplace_groups` needs to retrofit to 2.0

```ruby

Api::Marketplaces.instance.list(params).marketplace_groups

```

In you case you need debug the same way to modify the data output of an API.
