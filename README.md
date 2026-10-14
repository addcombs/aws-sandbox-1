# aws-sandbox-1

Minimal Java 17 / Spring Boot application connecting to an existing PostgreSQL
database on AWS EC2 using JDBC. It does not create databases or modify schemas.

## Configure the EC2 database

The database address and credentials are not stored in this repository. Obtain
them from your existing AWS setup.

- Prefer running the app in the same VPC as PostgreSQL and use its private DNS
  name. Allow TCP 5432 in the database security group **only from the app's
  security group**, not from `0.0.0.0/0`. For access from your workstation, use
  a VPN or SSH tunnel rather than exposing PostgreSQL publicly.
- PostgreSQL must listen on the private interface (`listen_addresses`), and its
  `pg_hba.conf` must allow the app's database/user and source address. Use
  `hostssl` rules with SCRAM authentication for remote connections.
- Use an existing database and a dedicated, least-privileged login, not the
  PostgreSQL superuser.
- TLS defaults to `verify-full`: enable TLS on PostgreSQL, use a hostname matching
  its certificate, and install the issuing CA at `~/.postgresql/root.crt` for the
  OS user running the app. Do not disable verification for remote connections.

## Build and run

Requires JDK 17+ and Maven 3.6.3+.

```bash
mvn verify
export DB_HOST='your-database-private-dns'
export DB_PORT='5432'                  # optional; defaults to 5432
export DB_NAME='your-existing-database'
export DB_USERNAME='your-app-user'
read -r -s -p 'Database password: ' DB_PASSWORD; echo
export DB_PASSWORD
java -jar target/aws-sandbox-1-0.0.1-SNAPSHOT.jar
```

Alternatively, run with `mvn spring-boot:run` after exporting the variables.
The app binds to `127.0.0.1:8080` by default. Set `SERVER_PORT` to change the port.
Set `SERVER_ADDRESS` to a private interface address if other hosts must connect,
and restrict the app's security group accordingly.

For **local-only** development against PostgreSQL without TLS, explicitly export
`DB_SSLMODE=disable`. The default is `verify-full`.

In deployment, inject credentials using your secret-management/service
configuration (for example, AWS Secrets Manager); do not commit them or put them
in shell history. `.env` files are ignored by Git but are **not** automatically
loaded by Spring Boot.

## Verify connectivity

```bash
curl --fail http://127.0.0.1:8080/actuator/health
```

`{"status":"UP"}` means the application and its database health check succeeded.
An unreachable database or invalid credentials produces HTTP 503 with
`{"status":"DOWN"}`; starting the web server alone does not prove connectivity.
Only the health endpoint is exposed, without database or credential details.
No business API or schema is assumed.

If connectivity fails, check the VPC route, security group, host firewall,
PostgreSQL listener/`pg_hba.conf`, database name, credentials, and TLS CA/hostname.
The automated tests use an in-memory test-only H2 database; use the health check
above to verify your actual PostgreSQL connection.