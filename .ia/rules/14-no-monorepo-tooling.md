# No Monorepo Tooling

Never suggest npm workspaces, Turborepo, Nx, or any monorepo tooling in this project.

These tools hoist `node_modules` to the root and create tooling-level coupling, which directly contradicts the independence principle: every service must be extractable into its own repo without changes.

The `file:` reference approach for shared-libs is the intentional alternative — each service installs shared-libs locally as if they were external packages.
