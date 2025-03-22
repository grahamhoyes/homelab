# Foundational Infrastructure

This directory has specific infrastructure components which have no dependencies of their own, but are dependencies of other [controllers](/flux/infrastructure/controllers/) (which themselves are dependencies of [configs](/flux/infrastructure/configs/)).

At the moment, the only such case is the 1password operator. A few other operators require secrets at deploy time, which are created by `OnePasswordItem` resources. Hence, 1password must be able to at least install its CRDs before everything else.
