# VerSSA Reverse Proxy Setup

- Create a single new instance for haproxy.
- Create DNS entries for the Kubernetes worker nodes and for the haproxy node.
- Configure haproxy to use name-based routing to balance loads amongst the three haproxy nodes.
- I'm only going to set up proxying for `verssa-r.org` initially. I still need to read up on VICE to figure out how to
  set it up.

| Domain Name               | Type  | IP Address or Canonical Name |
| ------------------------- | ----- | ---------------------------- |
| haproxy.verssa-r.org      | A     | 172.28.1.241                 |
| k8s-worker-1.verssa-r.org | A     | 172.28.1.169                 |
| k8s-worker-2.verssa-r.org | A     | 172.28.1.146                 |
| k8s-worker-3.verssa-r.org | A     | 172.28.1.64                  |
| auth.verssa-r.org         | CNAME | haproxy.verssa-r.org         |
| de.verssa-r.org           | CNAME | haproxy.verssa-r.org         |
| vault.verssa-r.org        | CNAME | haproxy.verssa-r.org         |
| *.verssa-r.run            | CNAME | haproxy.verssa-r.org         |
