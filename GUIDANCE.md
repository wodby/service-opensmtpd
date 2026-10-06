# OpenSMTPD on Wodby

What Wodby sets up for this OpenSMTPD service. It is a mail relay: applications hand their outgoing mail to it and it delivers the mail, directly or through an SMTP provider.

## How applications reach it

- SMTP: host is the name of this app service inside the environment, port `25`. No TLS, no username and no password; no token is generated. It accepts mail from any sender for any recipient, and is meant to be reached only from inside the environment.
- The service carries the `smtpd` label, so it satisfies the mail link of application services. A linked PHP service, for example, receives the host and port as `MSMTP_HOST` and `MSMTP_PORT` and PHP's `mail()` is delivered through it without code; other runtimes receive variables named by their own service, such as `SMTP_HOST` and `SMTP_PORT`. Read those variables; do not hardcode the host.
- SMTP provider credentials belong to this service, not to the application. Do not put a provider's host, user or password into application code or its variables.

## Where the mail goes

- Without a relay, OpenSMTPD delivers straight to the recipients' mail servers from the cluster's address. Recipient providers often reject such mail or mark it as spam.
- The manifest declares an `smtp` integration ("Third-party SMTP server for Relay"). When an SMTP provider integration is attached to it, Wodby supplies the relay host, port, protocol, user and password, and all mail is forwarded through that provider. In the container these are `RELAY_HOST`, `RELAY_PORT`, `RELAY_PROTO`, `RELAY_USER` and `RELAY_PASSWORD`.
- With a relay, the sender address must be one the provider accepts.

## Generated configuration

On every start the container renders `/etc/smtpd/smtpd.conf` from environment variables and, when a relay user is set, the credentials table next to it. Never edit them. Other variables read by the template: `OPENSMTPD_MAX_MESSAGE_SIZE` (default `35M`), `OPENSMTPD_EXPIRE` (how long undelivered mail stays queued, default `4d`) and `OPENSMTPD_BOUNCE_WARN`.

## Data

The mail queue is in `/var/spool/smtpd`. The `spool` volume is optional; without it, mail that is still queued is lost when the container is replaced. The manifest declares no backups, imports or actions.

## Check the result

From this service's container:

- `printf 'Subject: test\n\nRelay test.\n' | sendmail -v -f <accepted sender> <recipient>` shows whether the message was accepted.
- `smtpctl show queue` lists mail that has not been delivered yet. The service log shows the relay's answer for each delivery.
