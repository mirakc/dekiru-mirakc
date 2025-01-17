# v2からv3への移行

ここでは，v2からv3への移行について説明します．

## config.yml {#config-yml}

`tuners[].dedicated-for`が廃止され，`timeshift.recorders[].uses`および
`onair-program-trackers[].local.uses`が追加されました．

```yaml
# v2
tuners:
  - name: gr-tracker
    types: [GR]
    command: ...
    dedicated-for: gr-tracker
  - name: timeshift-etv
    types: [GR]
    command: ...
    dedicated-for: 'timeshift#etv'

timeshift:
  recorders:
    etv:
      service-id: 3273701032
      ...

onair-program-trackers:
  gr-tracker:
    local:
      channel-types: [GR]

# v3
tuners:
  - name: gr-tracker
    types: [GR]
    command: ...
  - name: timeshift-etv
    types: [GR]
    command: ...

timeshift:
  recorders:
    etv:
      service-id: 3273701032
      ...
      uses:
        tuner: timeshift-etv
        channel-type: GR
        channel: '26'

onair-program-trackers:
  gr-tracker:
    local:
      channel-types: [GR]
      uses:
        tuner: gr-tracker
```
