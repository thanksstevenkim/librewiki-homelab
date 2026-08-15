# 04. MariaDB

This document describes how MariaDB was installed and prepared for MediaWiki in the LibreWiki Homelab.

The goal of this stage was to add a relational database backend and create a dedicated database and database user for MediaWiki.

The initial architecture became:

```text
Browser
   │
   ▼
Nginx
   │
   ▼
PHP-FPM
   │
   ▼
MediaWiki
   │
   ▼
MariaDB
```

At this stage, all services still run on the same Ubuntu Server VM.

## Environment

Guest system:

- Ubuntu Server 26.04 LTS
- ARM64
- UTM virtual machine

Application VM:

```text
librewiki-ubuntu
```

Database software:

- MariaDB Server
- MariaDB Client

MariaDB was selected because it is a natural fit for MediaWiki and also provides a useful platform for practicing database administration.

## Why MariaDB?

MediaWiki supports multiple database backends, but MariaDB is a particularly appropriate choice for this project.

Reasons include:

- strong compatibility with MediaWiki
- widespread use in MediaWiki deployments
- straightforward administration
- good integration with Ubuntu
- useful experience for Linux infrastructure work
- easy migration later to a dedicated database VM

This homelab will eventually separate the application and database roles.

The planned architecture is:

```text
Ubuntu Server
├── Nginx
├── PHP-FPM
└── MediaWiki
        │
        │ TCP 3306
        ▼
Rocky Linux
└── MariaDB
```

MariaDB is therefore both an application dependency and a future infrastructure exercise.

## Installing MariaDB

The package index was refreshed:

```bash
sudo apt update
```

MariaDB Server and Client were installed:

```bash
sudo apt install mariadb-server mariadb-client
```

The server package installs and enables the MariaDB systemd service.

## Checking the Service

The service status can be checked with:

```bash
systemctl status mariadb
```

A healthy service should report:

```text
active (running)
```

A shorter check is:

```bash
systemctl is-active mariadb
```

Expected result:

```text
active
```

The installed version can be checked with:

```bash
mariadb --version
```

## Connecting as the Database Administrator

The initial administrative connection was performed with:

```bash
sudo mariadb
```

This opens a MariaDB shell similar to:

```text
MariaDB [(none)]>
```

The available databases can be inspected with:

```sql
SHOW DATABASES;
```

Typical system databases include:

```text
information_schema
mysql
performance_schema
sys
```

The MariaDB shell can be exited with:

```sql
EXIT;
```

## Unix Socket Authentication

The MariaDB `root` account was configured to use Unix socket authentication.

This means database administration is performed through the Linux root privilege context:

```bash
sudo mariadb
```

rather than by exposing a password-authenticated MariaDB root account.

The model is:

```text
Linux user
   │
   │ sudo
   ▼
Linux root
   │
   │ unix_socket
   ▼
MariaDB root
```

This avoids the need to use the MariaDB root account remotely.

MediaWiki itself does not use the root database account.

## Running `mariadb-secure-installation`

The initial database security helper was executed:

```bash
sudo mariadb-secure-installation
```

The following choices were made.

### Enter current password for root

No password had been configured for the database root account, so the prompt was left blank by pressing Enter.

### Switch to unix_socket authentication

Selected:

```text
Y
```

This keeps administrative database access tied to Linux root privileges.

### Change the root password

Selected:

```text
n
```

A separate MariaDB root password is unnecessary when Unix socket authentication is used.

### Remove anonymous users

Selected:

```text
Y
```

Anonymous database accounts are not needed for this environment.

### Disallow root login remotely

Selected:

```text
Y
```

Remote applications should never authenticate to MariaDB using the root account.

### Remove test database and access to it

Selected:

```text
Y
```

The default test database is unnecessary for the MediaWiki environment.

### Reload privilege tables now

Selected:

```text
Y
```

This applies the privilege changes immediately.

