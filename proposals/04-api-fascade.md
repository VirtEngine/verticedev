# API Fascade

The REST api needs to be changed for 2.0 since we'll no longer use the HMAC based approach. We'll rather use Authorization:Bearer token as [detailed here](https://github.com/megamsys/verticedev/blob/master/proposals/02-multitenant-authentication.md)

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

We'll not use `Kubeclient` directly but rather include a module which will build the machinery to call the client. 

### APIMachinery 

The apis will have two prefixes to support

`/api`
`/oapi`

Hence we need a module something like this.

```ruby
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

### VerticeResource

- `VerticeResource` will be modified to include `ApiMachinery`

- Initialize the client:

```
client = Kubeclient::Client.new('http://localhost:8080/api/', "v1")

```

Or without specifying version (it will be set by default to "v1")

```
client = Kubeclient::Client.new('http://localhost:8080/api/')

```

- `ApiDispatcher` will initialze the `fn`

- `ApiDispatcher` will use a modifier **vertice_resource** which will call the `client`/`action`


### APIMachinery 


```ruby


class ApiDispatcher
    include VerticeResource

    class ApiDispatcher::NotReached < StandardError
        def initialize(message)
            super(message)
        end

    end

    class ApiDispatcher::Flunked  < StandardError
        include HttpErrors

        def initialize(response)
            @response = response
        end

        def h401?
            return is_http_401?(@response)
        end

        def h403?
            return is_http_403?(@response)
        end

        def h404?
            return is_http_404?(@response)
        end
    end

    attr_accessor :megfunc, :megact, :parms, :swallow_404

    NAMESPACES           = 'Namespaces'.freeze
    PODS                 = 'Pods'.freeze

    #explore if there is a way to cut down these strings by dyanmically figuring out from
    ## "get_" +PODS.downcase
    GET_POD           = 'get_pod'.freeze
    CREATE_POD        = 'create_pod'.freeze

    def initialize(ignore_404 = false)
        @swallow_404 = ignore_404
    end

    def api_request(jlaz, jmethod, jparams, passthru=false )
              set_attributes(jlaz, jmethod, jparams, passthru)

        begin
            Rails.logger.debug "\033[01;35mFASCADE #{megfunc}::#{megact} \33[0;34m"

            raise Nilavu::InvalidParameters if !satisfied_args?(passthru, jparams)
            invoke_submit
        rescue Megam::API::Errors::ErrorWithResponse => m
                      raise_api_errors(ApiDispatcher::Flunked.new(m))
        rescue StandardError => se

            raise ApiDispatcher::NotReached.new(se.message)
        end
    end



    def meg_function(arg=nil)
        if arg != nil
            @megfunc = arg
        else
            @megfunc
        end
    end

    def meg_action(arg=nil)
        if arg != nil
            @megact = arg
        else
            @megact
        end
    end

    def parameters(arg=nil)
        if arg != nil
            @parms = arg
            @parms = @parms.merge({:host => endpoint })
        else
            @parms
        end
    end

    private

    def set_attributes(jlaz, jmethod, parms, passthru)
        meg_function(jlaz)
        meg_action(jmethod)
        parameters(parms)
    end

    ## I think you just want "Authorization Bearer:"
    def satisfied_args?(passthru, params={})
        unless passthru
            return params[:email] && params[:authorization_bearer].present?
        end
        return true
    end

    def endpoint
        GlobalSetting.http_api
    end

    def debug_print(jparams)
        jparams.each do |name, value|
            Rails.logger.debug("> #{name}: #{value}")
        end
    end

    def raise_api_errors(e)
        return if (e.h404? && swallow_404)
        raise e
    end
end
```

### nilavu.conf

The `http` url we'll will have the value `http://localhost:8080`.


- Upon completion from the `ApiDispatcher` the `Kube::xxx` ruby object will be passed back to the layer (`models/api/marketplaces`) which will **convert as appropriate** to the its `scrubber`.

