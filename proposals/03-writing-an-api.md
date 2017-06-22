# Writing an API

The REST api needs to be changed for 2.0 since we'll no longer use the HMAC based approach. We'll rather use BasicAuth (username/password) or AuthenticationToken as [detailed in ](https://github.com/megamsys/verticedev/blob/master/proposals/02-multitenant-authentication.md)

# Solution Overview

Here elaborate the changes needed for the API to work with 2.0. We go over one api which is `marketplaces`.

We will use the rubygem [kubeclient](https://github.com/abonas/kubeclient). 

Please remove **megam_api** and include `kubeclient` in the Gemfile of 2.0 code.

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
## Marketplace 

Every models gets loaded using an API fascade.

            *------------------------*
            | models/api/marketplaces|
            *------------------------*
                      |
                      V
                      *------------------------*
                      | models/api_dispatcher  |
                      *------------------------*

## Changes in ApiDispatcher

We'll not use `Kubeclient` directly but rather include a module which will build the machinery to call the client. 

### APIMachinery 

The apis will have two prefixes to support

`/api`
`/oapi`

Hence we need a module something like this.

```
module APIMachinery

  # form a group of api that will need /oapi prefix.
  # i forgot, but there is a better way to do this in ruby
  OAPI_GROUP = ['template', ]...
  
  ## returns KubeClient where 
  ##         fn is a function like Pods, Templates
  ##         action is get_pods, get_templates
  def client(fn,action)
  ## raise error if there are no required parms
  raise Nilavu::InvalidParameters unless found_required

  ## build the kubeclient with the  fn_url
  end
  
  def fn_url(fn)
    raise Nilavu::InvalidParameters unless fn
    
    return '/oapi' OAPI_GROUP.include?(fn) || 'api'
 end

  ## figure out if the required parms are present 
  ## split it mulitple methods if there are more to do.
  def found_required
    
  end
end


```


- By default `ApiMachinery` will set `/api` as prefix.

- `ApiDispatcher` will be modified to include `ApiMachinery`

- `ApiDispatcher` will initialze a `client` with the `fn`

- `ApiDispatcher` will use `client` and call the `action`

- Upon completion from the `ApiDispatcher` the `Kube::xxx` ruby object will be passed back to the layer (`models/api/marketplaces`) which will **convert as appropriate** to the its `scrubber`.

For **marketplaces**  we have another change to make

- The `honeypot` methods that load the data needs to be modified.