## Security Model

After the initial hardening, the intended access model is:

```text
Administrator
     │
     │ sudo mariadb
     ▼
MariaDB root
```

and:

```text
MediaWiki
    │
    │ dedicated username/password
    ▼
librewiki database
```

The application and administrator accounts are deliberately separate.

## Creating the MediaWiki Database

The MariaDB shell was opened:

```bash
sudo mariadb
```

A dedicated database was created:

```sql
CREATE DATABASE librewiki
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;
```

The database name is:

```text
librewiki
```

`utf8mb4` was selected so the database can store the full Unicode character range.

This is important for a wiki environment that may contain:

- Korean
- Japanese
- Chinese
- emoji
- uncommon Unicode symbols
- multilingual page titles and content

## Creating the MediaWiki Database User

A dedicated application account was created:

```sql
CREATE USER 'librewiki'@'localhost'
IDENTIFIED BY 'CHANGE_ME';
```

The actual password is stored only in the server's MediaWiki configuration and is not committed to Git.

The database account name is:

```text
librewiki
```

The host is initially:

```text
localhost
```

because MediaWiki and MariaDB currently run on the same VM.

## Granting Permissions

The MediaWiki user was granted privileges only on the MediaWiki database:

```sql
GRANT ALL PRIVILEGES ON librewiki.* TO 'librewiki'@'localhost';
```

The assigned privileges can be inspected with:

```sql
SHOW GRANTS FOR 'librewiki'@'localhost';
```

The goal is not to give the application global database privileges.

Instead:

```text
librewiki user
      │
      └── librewiki.*
```

is allowed, while unrelated databases remain outside the application's scope.

## Why Not Use `root` for MediaWiki?

Using the database root account for a web application would give the application far more privileges than it requires.

A compromised MediaWiki process using a root database account could potentially affect:

- other databases
- database users
- permissions
- administrative configuration

Using a dedicated account limits the scope of the application credentials.

The principle is:

> Give the service only the database access it needs.

## Testing the Application User

The database account can be tested from the shell:

```bash
mariadb -u librewiki -p librewiki
```

MariaDB prompts for the application password.

A successful connection should open:

```text
MariaDB [librewiki]>
```

The current database can be checked with:

```sql
SELECT DATABASE();
```

Expected result:

```text
librewiki
```

The connection can then be closed:

```sql
EXIT;
```

## Database Connection at This Stage

Because both MediaWiki and MariaDB run on the same machine, the application later uses:

```text
Database host: localhost
Database name: librewiki
Database user: librewiki
Database password: <secret>
```

The traffic does not need to leave the Ubuntu VM.

Current layout:

```text
Ubuntu Server
│
├── MediaWiki
│      │
│      │ localhost
│      ▼
└── MariaDB
       └── librewiki
```

## MariaDB Network Exposure

At this stage, MariaDB does not need to be exposed to other machines.

There is no reason to make TCP port:

```text
3306
```

publicly reachable.

Later, when MariaDB is moved to Rocky Linux, network access will be configured intentionally.

That future design will look more like:

```text
Ubuntu application server
192.168.x.x
       │
       │ TCP 3306
       ▼
Rocky Linux database server
192.168.x.y
```

At that point, the database should accept connections only from the application server rather than the entire network.

## Useful MariaDB Commands

Open the administrative shell:

```bash
sudo mariadb
```

Show databases:

```sql
SHOW DATABASES;
```

Show users:

```sql
SELECT User, Host FROM mysql.user;
```

Show grants:

```sql
SHOW GRANTS FOR 'librewiki'@'localhost';
```

Use the application database:

```sql
USE librewiki;
```

Show tables:

```sql
SHOW TABLES;
```

Exit:

```sql
EXIT;
```

## Service Management

Check MariaDB:

```bash
systemctl status mariadb
```

Start MariaDB:

```bash
sudo systemctl start mariadb
```

