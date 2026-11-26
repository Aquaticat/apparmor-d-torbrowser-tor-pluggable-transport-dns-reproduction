# apparmor.d torbrowser-tor: pluggable transports cannot resolve names

This repository reproduces a problem in [apparmor.d](https://github.com/roddhjav/apparmor.d)'s `torbrowser-tor` profile,
for a report on that project's issue tracker.

The profile runs Tor Browser's pluggable transports (lyrebird, which carries Snowflake) inside Tor's own profile,
through [`@{lib_dirs}/TorBrowser/Tor/PluggableTransports/** mix`][pt].
That profile's only network rules are [`network inet stream` and `network netlink raw`][net],
and it denies [`/etc/nsswitch.conf` and `/etc/resolv.conf`][deny].
Snowflake's client names a [domain front][front] and [ICE server URLs][ice],
so it needs both name resolution and UDP, and it cannot connect.
A `deny` rule [denies without logging][quiet], and `deny` rules [are enforced even in complain mode][complain],
so the lookup failure is silent.

## Expected result

- ❌ The stock profile fails (red): its last step, the lookup under `torbrowser-tor`, fails, which reproduces the problem.
- ✅ The profile with the pluggable-transports child passes (green): the patch fixes the lookup.

GitHub shows the stock job with a red cross and the other with a green check,
so a run whose overall status is "failure" is the expected outcome.

## Steps both jobs share

Both jobs build apparmor.d at commit [`4c4fec7`][commit]
with its own [`just build`][jbuild] and [`prebuild`][jprebuild]
on Ubuntu 26.04, the only Ubuntu release `prebuild` [accepts][ubuntu],
install only `torbrowser-tor` with [`just local torbrowser-tor`][jlocal],
put it in enforce mode and check that it is.

The lookup target only has to be a public name;
`app.datapacket.com` is the name lyrebird failed to resolve on the reporting machine.

## ❌ Stock profile: fails (red)

The job `stock torbrowser-tor (a red run means the bug reproduced)`
runs `getent hosts app.datapacket.com` outside any profile (control),
then under `torbrowser-tor`.
Its last step asserts the lookup works,
so a red run on that step means the problem reproduced.

## ✅ Profile with the pluggable-transports child: passes (green)

The job `torbrowser-tor with the pluggable-transports child profile`
applies [`torbrowser-tor-pluggable-transports.patch`](torbrowser-tor-pluggable-transports.patch) first.
It asserts the lookup works under `torbrowser-tor//pluggable-transports`
and still fails under `torbrowser-tor` itself.

[pt]: https://github.com/roddhjav/apparmor.d/blob/4c4fec70fcd19307a871b3ecc7f117a9ad989c9f/apparmor.d/groups/browsers/torbrowser-tor#L25
[net]: https://github.com/roddhjav/apparmor.d/blob/4c4fec70fcd19307a871b3ecc7f117a9ad989c9f/apparmor.d/groups/browsers/torbrowser-tor#L18-L19
[deny]: https://github.com/roddhjav/apparmor.d/blob/4c4fec70fcd19307a871b3ecc7f117a9ad989c9f/apparmor.d/groups/browsers/torbrowser-tor#L41-L44
[front]: https://pkg.go.dev/gitlab.torproject.org/tpo/anti-censorship/pluggable-transports/snowflake/v2@v2.15.4/client/lib#:~:text=FrontDomain%20is%20the%20full%20URL%20of%20an%20optional%20front%20domain
[ice]: https://pkg.go.dev/gitlab.torproject.org/tpo/anti-censorship/pluggable-transports/snowflake/v2@v2.15.4/client/lib#:~:text=ICEAddresses%20are%20a%20slice%20of%20ICE%20server%20URLs
[quiet]: https://manpages.debian.org/forky/apparmor/apparmor.d.5.en.html#:~:text=denied%20without%20logging
[complain]: https://github.com/roddhjav/apparmor.d/blob/4c4fec70fcd19307a871b3ecc7f117a9ad989c9f/docs/issues.md#L26
[commit]: https://github.com/roddhjav/apparmor.d/commit/4c4fec70fcd19307a871b3ecc7f117a9ad989c9f
[jbuild]: https://github.com/roddhjav/apparmor.d/blob/4c4fec70fcd19307a871b3ecc7f117a9ad989c9f/Justfile#L88-L94
[jprebuild]: https://github.com/roddhjav/apparmor.d/blob/4c4fec70fcd19307a871b3ecc7f117a9ad989c9f/Justfile#L101-L104
[jlocal]: https://github.com/roddhjav/apparmor.d/blob/4c4fec70fcd19307a871b3ecc7f117a9ad989c9f/Justfile#L180-L203
[ubuntu]: https://github.com/roddhjav/apparmor.d/blob/4c4fec70fcd19307a871b3ecc7f117a9ad989c9f/cmd/prebuild/main.go#L34-L40
