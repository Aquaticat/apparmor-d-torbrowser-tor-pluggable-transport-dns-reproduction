# apparmor.d torbrowser-tor: pluggable transports cannot resolve names

This repository reproduces a problem in [apparmor.d](https://github.com/roddhjav/apparmor.d)'s `torbrowser-tor` profile,
for a report on that project's issue tracker.

The profile runs Tor Browser's pluggable transports (lyrebird, which carries Snowflake) inside Tor's own profile,
through `@{lib_dirs}/TorBrowser/Tor/PluggableTransports/** mix`.
That profile allows only `network inet stream` and denies `/etc/resolv.conf` and `/etc/nsswitch.conf`.
Snowflake has to resolve its domain front and its STUN servers, so it cannot connect.
`deny` rules are not logged, so the lookup failure is silent.

## What the workflow does

Both jobs build apparmor.d at commit `4c4fec70fcd19307a871b3ecc7f117a9ad989c9f` with its own `prebuild` tool
on Ubuntu 26.04, the only Ubuntu release that tool accepts,
puts `torbrowser-tor` in enforce mode and checks that it is
and install only `torbrowser-tor` with `just local torbrowser-tor`.

- **stock torbrowser-tor** runs `getent hosts app.datapacket.com` outside any profile (control),
  then under `torbrowser-tor`. The last step asserts the lookup works,
  so a red run on that step means the problem reproduced.
- **torbrowser-tor with the pluggable-transports child profile** applies
  [`torbrowser-tor-pluggable-transports.patch`](torbrowser-tor-pluggable-transports.patch) first.
  It asserts the lookup works under `torbrowser-tor//pluggable-transports`
  and still fails under `torbrowser-tor` itself.

`app.datapacket.com` is one of the domain fronts in Tor Browser's built-in Snowflake bridge lines.
