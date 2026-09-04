# Repository map

ChurchBridge is split across repositories with independent release cycles. This
page is the authoritative map of what is public, what is not, and why.

## Public

| Repository | Contents | Release cycle |
| --- | --- | --- |
| [`hainesdev/churchbridge`](https://github.com/hainesdev/churchbridge) | This meta-repository: product story, architecture overview, repository map, release manifest | Documentation only |
| [`hainesdev/churchbridge-platform`](https://github.com/hainesdev/churchbridge-platform) | Web client, FastAPI backend, translation pipeline, sanctuary display, mobile listener PWA, deployment tooling, benchmark harness | Continuous deployment to production |
| [`hainesdev/churchbridge-ios`](https://github.com/hainesdev/churchbridge-ios) | Native SwiftUI iPhone app for production-quality voice capture | Xcode Cloud and TestFlight |

The platform and the iOS app are **not** wired together with Git submodules.
Their release, deployment, and signing lifecycles are genuinely independent, and
coupling them in Git would create false constraints in both directions. The
relationship between versions is recorded as documentation instead, in
[`../releases/components.yml`](../releases/components.yml).

## Not public, and why

| Component | Why it stays private |
| --- | --- |
| Shared edge infrastructure | Runs the reverse proxy, certificates, and other sites on the same host. Contains operational material that has no public value and real risk. |
| macOS/iOS VM tooling | Local build automation carrying host names, credentials, and pairing configuration. If public tooling is ever wanted, it will be a fresh repository containing only redacted scripts and sample configuration — not this workspace. |
| Production operations runbook | Server addresses, filesystem paths, deploy-key setup, recovery procedures. Public deployment *architecture* and environment-variable *names* live in the platform repository; values and topology do not. |

## Third-party components

ChurchBridge depends on third-party projects that are not ChurchBridge code and
are not presented as such. Notably, the macOS virtualization used for iOS builds
is [sickcodes/Docker-OSX](https://github.com/sickcodes/Docker-OSX), an upstream
project used as a dependency — not forked, not rebranded.

Each public repository carries its own third-party notices covering runtime
dependencies, model assets, and any bundled data.
