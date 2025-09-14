# Barman Cloud Plugin

This contains the manifest for the [Barman Cloud CNPG-I backup plugin](https://github.com/cloudnative-pg/plugin-barman-cloud), which replaced builtin barman cloud support.

The manifest has been modified to use this `database` namespace instead of the default `cnpg-system`.

Using the raw manifest is a temporary measure until [the helm chart is available](https://github.com/cloudnative-pg/charts/pull/598).
