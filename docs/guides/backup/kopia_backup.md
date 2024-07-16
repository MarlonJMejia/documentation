---
title: Backup of Directories - Kopia
author: Marlon Mejia
contributors: Mejia
tested_with: 9.0
tags:
  - backup
  - rsnapshot
---

# Snapshots of Directories - _kopia_

## Prerequisites

- Know how to install additional repositories
- Know about mounting filesystems external to your machine (external drive, remote filesystem, and so on.)
- Minor Knowledge of scripting
- Know how to change the crontab for the root user
- Knowledge of client and server interaction as you may need to create a server repository for Kopia.

## Introduction

*Kopia* is an utility to create archival backups of your directories based on Repositories that can have an unlimited amount of Policies. Repositories allow you to encrypt and define files you may want to back up.

*Kopia* is very flexible as it can backup to multiple storage types making it easy to create an off-site backup solution to a cloud solution. [*List of Storage Locations*](https://kopia.io/docs/repositories/)

*Kopia* creates snapshots of the files and directories you designate, then encrypts these snapshots before they leave your computer, and finally uploads these encrypted snapshots to cloud/network/local storage called a repository. Snapshots are maintained as a set of historical point-in-time records based on policies that you define.

## Installing Kopia on Rocky Linux 9

Import the GPG Key from Kopia

```bash
rpm --import https://kopia.io/signing-key
```

Create the Repository file for `dnf` or `yum`:

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kopia.repo
[Kopia]
name=Kopia
baseurl=http://packages.kopia.io/rpm/stable/\$basearch/
gpgcheck=1
enabled=1
gpgkey=https://kopia.io/signing-key
EOF
```

You will now be able to install *Kopia*

```bash
dnf install kopia
```

???+ note "Kopia UI"

    Kopia has a UI Client that you can utilize, you are free to download it if you feel more comfortable with UI Clients, but the author will not be using it on this guide.
    `install kopia-ui`


### Enabling auto-completion for Kopia

Using `kopia --completion-script-bash`

```bash
echo 'eval "$(kopia --completion-script-bash)"'\
| tee /etc/bash_completion.d/kopia-completion.sh
```

Source the file to add to your current session

```bash
source /etc/bash_completion.d/kopia-completion.sh
```

Autocompleted command options will be shown when you double ++tab+tab++ after typing Kopia

```bash
[root@rockytest rocky]# kopia
benchmark   diff        help        list        mount       policy      repository  restore     server      show        snapshot
```

## Creating a local repository

Create a folder to use with the repository

```bash
mkdir /.kopia_backups
```

Create the repository for your local filesystem

```bash
kopia repository create filesystem --path /.kopia_backups
```

When first creating a repository you will need to provide a password, you will receive the policy retention defined for your repository, these are defaults and derived from the global policy.

??? note

    Kopia cannot use multiple passwords, if you are currently using this guide for production make certain this password is stored on another entity other than yourself.


```bash
[root@rockytest .kopia_backups]# kopia repository create filesystem --path /.kopia_backups
Enter password to create new repository:
Re-enter password for verification:
Initializing repository with:
  block hash:          BLAKE2B-256-128
  encryption:          AES256-GCM-HMAC-SHA256
  splitter:            DYNAMIC-4M-BUZHASH
Connected to repository.

NOTICE: Kopia will check for updates on GitHub every 7 days, starting 24 hours after first use.
To disable this behavior, set environment variable KOPIA_CHECK_FOR_UPDATES=false
Alternatively you can remove the file "/root/.config/kopia/repository.config.update-info.json".

Retention:
  Annual snapshots:                 3   (defined for this target)
  Monthly snapshots:               24   (defined for this target)
  Weekly snapshots:                 4   (defined for this target)
  Daily snapshots:                  7   (defined for this target)
  Hourly snapshots:                48   (defined for this target)
  Latest snapshots:                10   (defined for this target)
  Ignore identical snapshots:   false   (defined for this target)
Compression disabled.

To find more information about default policy run 'kopia policy get'.
To change the policy use 'kopia policy set' command.

NOTE: Kopia will perform quick maintenance of the repository automatically every 1h0m0s
and full maintenance every 24h0m0s when running as root@rockytest.

See https://kopia.io/docs/advanced/maintenance/ for more information.

NOTE: To validate that your provider is compatible with Kopia, please run:

$ kopia repository validate-provider
```

## Your initinal snapshot and incremental snapshots

Creating a snapshot is as simple as a `kopia snapshot create`:

```bash title="Verify you are using the correct repository"
kopia repository status
```

```bash title="Create a snapshot for the current users home folder"
kopia snapshot create $HOME
```

Below is the output of the created snapshot:

>_**{user}**@**{hostname}**:**/{directory}**_<br>
>**{CREATIONDATE}** - **{{SNAPSHOT ID}}** - **{{SIZE}}** - **{{PERMISSIONS}}**

```bash title="output"
[root@rockytest ~]# kopia snapshot ls
root@rockytest:/root
2024-07-15 18:57:22 EDT ke68996eddc0f04fd44419874c1624925 28.5 MB dr-xr-x--- files:78 dirs:106 (latest-1,hourly-1,daily-1,weekly-1,monthly-1,annual-1)
```

```bash title="Create an incremental backup of the backup."
mkdir -p $HOME/folder/{1..100}
kopia snapshot create $HOME
```

```bash title="output"
[root@rockytest ~]# kopia snapshot ls
root@rockytest:/root
2024-07-15 18:57:22 EDT ke68996eddc0f04fd44419874c1624925 28.5 MB dr-xr-x--- files:78 dirs:106 (latest-2,hourly-2)
2024-07-15 19:11:57 EDT k2e8cc246eeeef5d3f051e67fa513c7b0 28.7 MB dr-xr-x--- files:110 dirs:208 (latest-1,hourly-1,daily-1,weekly-1,monthly-1,annual-1)
```

Viewing the contents of the snapshot with `kopia list`:

```bash
kopia list  k2e8cc246eeeef5d3f051e67fa513c7b0 -r -l
```

```bash title=output
drwxr-xr-x            0 2024-07-15 19:11:57 EDT kf652ba8cf935331461506477ecd8bf3a  folder/2/
```

List a specific folder content and files by the object ID

```bash
kopia list kf652ba8cf935331461506477ecd8bf3a -l
```

```bash title="Json Format"
kopia content show  kf652ba8cf935331461506477ecd8bf3a -j
```

## Configuring your repository policy

Let's look at out current policy

```bash
kopia policy list
```

```bash

```

## Configuring the global policy

In this case, _rsnapshot_ will run locally to back up a particular machine. In this example, we will break down the configuration file, and show you exactly what you need to change.

You will need to use `vi` (or edit with your favorite editor) to open the _/etc/rsnapshot.conf_ file.

The first thing to change is the _snapshot_root_ setting. The default has this value:

`snapshot_root   /.snapshots/`

You need to change this to your mount point that we created above plus the addition of "storage".

`snapshot_root   /mnt/backup/storage/`

You also want to tell the backup not to run if the drive is not mounted. To do this, remove the "#" sign (also called a remark, number sign, hash symbol, and so on.) next to `no_create_root` which looks this way:

`no_create_root 1`

Next go down to the section titled `# EXTERNAL PROGRAM DEPENDENCIES #` and remove the comment (again, the "#" sign) from this line:

`#cmd_cp         /usr/bin/cp`

It now reads:

`cmd_cp         /usr/bin/cp`

While you do not need `cmd_ssh` for this particular configuration, you will need it for our other option and it does not hurt to have it enabled. Find the line that says:

`#cmd_ssh        /usr/bin/ssh`

Remove the "#" sign:

`cmd_ssh        /usr/bin/ssh`

Next you need to skip down to the section titled `#     BACKUP LEVELS / INTERVALS         #`

Earlier versions of _rsnapshot_ had `hourly, daily, monthly, yearly` but are now `alpha, beta, gamma, delta`. It is a bit confusing. What you need to do is add a remark to any interval that you will not use. In the configuration, delta is already remarked out.

In this example, you are not going to be running any other increments other than a nightly backup. Just add a remark to alpha and gamma. When completed, your configuration file will be:

```text
#retain  alpha   6
retain  beta    7
#retain  gamma   4
#retain delta   3
```

Skip down to the `logfile` line, which by default is:

`#logfile        /var/log/rsnapshot`

Remove the remark:

`logfile        /var/log/rsnapshot`

Finally, skip down to the `### BACKUP POINTS / SCRIPTS ###` section and add any directories that you want to add in the `# LOCALHOST` section, remember to use ++tab++ rather than ++space++ between elements!

For now write your changes (`SHIFT :wq!` for `vi`) and exit the configuration file.

### Checking the configuration

You want to ensure that you did not add spaces or any other glaring errors to our configuration file while you were editing it. To do this, you run _rsnapshot_ against our configuration with the `configtest` option:

`rsnapshot configtest` will show `Syntax OK` if no errors exist.

You should get into the habit of running `configtest` against a particular configuration. The reason for that will be more evident when you get into the **Multiple Machine or Multiple Server Backups** section.

To run `configtest` against a particular configuration file, run it with the -c option to specify the configuration:

`rsnapshot -c /etc/rsnapshot.conf configtest`

## Running the backup the first time

With `configtest` verifying everything is OK, it is now time to run the backup for the first time. You can run this in test mode first if you like, so that you can see what the backup script is going to do.

Again, to do this you do not necessarily have to specify the configuration in this case, but it is a good idea to get into the habit of doing so:

`rsnapshot -c /etc/rsnapshot.conf -t beta`

Which will return something similar to this, showing you what will happen when the backup is actually run:

```bash
echo 1441 > /var/run/rsnapshot.pid
mkdir -m 0755 -p /mnt/backup/storage/beta.0/
/usr/bin/rsync -a --delete --numeric-ids --relative --delete-excluded \
    /home/ /mnt/backup/storage/beta.0/localhost/
mkdir -m 0755 -p /mnt/backup/storage/beta.0/
/usr/bin/rsync -a --delete --numeric-ids --relative --delete-excluded /etc/ \
    /mnt/backup/storage/beta.0/localhost/
mkdir -m 0755 -p /mnt/backup/storage/beta.0/
/usr/bin/rsync -a --delete --numeric-ids --relative --delete-excluded \
    /usr/local/ /mnt/backup/storage/beta.0/localhost/
touch /mnt/backup/storage/beta.0/
```

When the test meets your expectations, run it manually the first time without the test:

`rsnapshot -c /etc/rsnapshot.conf beta`

When the backup finishes, browse to /mnt/backup and examine the directory structure that it creates there. There will be a `storage/beta.0/localhost` directory, followed by the directories that you specified to backup.

### Further explanation

Each time the backup runs, it will create another beta increment, 0-6, or 7 days worth of backups. The newest backup will always be beta.0 whereas yesterday's backup will always be beta.1.

The size of each of these backups will appear to take up the same amount (or more) of disk space, but this is because of _rsnapshot's_ use of hard links. To restore files from yesterday's backup, you just copy them back from beta.1's directory structure.

Each backup is only an incremental backup from the previous run, BUT, because of the use of hard links, each backup directory, contains either the file or the hard-link to the file in whichever directory it was actually backed up in.

To restore files, you do not have to decide the directory or increment to restore them from, just what time stamp of the backup that you are restoring. It is a great system and uses far less disk space than many other backup solutions.

## Setting the backup to run automatically

With testing completed and secure in the knowledge that things will work without issue, the next step is to set up the crontab for the root user to automate the process every day:

`sudo crontab -e`

If you have not run this before, choose vim.basic as your editor or your own editor preference when the `Select an editor` line comes up.

You are going to set your backup to automatically run at 11 PM, so you will add this to the crontab:

```bash
## Running the backup at 11 PM
00 23 *  *  *  /usr/bin/rsnapshot -c /etc/rsnapshot.conf beta`
```

## Multiple machine or multiple server backups

Doing backups of multiple machines from a machine with a RAID array or large storage capacity, on-premise or from on an Internet connection elsewhere works well.

If running these backups over the Internet, you need to ensure that each location has adequate bandwidth for the backups to occur. You can use _rsnapshot_ to synchronize an on-site server with an off-site backup array or backup server to improve data redundancy.

## Assumptions

Running _rsnapshot_ from a machine remotely, on-premise. Running this exact configuration is possible remotely off-premise also.

In this case, you will want to install _rsnapshot_ on the machine that is doing all of the backups. Other assumptions are:

- That the servers you will be backing up to, have a firewall rule that allows the remote machine to SSH into it
- That each server that you will be backing up has a recent version of `rsync` installed. For Rocky Linux servers, run `dnf install rsync` to update your system's version of `rsync`.
- That you have connected to the machine as the root user, or that you have run `sudo -s` to switch to the root user

## SSH public or private keys

For the server that will be running the backups, you need to generate an SSH key-pair for use during the backups. For our example, you will be creating RSA keys.

If you already have a set of keys generated, you can skip this step. You can find out by doing an `ls -al .ssh` and looking for an `id_rsa` and `id_rsa.pub` key pair. If none exists, use the following link to set up keys for your machine and the server(s) that you want to access:

[SSH Public Private Key Pairs](../security/ssh_public_private_keys.md)

## _rsnapshot_ configuration

The configuration file needs to be nearly the same as the one we created for the **Basic Machine or Single Server Backup** , except that you need to change some of the options.

The snapshot root is the default:

`snapshot_root   /.snapshots/`

Comment this line out:

`no_create_root 1`

`#no_create_root 1`

The other difference here is that each machine will have its own configuration. When you get used to this, you will just copy one of your existing configuration files over to a different name and change it to fit any additional machines that you want to backup.

For now, you want to change the configuration file just (as shown above), and save it. Copy that file as a template for our first server:

`cp /etc/rsnapshot.conf /etc/rsnapshot_web.conf`

You want to change the configuration file and create the log and lockfile with the machine's name:

`logfile /var/log/rsnapshot_web.log`

`lockfile        /var/run/rsnapshot_web.pid`

Next, you want to change rsnapshot_web.conf so that it includes the directories you want to back up. The only thing that is different here is the target.

Here is an example of the web.ourdomain.com configuration:

```bash
### BACKUP POINTS / SCRIPTS ###
backup  root@web.ourourdomain.com:/etc/     web.ourourdomain.com/
backup  root@web.ourourdomain.com:/var/www/     web.ourourdomain.com/
backup  root@web.ourdomain.com:/usr/local/     web.ourdomain.com/
backup  root@web.ourdomain.com:/home/     web.ourdomain.com/
backup  root@web.ourdomain.com:/root/     web.ourdomain.com/
```

### Checking the configuration and running the initial backup

You can now check the configuration to ensure it is syntactically correct:

`rsnapshot -c /etc/rsnapshot_web.conf configtest`

You are looking for the `Syntax OK` message. If all is well, you can run the backup manually:

`/usr/bin/rsnapshot -c /etc/rsnapshot_web.conf beta`

Assuming that everything works, you can create the configuration files for the mail server (rsnapshot_mail.conf) and portal server (rsnapshot_portal.conf), test them, and do a trial backup.

## Automating the backup

Automating backups for the multiple machine or server version is slightly different. You want to create a bash script to call the backups in order. When one finishes the next will start. This script will look similar to:

`vi /usr/local/sbin/backup_all`

With the content:

```bash
#!/bin/bash/
# script to run rsnapshot backups in succession
/usr/bin/rsnapshot -c /etc/rsnapshot_web.conf beta
/usr/bin/rsnapshot -c /etc/rsnapshot_mail.conf beta
/usr/bin/rsnapshot -c /etc/rsnapshot_portal.conf beta
```

Save the script to /usr/local/sbin and make the script executable:

`chmod +x /usr/local/sbin/backup_all`

Create the crontab for root to run the backup script:

`crontab -e`

Add this line:

```bash
## Running the backup at 11 PM
00 23 *  *  *  /usr/local/sbin/backup_all
```

## Reporting the backup status

To ensure that everything is backing up according to plan, you might want to send the backup log files to your email. If you are running multiple machine backups using _rsnapshot_, each log file will have its own name, which you can send to your email for review by [Using the postfix For Server Process Reporting](../email/postfix_reporting.md) procedure.

## Restoring a backup

Restoring a few files or an entire backup involves copying the files you want from the directory with the date that you want to restore from back to your machine.

## Conclusions and other resources

Getting the setup right with _rsnapshot_ is a little daunting at first, but can save you loads of time backing up your machines or servers.

_rsnapshot_ is powerful, fast, and economical on disk space usage. You can find more on _rsnapshot_, by visiting [rsnapshot.org](https://rsnapshot.org/download.html).
