# Backups

I use this script to do nightly backups to a backup server on my home network using [Borg Backup](https://borgbackup.org) over SSH.  Borg does incremental backups of a system and stores them in an encrypted git-like repository.

If the host has a MariaDB database server running on it, the script will attempt to export of all of the server's databases prior to running the backup, saving them to `/var/mariadb_backups`.  Each database (schema) will be exported to a separate file.  The three most recent backup files will be retained, and older ones will be automatically removed.  These files will then be included in the system backup.

## What you'll need

- A backup server, accessible via SSH
- An SSH private key that Borg can use to connect without user intervention
- The equivalent public key installed on the backup server
- Borg must be installed on both machines
- A backup key (i.e. password) stored in a safe place.  I generate my backup keys with Bitwarden.

In my setup, I use a user called 'borg' on my backup server to receive the backup.  The backups run as root on the machine being backed up to avoid any permissions issues, though Borg also works great individual user backups without root permissions.

## On the machine being backed up

1. Copy the run-backup script from this repo to /root/bin (or another appropriate location) and set it to be executable.
2. Create a new SSH private key for the root user (don't set a passphrase)
3. Create a `.borg` file in root's home directory with the backup server address and Borg key you'll use to encrypt your backups
4. Set up a rule in root's `crontab` to run the backups on a schedule

```
sudo su - root
wget https://raw.githubusercontent.com/jpitoniak/shell-scripts/refs/heads/main/backups/run-backup -O /root/bin/run-backup
chmod 700 /root/bin/run-backup
ssk-keygen -t ed25519 -C 'root@MACHINE_NAME'
vi ~/.borg
crontab -e
```

Contents of the `.borg` file:

```
BACKUP_SERVER='123.45.67.89'
BORG_PASSPHRASE='PASTE_BACKUP_KEY_HERE'
```

- BACKUP_SERVER can be an IP address or hostname.  It must be reachable via SSH.
- BORG_PASSPHRASE should contain the key you'll use to encrypt your backups.  Make it long and random and keep it in a safe place like a password manager.

Contents of the crontab file:

```
MAILTO=you@yourdomain.com
15 0 * * * /root/bin/run-backup
```

- The MAILTO line is optional.  If present and your machine has a working `sendmail` program, you'll get a summary report mailed to you after the backup completes.
- The second line schedules the backups to begin daily at quarter past midnight.  I stagger the starting times across my different machines so as to not overwhelm my backup server with a bunch of connections all at once.


## On the backup server (machine receiving the backup)

1. As the 'borg' user, change to the directory where you'll store the backups
2. Create a new backup repository.
3. You'll be prompted to enter the backup key twice and then confirm that you've entered it correctly.
4. Add a new entry to the borg user's `authrotized_hosts` to allow the new machine to connect and access the `borg server` process.

```
sudo su - borg
cd /home/borg/backups
borg init --encryption=repokey ./MACHINE_NAME
vi ~/.ssh/authorized_keys
```

For the `authorized_keys` file, first open the /root/.ssh/id_ed25519.pub file on the machine being backed up and copy the contents.  Then enter the following, adding the contents to the end of the line:

```
command="borg serve --restrict-to-path /external/borg-backup/MACHINE\_NAME",restrict ID_ED25519_PUB_CONTENTS
```

## Create an initial backup (optional)

To start a backup, simply run `/root/bin/run-backup` from the command line.  If everything is set up correctly, the backup should start.

Depending on the size of your system, the first backup could take a long time.  If running it over SSH, recommend using a tool like `screen` to keep things running in the event that your SSH session gets disconnected before the backup finishes.