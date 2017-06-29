# API Fascade

The REST api needs to be changed for 2.0 with TLS to support HMAC based,  masterkey, OAuth.

# Solution Overview

Here elaborate the changes needed for the API fascade to work with 2.0. We go over one api which is `marketplaces`.

We will use the rubygem [kubeclient](https://github.com/abonas/kubeclient).

Please remove **megam_api** and include `kubeclient` in the Gemfile of 2.0 code.

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

We'll have a revamped `megam_api` ruby gem.


### VerticeResource

- `VerticeResource` is a rest resource that talks to our api server [boomi](https://gitlab.com/megamsys/boomi)

### APIDispatcher


- `ApiDispatcher` will initialze the `fn`

- `ApiDispatcher` will use the  **invoke_submit** which will call the `client`/`action`

### nilavu.conf

The `http` url will have the value `http://localhost:8080`


- Upon completion from the `ApiDispatcher` the `MegamAPI::xxx` ruby object will be passed back to the layer (`models/api/marketplaces`) which will **convert as appropriate** to the its `scrubber`.
