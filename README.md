# Git

**TOC**
- [Bitbucket](#bitbucket)
- [SSH](#ssh)
- [Worktree](#worktree)


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


## SSH

Note that there are at least two bug reports for ssh-add -d/-D not removing keys:

- ["Debian Bug report #472477: ssh-add -D does not remove SSH key from gnome-keyring-daemon memory"](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=472477)
- ["Ubuntu: ssh-add -D deleting all identities does not work. Also, why are all identities auto-added?"](https://bugs.launchpad.net/ubuntu/+source/openssh/+bug/505278)

The exact issue is:

> ssh-add -d/-D deletes only manually added keys from gnome-keyring.
> There is no way to delete automatically added keys.
> This is the original bug, and it's still definitely present.

> So, for example, if you have two different automatically-loaded ssh identities associated with two different GitHub accounts -- say for work and for home -- there's no way to switch between them. GitHubtakes the first one which matches, so you always appear as your 'home' user to GitHub, with no way to upload things to work projects.

> Allowing ssh-add -d to apply to automatically-loaded keys (and ssh-add -t X to change the lifetime of automatically-loaded keys), would restore the behavior most users expect.

More precisely, about the issue:

> The culprit is gpg-keyring-daemon:

> > It subverts the normal operation of ssh-agent, mostly just so that it can pop up a pretty box into which you can type the passphrase for an encrypted ssh key.
> > And it paws through your .ssh directory, and automatically adds any keys it finds to your agent.
> > And it won't let you delete those keys.

> How do we hate this? Let's not count the ways -- life's too short.

> > The failure is compounded because newer ssh clients automatically try all the keys in your ssh-agent when connecting to a host.
> > If there are too many, the server will reject the connection.
> > And since gnome-keyring-daemon has decided for itself how many keys you want your ssh-agent to have, and has autoloaded them, AND WON'T LET YOU DELETE THEM, you're toast.

This bug is still confirmed in Ubuntu 14.04.4, as recently as two days ago (August 21st, 2014)

A possible workaround:

> Do ssh-add -D to delete all your manually added keys. This also locks the automatically added keys, but is not much use since gnome-keyring will ask you to unlock them anyways when you try doing a git push.
> Navigate to your ~/.ssh folder and move all your key files except the one you want to identify with into a separate folder called backup. If necessary you can also open seahorse and delete the keys from there.
> Now you should be able to do git push without a problem.

## Worktree

### Variant with Docker - Worktree внутри проекта (проще всего)

На хосте:
```
git worktree add ./worktrees/feat-payments feat-payments
git worktree add ./worktrees/fix-auth fix-auth
```

Структура:
```
/var/www/                        ← main branch
/var/www/worktrees/
    feat-payments/               ← отдельная ветка, свои файлы
    fix-auth/                    ← отдельная ветка, свои файлы
```

Всё это уже внутри /var/www → уже смонтировано → видно в обоих контейнерах. 

В .gitignore добавить: ```/worktrees/```

### Запуск нескольких сессий:

```
# Терминал 1 — основная ветка
docker exec -it sswb-claude bash -c "cd /var/www && claude"

# Терминал 2 — feat-payments
docker exec -it sswb-claude bash -c "cd /var/www/worktrees/feat-payments && claude"

# Терминал 3 — fix-auth
docker exec -it sswb-claude bash -c "cd /var/www/worktrees/fix-auth && claude"
```

Каждый claude процесс видит только свою директорию → нет конфликтов по файлам.

### Важный нюанс — .env и база

В твоём случае все три Postgres (main, test, arch) доступны с хоста по портам. Когда запускаешь claude в worktree — он наследует .env из своей директории (worktree). Нужно решить один из вариантов:
```
# Вариант 1: симлинк на общий .env
cd worktrees/feat-payments && ln -s ../../.env .env

# Вариант 2: отдельный .env с другой БД (для полной изоляции)
cp ../../.env worktrees/feat-payments/.env
# поменять DB_DATABASE=ssw_feat в этом .env
```

Для **большинства задач — симлинк**, одна база. Для параллельных миграций — отдельная база обязательно.

### Remove 

```
# 1. Удалить worktree (директорию + регистрацию в git)
git worktree remove ./worktrees/feature-ssp-551-2

# 2. Удалить ветку (если она больше не нужна)
git branch -d feature/ssp-551-2

# Если в worktree есть несохранённые изменения — git откажет. Форс:
git worktree remove --force ./worktrees/feature-ssp-551-2

# Проверить все активные worktree:
git worktree list
```




