# etcd-kubernetes-resources-count-exporter

This prometheus exporter is meant to connect to an etcd instance containing kubernetes data and exports the number of Kubernetes API resources.

## Known limitataions / TODOs

- This exporter has a very good memory. In case a namespace gets deleted, it will keep exporting the latest known resources for it unless it is restarted.
- Works on tenant clusters only without specifying custom settings through helm values.
