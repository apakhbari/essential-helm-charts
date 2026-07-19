### 🔍 Download and Copy Helm Binary

The official Helm project provides binary releases for Linux on their GitHub page . For an air-gapped environment, you'll need to download the binary on a machine with internet access and transfer it to your cluster's control plane node.

1.  **Download the tarball** on the internet-connected machine:
    ```bash
    curl -L -o helm-linux-amd64.tar.gz https://get.helm.sh/helm-v4.2.0-linux-amd64.tar.gz
    ```
    *Check the [official Helm releases page](https://github.com/helm/helm/releases) for the latest version number* .

2.  **Transfer the file** to your air-gapped cluster node (e.g., via USB drive or SCP if allowed).

3.  **Unpack and install** the binary on the cluster node, and move it to a directory in your `PATH` like `/usr/local/bin` :
    ```bash
    tar -zxvf helm-linux-amd64.tar.gz
    sudo mv linux-amd64/helm /usr/local/bin/helm
    ```

4.  **Verify the installation** :
    ```bash
    helm version --short
    ```

### 📝 Key Details

*   **Binary Location**: The `helm` binary is placed in `/usr/local/bin`, making it available system-wide .
*   **Permission**: The `sudo mv` command sets the binary as owned by `root`, which is standard for system binaries.
*   **Version Update**: The command uses Helm v4.2.0 as an example. Always replace the version `v4.2.0` with the latest stable version from the [Helm releases page](https://github.com/helm/helm/releases) .

Once the binary is installed, you can proceed with using Helm to install applications from your local chart archives and your internal container registry.