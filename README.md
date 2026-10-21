# Kedify

A Codex plugin for Kubernetes CPU and memory recommendations with kedr and
read-only access to Kedify organizations, clusters, agents, savings, and
autoscaling diagnostics through MCP.

## How to use it

Add the GitHub repository as a marketplace, then install the Kedify plugin:

```sh
codex plugin marketplace add jkremser/skills
codex plugin add kedify@kedify
```

The first command adds the `kedify` marketplace. The second installs the
`kedify` plugin from that marketplace.

When prompted to connect the MCP server, choose **Kedify Dashboard** and sign in
with your existing dashboard account. Start a new Codex session after
installation to use the bundled tools and skills.
