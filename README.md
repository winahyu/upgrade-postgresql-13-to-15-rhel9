# Upgrade PostgreSQL 13 to 15 on RHEL 9 manually

Use the step below when the simpler path described here: https://access.redhat.com/solutions/7102084  is not possible for any reason. 

## Preparation

Existing installation:

- `postgresql-server` 13.xx
- Data directory: `/var/lib/pgsql/`
- Binary: `/usr/bin/`

Install PostgreSQL 15 from the PostgreSQL repo:

```bash
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo dnf install -y postgresql15-server
sudo /usr/pgsql-15/bin/postgresql-15-setup initdb
```

This provides an installation with:

- `postgresql-15` systemd unit
- Data directory: `/var/lib/pgsql/15/`
- Binary: `/usr/pgsql-15/bin/`

## Start the migration

Stop the PostgreSQL 13 server:

```bash
sudo systemctl stop postgresql
```

Review the current config (for example `listen_addresses`, `max_connections`, and `local connections`) in below files, will add the same config to the new installation:

- `/var/lib/pgsql/data/pg_hba.conf`
- `/var/lib/pgsql/data/postgresql.conf`

Run the data migration sanity check:

```bash
sudo -i -u postgres /usr/pgsql-15/bin/pg_upgrade \
  --old-datadir=/var/lib/pgsql/data \
  --new-datadir=/var/lib/pgsql/15/data \
  --old-bindir=/usr/bin \
  --new-bindir=/usr/pgsql-15/bin \
  --check
```

Make sure the result includes **Clusters are compatible** and every line is `OK`.

Run the data migration:

```bash
sudo -i -u postgres /usr/pgsql-15/bin/pg_upgrade \
  --old-datadir=/var/lib/pgsql/data \
  --new-datadir=/var/lib/pgsql/15/data \
  --old-bindir=/usr/bin \
  --new-bindir=/usr/pgsql-15/bin \
  --link
```

Wait until **Upgrade Complete** appears and every line is `OK`.

## Clean up PostgreSQL 13 and PostgreSQL 15 packages

Data remains intact with this process:

```bash
sudo systemctl stop postgresql-15
sudo dnf remove -y postgresql15-server
sudo dnf remove -y postgresql-server
sudo dnf remove -y pgdg-redhat-repo-42.0-64.rhel9PGDG.noarch
```

Reconfigure the data path to remove the version 13 layout and make version 15 the default:

```bash
sudo -i -u postgres
cd /var/lib/pgsql/
mv data data_13   # can be removed later
mv 15/* .
rm -Rf 15
exit
```
Update `/var/lib/pgsql/data/pg_hba.conf` and `/var/lib/pgsql/data/postgresql.conf` to satisfy previous configuration

## Install PostgreSQL 15 from the RHEL repo

```bash
sudo dnf module enable postgresql:15 -y
sudo dnf install postgresql-server
sudo systemctl enable --now postgresql
```

## Check connection

```bash
sudo -i -u postgres psql
```

It should show the client version, for example: `psql (15.18)`.

```bash
sudo -i -u postgres psql -c "show server_version;"
```

Expected output:

```text
 server_version
----------------
 15.18
```
and use list command to verify the database.

```
sudo -i -u postgres psql -c "\list+"
```

## PostgreSQL suggestion

```bash
sudo -i -u postgres vacuumdb --all --analyze-in-stages
```
