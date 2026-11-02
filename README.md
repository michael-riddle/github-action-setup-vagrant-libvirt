# github-action-setup-vagrant-libvirt

Install Vagrant, the vagrant-libvirt provider and libvirt on an Ubuntu
GitHub Actions runner, for Beaker acceptance tests with
`BEAKER_HYPERVISOR=vagrant_libvirt`.

Ubuntu stopped shipping `vagrant` and `vagrant-libvirt` after 23.10.
[`ppa:evgeni/vagrant`](https://launchpad.net/~evgeni/+archive/ubuntu/vagrant)
carries matching builds of both, with Vagrant 2.3.7, the last MPL-licensed
release. Installing from the PPA alone fails the job whenever Launchpad is
down, so this action tries, in order:

1. **The Actions cache**: the PPA's packages, saved by a `cache-only` run on
   the default branch. No request goes to Launchpad. Their dependencies come
   from the Ubuntu archive.
2. **The PPA**: `add-apt-repository`, then a refresh of the PPA's index alone
   and the install. Each step is tried three times, 30s and then 60s apart.
3. **HashiCorp**: `vagrant` from <https://apt.releases.hashicorp.com>, pinned
   to `vagrant-version`, and `vagrant plugin install vagrant-libvirt`, pinned
   to `vagrant-libvirt-version` and built against `libvirt-dev`. This is
   BSL-licensed Vagrant, so the job gets a warning.

On an Ubuntu series the PPA doesn't publish for (`ppa-series`), the action
goes straight to HashiCorp with a notice.

## Usage

In the acceptance job:

```yaml
      - name: Setup libvirt for Vagrant
        uses: simp/github-action-setup-vagrant-libvirt@v1.0.0
      - name: beaker
        env:
          BEAKER_HYPERVISOR: 'vagrant_libvirt'
          VAGRANT_DEFAULT_PROVIDER: 'libvirt'
        run: bundle exec rake beaker:suites[default,almalinux9]
```

GitHub only lets a pull request restore caches saved from its own ref, its
base branch or the default branch, so a cache saved by one pull request is
no use to the next. Save the packages from the default branch with a
separate workflow, using a runner on a series the PPA publishes for:

```yaml
name: Vagrant package cache

'on':
  push:
    branches: [main]
  schedule:
    - cron: '17 6 * * 1,4'
  workflow_dispatch: {}

jobs:
  cache:
    runs-on: ubuntu-24.04
    steps:
      - uses: simp/github-action-setup-vagrant-libvirt@v1.0.0
        with:
          cache-only: 'true'
```

A `cache-only` run looks up the packages the PPA currently provides and
derives the cache key from their file names. If that key is already cached,
it restores it, which also keeps GitHub from evicting it after seven days
without use. Otherwise it downloads the packages and saves them. A new PPA
build gets a new key, and installs restore the newest key for their Ubuntu
series and architecture. The run fails if the PPA can't be reached after
the retries.

## Inputs

| Input | Default | Description |
|---|---|---|
| `cache-only` | `'false'` | `'true'` saves the PPA's packages to the Actions cache instead of installing anything |
| `cache` | `'true'` | `'false'` ignores the Actions cache when installing |
| `ppa-series` | `'noble'` | Space-separated Ubuntu codenames the PPA publishes for |
| `vagrant-version` | `'2.4.9-1'` | HashiCorp's apt version of `vagrant`, for the fallback |
| `vagrant-libvirt-version` | `'0.12.2'` | The `vagrant-libvirt` plugin version for the fallback |

## Outputs

| Output | Description |
|---|---|
| `source` | Install: `cache`, `ppa` or `hashicorp`. Cache-only: `cached` (already cached), `saved`, or `none` (the PPA doesn't publish for this series) |

## License

[Apache-2.0](LICENSE)
