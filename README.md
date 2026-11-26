# apparmor.d torbrowser-tor: pluggable transports cannot resolve names

This repository reproduces a problem in [apparmor.d](https://github.com/roddhjav/apparmor.d)'s `torbrowser-tor` profile,
for a report on that project's issue tracker.

The profile runs Tor Browser's pluggable transports (lyrebird, which carries Snowflake) inside Tor's own profile,
through [`@{lib_dirs}/TorBrowser/Tor/PluggableTransports/** mix`][pt].
That profile allows only [`network inet stream`][net]
and denies [`/etc/nsswitch.conf` and `/etc/resolv.conf`][deny].
Snowflake's client names a [domain front][front] and [ICE server URLs][ice],
so it needs both name resolution and UDP, and it cannot connect.
`deny` rules are not logged, and they [are enforced even in complain mode][complain],
so the lookup failure is silent.

## What the workflow does

Both jobs build apparmor.d at commit `4c4fec70fcd19307a871b3ecc7f117a9ad989c9f` with its own `just build` and `prebuild`
on Ubuntu 26.04, the only Ubuntu release `prebuild` accepts,
install only `torbrowser-tor` with `just local torbrowser-tor`,
put it in enforce mode and check that it is.

- **stock torbrowser-tor** runs `getent hosts app.datapacket.com` outside any profile (control),
  then under `torbrowser-tor`. The last step asserts the lookup works,
  so a red run on that step means the problem reproduced.
- **torbrowser-tor with the pluggable-transports child profile** applies
  [`torbrowser-tor-pluggable-transports.patch`](torbrowser-tor-pluggable-transports.patch) first.
  It asserts the lookup works under `torbrowser-tor//pluggable-transports`
  and still fails under `torbrowser-tor` itself.

The lookup target only has to be a public name;
`app.datapacket.com` is the domain front in the Snowflake bridge lines on the reporting machine.

[pt]: https://github.com/roddhjav/apparmor.d/blob/4c4fec70fcd19307a871b3ecc7f117a9ad989c9f/apparmor.d/groups/browsers/torbrowser-tor#L25
[net]: https://github.com/roddhjav/apparmor.d/blob/4c4fec70fcd19307a871b3ecc7f117a9ad989c9f/apparmor.d/groups/browsers/torbrowser-tor#L18
[deny]: https://github.com/roddhjav/apparmor.d/blob/4c4fec70fcd19307a871b3ecc7f117a9ad989c9f/apparmor.d/groups/browsers/torbrowser-tor#L41-L44
[front]: https://pkg.go.dev/gitlab.torproject.org/tpo/anti-censorship/pluggable-transports/snowflake/v2@v2.15.4/client/lib#:~:text=FrontDomain%20is%20the%20full%20URL%20of%20an%20optional%20front%20domain
[ice]: https://pkg.go.dev/gitlab.torproject.org/tpo/anti-censorship/pluggable-transports/snowflake/v2@v2.15.4/client/lib#:~:text=ICEAddresses%20are%20a%20slice%20of%20ICE%20server%20URLs
[complain]: https://github.com/roddhjav/apparmor.d/blob/4c4fec70fcd19307a871b3ecc7f117a9ad989c9f/docs/issues.md#L26
