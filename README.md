# Keyra FAQ

Frequently asked questions for the Keyra WPF desktop application.

## Table of Contents

- [How do I delete a vault?](#how-do-i-delete-a-vault)
- [How do I copy a secret from one vault to another?](#how-do-i-copy-a-secret-from-one-vault-to-another)
- [How do I copy the raw secret value?](#how-do-i-copy-the-raw-secret-value)
- [How do I integrate a vault with the Internal Master Vault?](#how-do-i-integrate-a-vault-with-the-internal-master-vault)
- [How do I protect a vault with Active Directory on a domain-joined machine?](#how-do-i-protect-a-vault-with-active-directory-on-a-domain-joined-machine)
- [How do I configure Git so new vaults are automatically created as Git repositories?](#how-do-i-configure-git-so-new-vaults-are-automatically-created-as-git-repositories)

---

## How do I delete a vault?

1. Right-click the vault in the tree view (hold **Shift** to reveal advanced options)
2. Select **Delete Vault**
3. Confirm deletion in the dialog

The deletion process removes:

- The vault directory and all its secrets and folders
- The encryption key from the `keys/` folder
- The certificate from the Windows Certificate Store (Current User > Personal), if one was imported

If the vault has a Git remote configured, a second confirmation dialog asks whether to also delete the remote repository. You must type the vault name to confirm remote deletion.

**Warning:** Vault deletion cannot be undone. Ensure you have exported or backed up the vault before deleting.

---

## How do I copy a secret from one vault to another?

Keyra supports cross-vault secret transfer via clipboard. The secret is re-encrypted with the target vault's key during paste.

### Copy

1. Unlock the source vault
2. Select the secret in the secrets list
3. Press **Ctrl+Shift+C** or right-click and select **Copy Secret**

This copies the encrypted secret and its metadata to the clipboard as a `SecretTransferData` package. The ciphertext remains encrypted with the source vault's key.

### Paste

1. Unlock the target vault
2. Select the target vault or folder in the tree view
3. Press **Ctrl+V** or right-click and select **Paste Secret**

Keyra automatically:

1. Detects the source key from the clipboard data
2. Locates the source key in the repository (the source vault must still exist or the key must be accessible)
3. Decrypts the secret with the source key
4. Re-encrypts the secret with the target vault's key
5. Creates the secret in the target location

If the source key cannot be found (e.g., the source vault was deleted), the paste will fail with a "Key not found" error.

---

## How do I copy the raw secret value?

There are three copy options for secrets:

| Action | Shortcut | Description |
|--------|----------|-------------|
| **Copy Plaintext** | Ctrl+C | Decrypts the secret and copies the plaintext value to the clipboard |
| **Copy Secret** | Ctrl+Shift+C | Copies the full encrypted secret package for cross-vault transfer |
| **Copy Raw Secret** | Context menu (advanced) | Copies the secret transfer data as raw JSON (for developer use) |

To copy just the decrypted password or value, select the secret and press **Ctrl+C**. The plaintext is placed on the clipboard and can be pasted into any application.

---

## How do I integrate a vault with the Internal Master Vault?

Integrating a vault with the Internal Master Vault enables automatic unlocking -- when Keyra starts and unlocks the Master Vault, all integrated vaults are unlocked automatically without any password prompts.

### During vault creation

1. Open **Vault > Create New Vault**
2. Check **Integrate with Master Key**
3. The authenticator chain section is hidden (no password needed)
4. Click **Create Vault**

The vault key is protected with a randomly generated password that is stored encrypted inside the Internal Master Vault. You never see or manage this password.

### After vault creation

If you already have a vault and want to integrate it later:

1. Unlock the vault
2. Hold **Shift** and right-click the vault in the tree view
3. Select **Integrate Vault**
4. Confirm in the dialog

Keyra generates a new random password, re-protects the vault key with it, and stores the password in the Internal Master Vault. From that point on, the vault unlocks automatically at startup.

### Default setting

To make all new vaults integrate with the Master Key by default:

1. Go to **Tools > Options**
2. Enable **Create vaults as Master Key integrated**

This setting can also be enforced via Group Policy (`CreateVaultsAsMasterKeyIntegrated` DWORD under `HKLM\Software\Policies\PenguinConverters\Keyra`).

---

## How do I protect a vault with Active Directory on a domain-joined machine?

On Active Directory domain-joined machines, Keyra can protect vault keys using DPAPI-NG with AD security principals (users and groups). This binds the key to specific AD identities so that only authorized domain accounts can decrypt it.

### Exporting with AD protection

1. Right-click the vault and select **Export Vault**
2. Check **Enable AD Protection** (available only on domain-joined machines)
3. Your current user is added automatically
4. Click **Add** to open the Windows Object Picker and select additional AD users or groups
5. Choose the logic mode:
   - **OR** (default): Any one of the selected principals can decrypt the key
   - **AND**: All selected principals must authenticate together
6. Complete the export

### Best practice: always export with AD protection

When sharing vaults on a domain-joined machine, **always export with AD protection enabled**. This eliminates the need for an initial password exchange between parties:

- Without AD protection, you must securely communicate the vault password to every recipient -- a significant security risk if no shared vault is already established
- With AD protection, you select the target users or groups during export. Recipients import the vault and can decrypt it using their AD credentials alone -- no password exchange required
- The DPAPI-NG protection descriptor ensures that only the specified AD principals can access the key, enforced by Windows at the OS level

This is especially important when onboarding new team members who do not yet have access to a shared vault for secure password exchange.

### Creating vaults with AD protection

When creating a new vault with the DPAPI-NG provider, you can add AD principals (SIDs) directly during creation:

1. Select **DPAPI-NG** as the key storage provider
2. Configure the authenticator chain (password, YubiKey, etc.)
3. Add AD users or groups to the protection descriptor
4. Choose OR or AND logic for multi-principal access

The vault key is then protected with both the authenticator chain and the AD descriptor.

---

## How do I configure Git so new vaults are automatically created as Git repositories?

Keyra supports per-vault Git integration for syncing encrypted secrets across machines. Global Git settings define defaults that are applied to every new vault.

### Configure global Git settings

1. Go to **Tools > Options**
2. In the Git configuration section, configure:

| Setting | Description |
|---------|-------------|
| **Enabled** | Master switch for Git integration |
| **Remote Name** | Default remote name (default: `origin`) |
| **Default Branch** | Branch name for new repos (default: `main`) |
| **Auto-commit on save** | Automatically commit when secrets are saved |
| **Auto-push on save** | Automatically push after each commit |
| **Auto-pull on load** | Pull latest changes when a vault is opened |
| **Server Base URL** | Git server URL for automatic repository creation |
| **Author Name** | Git commit author name |
| **Author Email** | Git commit author email |

3. Click **OK** to save

The settings are stored in `{repository}/git.json`.

### Git Personal Access Token (PAT)

The Git PAT is stored in the Internal Master Vault (encrypted) rather than in the `git.json` file. If you previously stored the PAT in `git.json`, Keyra automatically migrates it to the Internal Master Vault on startup.

### Creating vaults with Git

When creating a new vault with Git enabled, you have three options:

1. **No Repository** -- Vault is created without Git (you can add Git later)
2. **Use Custom Repository** -- Clone an existing Git repository URL into the vault
3. **Use Global Server** -- Automatically create a new repository on the configured Git server using the Server Base URL + vault name

When global Git settings are enabled, the auto-commit, auto-push, and auto-pull settings are propagated to the new vault automatically.

### Per-vault overrides

Each vault can override the global Git settings. Right-click a vault and select **Git Settings** to configure vault-specific Git behavior. By default, vaults inherit from the global configuration.

---

## License

Proprietary -- Penguin Converters AG
