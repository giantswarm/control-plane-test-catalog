hello-world-app
---------------

A very barebones app for testing features of the Giant Swarm App Platform.

It contains just enough to deploy your first hello world website on your cluster.

## Configuration

In order to make this app work, you'll need to provide at least the `hostname` in your user-values.

```
hostname: helloworld.my-example-domain.com
```

By default, TLS is disabled, but can be enabled by setting

```
obtainTLSCertificate: true
```

in your user-values.
