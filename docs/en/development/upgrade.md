---
created: "2025-2-11"
title: Upgrading the GitLab Version
---

## Prerequisites

1. The new version image has been successfully built.
2. The Helm chart has been upgraded and has passed all integration tests.

## Upgrade Steps

1. Update the tool version: Replace the previous version of the tool with the latest version.
2. Update submodules:

    ```bash
    make update-submodule
    ```

3. Commit your code and build the operator image for testing
