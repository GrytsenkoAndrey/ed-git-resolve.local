# Git

## Bitbucket

### 2 ssh keys for different repositories

[link](https://support.atlassian.com/bitbucket-cloud/docs/managing-multiple-bitbucket-user-ssh-keys-on-one-device/)


1. Check that SSH is installed and the SSH Agent is started (see the relevant [SSH setup guide for your operating system](https://support.atlassian.com/bitbucket-cloud/docs/configure-ssh-and-two-step-verification/)).

2. Create an SSH key pair for each account, such as:
   
```ssh-keygen -t ed25519 -b 4096 -C "{username@emaildomain.com}" -f {ssh-key-name}```

3. Add each private key to the SSH agent, such as:
   
```ssh-add ~/.ssh/{ssh-key-name}```

4. Add each private key to the SSH configuration, such as:

```
#bitbucket_username1 account
Host bitbucket.org-bitbucket_username1
  HostName bitbucket.org
  User git
  IdentityFile ~/.ssh/{ssh-key-bitbucket_username1}
  IdentitiesOnly yes

#bitbucket_username2 account
Host bitbucket.org-bitbucket_username2
  HostName bitbucket.org
  User git
  IdentityFile ~/.ssh/{ssh-key-bitbucket_username2}
  IdentitiesOnly yes
```

Where bitbucket_username1 and bitbucket_username2 are the Bitbucket usernames of the two accounts the SSH keys were created for. Your Bitbucket username is listed under **Bitbucket profile settings** on your [Bitbucket Personal settings page](https://bitbucket.org/account/settings/).

5. Add the public keys to the corresponding accounts on Bitbucket Cloud (see the relevant [SSH setup guide for your operating system](https://support.atlassian.com/bitbucket-cloud/docs/configure-ssh-and-two-step-verification/)).

6. Clone repositories or update the git remote for repositories already cloned.

- To clone a repository, use the git clone command with the updated host bitbucket.org-{bitbucket_username}, such as:

```git clone git@bitbucket.org-{bitbucket_username}:{workspace}/{repo}.git```

- To update the Git remote for repositories already cloned, run the git remote command from within the repository with the updated host bitbucket.org-{bitbucket_username}, such as:

```git remote set-url origin git@bitbucket.org-{bitbucket_username}:{workspace}/{repo}.git```

7. Update the Git configuration for the repository to the relevant user by navigating into the relevant repository and updating the Git configuration, such as:

```
git config user.name "{bitbucket_username}"
git config user.email "{user@example.com}"
```

Running git config without the --global option will set the user details for an individual repository, rather than user-wide.
