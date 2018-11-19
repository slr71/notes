# VerSSA Reverse Proxy Setup

- Create a single new instance for haproxy.
- Create DNS entries for the Kubernetes worker nodes and for the haproxy node.
- Configure haproxy to use name-based routing to balance loads amongst the three haproxy nodes.
- I'm only going to set up proxying for `verssa-r.org` initially. I still need to read up on VICE to figure out how to
  set it up.
