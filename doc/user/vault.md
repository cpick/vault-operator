# Vault Usage Guide

This document describes how to initialize, unseal, and interact with a deployed Vault cluster.

## Prerequisites

* [Vault Commands (CLI)][vault-cli] installed
* Before beginning, create the [example][example_vault] Vault cluster

## Initializing a Vault cluster

Initialize a new Vault cluster before performing any operations.

1. Configure port forwarding between the local machine and the first sealed Vault node:

    ```sh
    kubectl -n default get vault example -o jsonpath='{.status.vaultStatus.sealed[0]}' | xargs -0 -I {} kubectl -n default port-forward {} 8200
    ```

2. Open a new terminal.

3. Export the following environment for [Vault CLI environment][vault-cli-env]:

    ```sh
    export VAULT_ADDR='https://localhost:8200'
    export VAULT_SKIP_VERIFY="true"
    ```

4. Verify that the Vault server is accessible using the Vault CLI:

    ```sh
    $ vault status

    Error checking seal status: Error making API request.

    URL: GET https://localhost:8200/v1/sys/seal-status
    Code: 400. Errors:

    * server is not yet initialized
    ```

A response confirms that the Vault CLI is ready to interact with the Vault server. However, the output indicates that the Vault server is not initialized.

5. Initialize the Vault server to generate the unseal keys and the root token.

    See [Initializing the Vault][initialize-vault] on how to initialize a Vault cluster.

## Unsealing a sealed node

1. Configure port forwarding between the local machine and the first sealed Vault node:

    ```sh
    kubectl -n default get vault example -o jsonpath='{.status.vaultStatus.sealed[0]}' | xargs -0 -I {} kubectl -n default port-forward {} 8200
    ```

2. Open a new terminal.

3. Export the following environment for [Vault CLI environment][vault-cli-env]:

    ```sh
    export VAULT_ADDR='https://localhost:8200'
    export VAULT_SKIP_VERIFY="true"
    ```

4. Unseal the Vault node by using the unseal keys generated from the initialized Vault cluster.

    See [Seal/Unseal a Vault node][seal-unseal-vault] on how to unseal a vault node.

The first node that is unsealed in a multi-node Vault cluster will become the active node. The active node holds the leader election lock. The other unsealed nodes become standby.

## Writing secrets to the active node

1. Check the active Vault node:

    ```sh
    kubectl -n default get vault example -o jsonpath='{.status.vaultStatus.active}'
    ```

2. Configure port forwarding between the local machine and the active Vault node:

    ```sh
    kubectl -n default get vault example -o jsonpath='{.status.vaultStatus.active}' | xargs -0 -I {} kubectl -n default port-forward {} 8200
    ```

3. Open a new terminal.

4. Export the following environment for [Vault CLI environment][vault-cli-env].
    The root token used to authenticate the Vault CLI requests is given below. Replace `<root-token>` with the root token generated during [Initialization](#initializing-a-vault-cluster).

    ```sh
    export VAULT_ADDR='https://localhost:8200'
    export VAULT_SKIP_VERIFY="true"
    export VAULT_TOKEN=<root-token>
    ```

    Consult the [Vault Authentication][authentication] docs for more advanced configuration.

5. Write and read an example secret:

    ```sh
    $ vault write secret/foo value=bar

    $ vault read secret/foo

    Key             	Value
    ---             	-----
    refresh_interval	768h0m0s
    value           	bar
    ```

    Successful operations indicate that the active Vault node is serving requests.

## Enable audit backend

[File audit backend][file_audit] is the most native experience for Vault in container.

1. Set up port-forward connection to active node and Vault CLI env same as previous section:

    ```sh
    kubectl -n default get vault example -o jsonpath='{.status.vaultStatus.active}' | xargs -0 -I {} kubectl -n default port-forward {} 8200
    ```

    Open another terminal and type:

    ```sh
    export VAULT_ADDR='https://localhost:8200'
    export VAULT_SKIP_VERIFY="true"
    export VAULT_TOKEN=<root-token>
    ```

2. Enable file audit backend and write audit log to standard output:

    ```sh
    vault audit-enable file file_path=stdout
    ```

   Then use docker/Kubernetes log collector to save logs and view later.

## Accessing Vault on Kubernetes

Vault-operator creates [Kubernetes services][k8s-services] for accessing Vault deployments.

The service always exposes the active Vault node. It hides failures by switching the service pointer to the currently active node when failover occurs.

The name and namespace of the service are the same as the Vault resource. For example, if the Vault resource's name is `example`  and the namespace is `default`, the service's name and namespace will also be `example` and `default` respectively.

Applications in the Kubernetes pod network can access the service through `https://example.default.svc:8200`.

## Starting a standby Vault node

A standby Vault node is initialized and unsealed, but does not hold the leader election lock. The standby node cannot serve user requests. It forwards user requests to the active node. If the active node goes down, a standby node becomes the active node.

1. Unseal the next sealed node.

    See [Unsealing a sealed node](#unsealing-a-sealed-node) for more information.

2. Verify that the node becomes standby:

    ```sh
    $ kubectl -n default get vault example -o jsonpath='{.status.vaultStatus.standby}'
    [example-1003480066-jzmwd]
    ```

    The setup now contains an active and a standby Vault node.

## Automated failover

In an HA Vault setup, when the active node goes down the standby node takes over the active role and starts serving client requests.

To see how it works, terminate the active node, and wait for the standby node to become active.

1. Terminate the active Vault node:

    ```
    kubectl -n default get vault example -o jsonpath='{.status.vaultStatus.active}' | xargs -0 -I {} kubectl -n default delete po {}
    ```

    The standby node becomes active.

2. Verify that the previous standby node is now active:

    ```
    $ kubectl -n default get vault example -o jsonpath='{.status.vaultStatus.active}'
    example-1003480066-jzmwd
    ```

Regular vault operations like reading and writing a secret to the active node should now succeed.

## Failure recovery

Vault-operator recovers any inactive or terminated Vault pods to maintain the size of cluster.

To see how it works, perform the following:

1. Ensure that a Vault node is terminated.

   If a node has not yet been terminated, follow the instructions in the [Automated failover](#automated-failover) section above.

2. Verify that the newly sealed Vault node is created:

    ```
    $ kubectl -n default get vault example -o jsonpath='{.status.vaultStatus.sealed}'
    [example-994933690-h066h]
    ```

A new Vault node is created to replace the terminated one. Unseal the node and continue using HA.

[ha]: https://www.vaultproject.io/docs/concepts/ha.html
[initialize-vault]: https://www.vaultproject.io/intro/getting-started/deploy.html#initializing-the-vault
[seal-unseal-vault]: https://www.vaultproject.io/intro/getting-started/deploy.html#seal-unseal
[authentication]: https://www.vaultproject.io/docs/concepts/auth.html
[vault-cli]: https://www.vaultproject.io/docs/install/index.html
[vault-cli-env]: https://www.vaultproject.io/docs/commands/environment.html
[k8s-services]: https://kubernetes.io/docs/concepts/services-networking/service/
[file_audit]:https://www.vaultproject.io/docs/audit/file.html
[example_vault]:./example_vault.yaml
