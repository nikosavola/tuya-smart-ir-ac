## Perf: make sensor "smart polling" actually work over Pulsar

The sensor coordinator is supposed to skip cloud polls when Pulsar already delivered
fresh data — but it recorded freshness using **raw** Tuya codes (`va_temperature`,
`va_humidity`) and compared them against **normalized** names (`temp_current`,
`humidity_value`), which never match. So it kept hitting the cloud anyway, burning
API quota (the opposite of the feature's intent).

Fix: normalize the codes via `TUYA_CODE_MAPPING` before recording freshness.
