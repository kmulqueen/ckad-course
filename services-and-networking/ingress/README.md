# Ingress

## Ingress Controller

With a deployment of the nginx ingress image, a service to expose it, a configmap to feed nginx configuration data, and a serviceaccount with the right permissions to access all of these objects, we should be ready with an ingress controller in it's simplest form.

## Ingress Resource

A set of rules/configurations applied on the ingress controller. Create rules to route traffic based on different conditions. Example rules:

- forward all incoming traffic to a single location
- route traffic to different applications based on a URL
- route users based on domain name
  - example 2 domain names, 2 different rules (see ./ingress-resource-3.yaml)

Within each rule, you can handle different paths.
