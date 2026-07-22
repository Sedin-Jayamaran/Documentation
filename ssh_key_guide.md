# SSH Key Management Guide for Prod and Non-Prod Environments

This guide explains how to generate, configure, and install separate SSH keys for your **Production (Prod)** and **Non-Production (Non-Prod / Dev / Staging)** environments.

Using separate keys increases security: if a non-prod key is compromised, your production servers remain secure.

---

## Step 1: Generate Separate SSH Keys

We will use the modern and secure **Ed25519** algorithm to generate the keys. If your server is very old and doesn't support Ed25519, you can use RSA (`-t rsa -b 4096`) instead.

Open your local terminal and run the following commands:

### A. Create the Non-Production Key
```bash
ssh-keygen -t ed25519 -C "your_email@example.com-nonprod" -f ~/.ssh/id_ed25519_nonprod
```
*   **`-t ed25519`**: Specifies the Ed25519 key type.
*   **`-C`**: Adds a descriptive comment to help identify the key.
*   **`-f`**: Specifies the filename and path. Saving it as `id_ed25519_nonprod` prevents overwriting your default key.

### B. Create the Production Key
```bash
ssh-keygen -t ed25519 -C "your_email@example.com-prod" -f ~/.ssh/id_ed25519_prod
```

> [!IMPORTANT]
> **Use Passphrases**: When prompted, enter a strong passphrase for both keys (especially the Production key). This encrypts the private key on your disk, protecting it if your laptop is lost or stolen.

---

## Step 2: Add the Keys to Your Remote Servers

You need to copy the **public key** (the `.pub` file) to the server. Never share or copy your private key.

### Option A: Using `ssh-copy-id` (Recommended & Easiest)
If you already have password-based SSH access to the server, use `ssh-copy-id`. It automatically sets the correct permissions on the server.

*   **For Non-Prod Server:**
    ```bash
    ssh-copy-id -i ~/.ssh/id_ed25519_nonprod.pub user@non-prod-server-ip
    ```
*   **For Prod Server:**
    ```bash
    ssh-copy-id -i ~/.ssh/id_ed25519_prod.pub user@prod-server-ip
    ```

### Option B: Manual Installation (If `ssh-copy-id` is not available)
1.  **View your public key content** on your local machine:
    *   *Non-Prod:* `cat ~/.ssh/id_ed25519_nonprod.pub`
    *   *Prod:* `cat ~/.ssh/id_ed25519_prod.pub`
2.  **Log into your remote server** (using password or existing access).
3.  **Create the `.ssh` directory and set permissions** (if it doesn't exist):
    ```bash
    mkdir -p ~/.ssh
    chmod 700 ~/.ssh
    ```
4.  **Append your public key string** to the `authorized_keys` file:
    ```bash
    echo "YOUR_PUBLIC_KEY_STRING_HERE" >> ~/.ssh/authorized_keys
    ```
5.  **Set permissions** on the `authorized_keys` file:
    ```bash
    chmod 600 ~/.ssh/authorized_keys
    ```

---

## Step 3: Configure Local SSH Config for Ease of Use

Instead of specifying the key manually every time you connect (e.g., `ssh -i ~/.ssh/id_ed25519_prod user@ip`), you can configure your local `~/.ssh/config` file to handle this automatically.

1. Open or create your local SSH config file:
   ```bash
   nano ~/.ssh/config
   ```
2. Add configurations for your servers:

```text
# --- Non-Production Servers ---
Host dev-server
    HostName dev.example.com          # Or IP address
    User ubuntu                       # Remote username
    IdentityFile ~/.ssh/id_ed25519_nonprod
    IdentitiesOnly yes

Host staging-server
    HostName staging.example.com      # Or IP address
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519_nonprod
    IdentitiesOnly yes

# --- Production Servers ---
Host prod-server-1
    HostName prod1.example.com        # Or IP address
    User admin
    IdentityFile ~/.ssh/id_ed25519_prod
    IdentitiesOnly yes

Host prod-server-2
    HostName prod2.example.com
    User admin
    IdentityFile ~/.ssh/id_ed25519_prod
    IdentitiesOnly yes
```

*   **`IdentityFile`**: Specifies exactly which private key to use.
*   **`IdentitiesOnly yes`**: Prevents SSH from trying other keys loaded in your SSH agent, which can trigger security blocklists on servers.

3.  **Save the file** (In nano: `Ctrl+O`, `Enter`, `Ctrl+X`).

Now, you can connect simply by typing:
```bash
ssh dev-server
ssh prod-server-1
```
SSH will automatically use the correct key and username!

---

## Step 4: Verify Local Permissions (Security Check)

Your SSH client will refuse to use private keys if they are readable by other users on your local computer. Ensure correct local permissions by running:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519_nonprod
chmod 600 ~/.ssh/id_ed25519_prod
```