Stop MariaDB:

```bash
sudo systemctl stop mariadb
```

Restart MariaDB:

```bash
sudo systemctl restart mariadb
```

Reload configuration where applicable:

```bash
sudo systemctl reload mariadb
```

## Logs

MariaDB service logs can be inspected through systemd:

```bash
journalctl -u mariadb
```

Recent entries:

```bash
sudo journalctl -u mariadb -n 50
```

Live log monitoring:

```bash
sudo journalctl -u mariadb -f
```

These logs become useful when troubleshooting:

- startup failures
- configuration mistakes
- storage problems
- authentication failures

## Database Storage

MariaDB data is typically stored under:

```text
/var/lib/mysql
```

This directory contains database files managed by MariaDB.

The directory should not be treated like ordinary application files.

Database backups should be created using database-aware tools rather than simply copying a live database directory.

## Backup Planning

Database backup was not automated at this stage, but it is a planned part of the homelab.

A future logical backup can use:

```bash
mariadb-dump
```

For example:

```bash
mariadb-dump -u root librewiki > librewiki.sql
```

When using Unix socket root authentication, a privileged form may be more appropriate:

```bash
sudo mariadb-dump librewiki > librewiki.sql
```

The backup strategy will eventually cover:

```text
MariaDB database
MediaWiki files
LocalSettings.php
uploaded files
Nginx configuration
```

Creating a backup is only half of the exercise.

The homelab will also test restoring the wiki from backup.

## Planned Database Separation

The database currently runs on Ubuntu only to keep the first deployment simple.

A later stage will move MariaDB to a dedicated Rocky Linux VM.

The purpose is not because Rocky Linux is required by MariaDB.

It is an infrastructure exercise intended to provide experience with:

- RHEL-family Linux
- `dnf`
- `firewalld`
- SELinux
- network database authentication
- database host permissions
- service separation
- migration between hosts

Before separation:

```text
Ubuntu
├── Nginx
├── PHP-FPM
├── MediaWiki
└── MariaDB
```

After separation:

```text
Ubuntu
├── Nginx
├── PHP-FPM
└── MediaWiki
       │
       ▼
Rocky Linux
└── MariaDB
```

This progression follows the overall homelab principle:

> Start simple, then add infrastructure complexity when there is a concrete reason to do so.

## Future Firewall Model

Once the database is separated, the intended network policy is approximately:

```text
Internet
   │
   X
MariaDB :3306
   ▲
   │ allowed
   │
MediaWiki application server
```

The database should not be directly reachable from the public Internet.

Only the application host should be able to establish the required database connection.

This will later provide an opportunity to practice both host firewall configuration and network segmentation.

## Credentials and Git

Real database credentials must never be committed to the public repository.

For example, this should not be committed:

```php
$wgDBpassword = 'real-database-password';
```

Documentation and sample configuration should use:

```php
$wgDBpassword = 'CHANGE_ME';
```

Database dumps should also be excluded from Git by default:

```gitignore
*.sql
```

The real MediaWiki configuration file will be excluded separately.

## Current Architecture After This Stage

At the end of the MariaDB stage:

```text
MacBook Air M5
│
└── UTM
    │
    └── Ubuntu Server
        │
        ├── Nginx
        │
        ├── PHP-FPM 8.5
        │
        └── MariaDB
            │
            ├── MariaDB root
            │   └── unix_socket administration
            │
            └── librewiki database
                └── librewiki application user
```

The web application itself is installed in the next stage.

## Result

This stage is complete when:

- MariaDB Server is installed
- MariaDB is running
- administrative access works through `sudo mariadb`
- anonymous users are removed
- remote root login is disabled
- the test database is removed
- the `librewiki` database exists
- a dedicated `librewiki` database user exists
- the user has privileges only on the intended database
- the application credentials are not stored in Git

The next stage is installing MediaWiki and connecting it to the database created here.
