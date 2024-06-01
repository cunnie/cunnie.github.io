---
title: "Debugging the vSphere API via the BOSH vSphere CPI from Your Workstation"
date: 2024-01-01T14:51:43-08:00
draft: false
---

### This Blog Post Is Not For You

This blog post is directed towards people who are working with the BOSH vSphere
CPI (Cloud Provider Interface), which is not you. There are more
interesting things to read. If you want suggestions, try
_[Ulysses](https://www.poetryfoundation.org/poems/45392/ulysses)_ by Sir Alfred
Lord Tennyson, a poem about an aged hero seeking to recapture his adventures of
youth.

### Challenge: Extending the Size of the Root Disk

I'd like to extend the size of the root partition, but it's a challenge: if I
don't make exactly the correct vSphere API calls, my changes will fail, and the
feedback loop is slow (hot-patching a BOSH Director and then running a deploy
takes at least 5 minutes each attempt).

What I really want is a REPL ([Read–eval–print
loop](https://en.wikipedia.org/wiki/Read%E2%80%93eval%E2%80%93print_loop)) so I
can test changes quickly. I want
[`pry-byebug`](https://www.rubydoc.info/gems/pry-byebug/3.10.1).

### How the CPI Works

The CPI entry is a shell script, `/var/vcap/jobs/vsphere_cpi/bin/cpi`, which is
a wrapper around a Ruby executable,
`/var/vcap/packages/vsphere_cpi/bin/vsphere_cpi`

The first argument passed to the CPI is a pathname,
`/var/vcap/jobs/vsphere_cpi/config/cpi.json`. These are its contents (at least
on my BOSH Director):

```json
{
  "cloud": {
    "plugin": "vsphere",
    "properties": {
      "vcenters": [
        {
          "host": "vcenter-80.nono.io",
          "user": "administrator@vsphere.local",
          "password": "some-crazy-password",
          "datacenters": [
            {
              "name": "dc",
              "vm_folder": "bosh-vsphere-vms",
              "template_folder": "bosh-vsphere-templates",
              "disk_path": "bosh-vsphere-disks",
              "allow_mixed_datastores": true,
              "clusters": [
                {
                  "cl": {
                    "resource_pool": "BOSH"
                  }
                }
              ],
              "datastore_pattern": "NAS-0",
              "persistent_datastore_pattern": "NAS-0"
            }
          ],
          "default_disk_type": "preallocated",
          "enable_auto_anti_affinity_drs_rules": false,
          "memory_reservation_locked_to_max": false,
          "upgrade_hw_version": false,
          "enable_human_readable_name": true,
          "http_logging": false,
          "connection_options": {
            "ca_cert": null
          }
        }
      ],
      "agent": {
        "ntp": [
          "0.pool.ntp.org",
          "1.pool.ntp.org"
        ],
        "mbus": "nats://73.189.219.4:4222"
      }
    }
  }
}
```

But wait, there's more: the BOSH Director passes in, _via STDIN_, the command
that it'd like the CPI to perform (e.g. Create a VM, delete a disk, etc.). This
is also in JSON.

Here's a truncated sample of a `create_vm` call which, I've saved to a file,
`stdin.json`:

```json
{
  "method": "create_vm",
  "arguments": [
    "882a8a0c-5df4-4e4b-bbe0-292ff1b63c01",
    "sc-fb254d39-49a8-44b9-baa7-a36537a7a4a0",
    {
      "datacenters": [
        {
          "clusters": [
            {
              "cl": {
                "resource_pool": "BOSH"
              }
            }
          ],
          "name": "dc"
        }
      ],
      "cpu": 2,
      "disk": 10240,
      "ram": 2048,
      "vmx_options": {
        "disk.enableUUID": "1"
      }
    },
```

A reasonable question to ask is, "Brian, how did you get the `create_vm` JSON?"
I'm glad you asked. I added the following lines of Ruby to
[`/var/vcap/packages/vsphere_cpi/bin/vsphere_cpi`](https://github.com/cloudfoundry/bosh-vsphere-cpi-release/blob/b95bac3d8cf50e1332663684336c30ccd1a492a7/src/vsphere_cpi/bin/vsphere_cpi#L38).
Put this after the `cpi_rpc_api_request_raw = ARGF.read` on line 36:

```Ruby
File.open("/tmp/#{Process.pid}.json", 'w') do |file|
  file.puts cpi_rpc_api_request_raw
end
```

I ran a deploy which created a VM: `bosh -d dummy deploy dummy.yml`.

I checked the `/tmp` directory on the Director for JSON files until I found
the one I wanted. I saved the file as `stdin.json`.

### Setting the Breakpoint

We're ready to start debugging. Let's edit
`src/vsphere_cpi/lib/cloud/vsphere/vm_creator.rb` and insert the following line
where we want to dynamically test things:

```ruby
require 'pry-byebug'; binding.pry
```

Let's run the CPI from our workstation, passing the two JSON files as
arguments:

```
cd ~/workspace/bosh-vsphere-cpi-release/src/vsphere_cpi/
bundle
bundle exec bin/vsphere_cpi cpi.json stdin.json
```

The previous command will create the VM while printing a slew of log messages
to STDERR, and then will drop you into the pry-byebug REPL. Let's do some quick
debugging:

```ruby
device_specification = Resources::VM.create_edit_device_spec(created_vm.system_disk)
 # Let's see what methods this newly-created object has:
device_specification.methods.sort - 1.methods
 # Hit ^D when you've finished troubleshooting to continue execution
```

### Clean-up: Delete the Created VM

You'll need to delete anything created by your testing because we're operating
_outside_ the BOSH Director. In our example, we've been testing the `create_vm`
method, so we'll need to delete the VM (using the vCenter web UI or `govc`). To
find the VM to delete, examine the JSON emitted by the CPI (it should be the
last or second-to-last line of output); it'll show the ID of any created
artifacts. In the example below, the created VM's ID is
`jammy_dummy_364cf7e0c4fc`:

```json
{
  "result": [
    "jammy_dummy_364cf7e0c4fc",
    {
      "vsphere-guest": {
        "type": "manual",
        "ip": "10.9.2.37",
        "netmask": "255.255.254.0",
        "cloud_properties": {
          "name": "guest"
        },
        "default": [
          "dns",
          "gateway"
        ],
        "dns": [
          "10.9.2.1",
          "8.8.8.8"
        ],
        "gateway": "10.9.2.1"
      }
    }
  ],
  "error": null,
  "log": ""
}
```

References:

- [`cpi.json`](/assets/cpi.json)
- [`stdin.json`](/assets/stdin.json)

### Debugging Integration Tests

Kick off a build, e.g. <https://ci.bosh-ecosystem.cf-app.com/teams/vsphere-cpi/pipelines/vsphere-cpi/jobs/lifecycle-7.0-nsxt31/builds/latest>.

Intercept it:

```bash
fly -t bosh i \
    --team vsphere-cpi \
    -u https://ci.bosh-ecosystem.cf-app.com/teams/vsphere-cpi/pipelines/vsphere-cpi/jobs/lifecycle-7.0-nsxt31/builds/252
```

Choose the "test" task. Set up your environment.

```
apt update && apt install -y git neovim less fd-find
```

Source in the [necessary environment variables](https://github.com/cloudfoundry/bosh-vsphere-cpi-release/blob/master/ci/tasks/run-lifecycle.sh#L6-L7):

```bash
source bosh-cpi-src/.envrc
source environment/metadata
export PATH="$PATH:$PWD/bosh-cpi-src/src/iso9660wrap"
```

Scan the output of the integration tests running in the browser: look for the
stemcell ID and set that (this saves a lot of time uploading):

```bash
export BOSH_VSPHERE_STEMCELL_ID="sc-9f526a87-9e9b-44a4-935c-321e141c5f64"
```

Edit your integration test. This is a good time for strategic use of `require
'pry-byebug'; binding.pry`:

```bash
nvim bosh-cpi-src/src/vsphere_cpi/spec/integration/root_size_disk_gb_spec.rb
```

Now run _only_ your integration test. You don't want to run _all_ the tests
because it can take more than four hours, and you probably don't have the
patience of the Buddha:

```bash
pushd bosh-cpi-src/src/vsphere_cpi
bundle exec rspec \
    ${RSPEC_FLAGS:-''} \
    --require ./spec/support/verbose_formatter.rb \
    --format VerboseFormatter \
    spec/integration/root_size_disk_gb_spec.rb
```

### Updates/Errata

**2024-06-01** More precise instructions.

**2024-01-26** Added _Debugging Integration Tests_.

**2024-01-21** Minor style changes.
