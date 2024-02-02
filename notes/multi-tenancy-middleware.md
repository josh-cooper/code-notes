---
title: Rails Multi-tenancy Middleware
tags:
  - rails
  - multi-tenancy
emoji: 👋
link: https://zander.wtf
modified: 2021-04-23
---

Implementing multi-tenancy in a Rails or Rack application can significantly enhance its architecture by allowing a single instance of the application to serve multiple tenants. This implementation involves adding middleware to handle tenant switching based on request parameters and ensuring isolation between tenants' data. Below are the details of adding a custom middleware for tenant switching in a Rails + Grape application.

### Tenant Switcher Service
The `Tenants::Switcher` service is responsible for switching the database schema search path to the tenant's schema, isolating tenant data:

```ruby
module Tenants
  class TenantNotFound < StandardError; end

  class Switcher
    class << self
      def switch!(tenant_name)
        set_search_path(tenant_name)
      rescue StandardError
        raise TenantNotFound, "One of the following schema(s) is invalid: #{tenant_name}"
      end

      private

      def set_search_path(tenant_name)
        ActiveRecord::Base.connection.schema_search_path = tenant_name
      end
    end
  end
end
```

### Tenant Middleware
A new middleware `Tenants::Middleware` is created to parse the tenant from the incoming request, switch to the appropriate tenant, and handle cases where the tenant cannot be found:

```ruby
# app/services/tenants/middleware.rb

module Tenants
  class Middleware
    def initialize(app)
      @app = app
    end

    def call(env)
      tenant = parse_tenant(env) || raise(Tenants::TenantNotFound)
      Tenants::Switcher.switch!(tenant)
      @app.call(env)
    rescue Tenants::TenantNotFound => e
      raise(e) if Rails.application.config.show_error_on_tenant_not_found
      [404, {"Content-Type" => "text/html"}, [Rails.public_path.join("404.html").read.to_s]]
    end

    private

    def parse_tenant(env)
      request = Rack::Request.new(env)
      token = env['grape_jwt_auth.parsed_token']
      default_tenant_for_health_check(request) || tenant_from_token(token) || tenant_from_env || raise_not_found(token)
    end

    # Implementation details omitted for brevity
  end
end
```

### Update to Grape API Class
Finally, the Grape API class is updated to use the newly defined middleware:

```ruby
# app/api/ready_admissions/api.rb

class API < Grape::API
  # Existing code omitted for brevity
  include Grape::Jwt::Authentication
  use Tenants::Middleware
end
```

This setup enables the application to dynamically switch tenants based on the request, effectively implementing multi-tenancy at the middleware level. It's important to handle errors gracefully, such as when a tenant is not found, to maintain a good user experience and application stability.
