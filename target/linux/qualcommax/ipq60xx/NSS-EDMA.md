# IPQ60xx NSS-EDMA Bring-Up Notes

This target is in staged NSS-EDMA bring-up. The first enabled board is
JDCloud RE-CS-02 (`jdcloud,re-cs-02`, `qcom,ipq6018`). Keep the host
EDMA/PPE path as the fallback until each hardware level below is verified.

## Measured IPQ6018 Constants

- NSS core count: one core, `qca-nss0` only.
- Firmware: `qca-nss0-retail.bin`; no `qca-nss1` firmware is required.
- Firmware line observed on traditional NSS hardware: `NSS.CP.11.4.0.5-6-R`.
- Reserved memory: `0x40000000..0x40ffffff`, size `0x01000000`, `no-map`.
- NSS node: `qcom,nss`, `qcom,id = <0>`, `qcom,load-addr = <0x40000000>`.
- NSS queues/IRQs/priorities: `qcom,num-queue = <4>`, `qcom,num-irq = <10>`, `qcom,num-pri = <4>`.
- NSS interrupts: GIC SPI `402` down to `393`, edge rising.
- NSS register names: `nphys`, `qgic-phys`; do not use the IPQ807x `vphys` region.
- NSS-DP external ports: phys_if `1..5` only.
- RE-CS-02 port map: `1=lan1`, `2=lan2`, `3=lan3`, `4=lan4`, `5=wan`.
- Traditional IPQ60xx NSS-DP constants: `NSS_DP_HAL_MAX_PORTS=5`, `NSS_DP_HAL_START_IFNUM=1`, `NSS_DP_MAX_INTERFACES=6`, `NSS_DP_QUEUE_NUM=4`, `NSS_DP_PREHEADER_SIZE=32`.

## MACsec DTS Note

The `nss-macsec0` `phy_addr` in `ipq6018-nss.dtsi` is not the board WAN PHY
MDIO address. RE-CS-02 WAN is the QCA8081 at MDIO address `0x0c`, while the
reference IPQ6018 NSS DTS keeps `nss-macsec0 phy_addr = <0x18>`. The current
NSS-EDMA bring-up does not carry or load `qca-nss-macsec`; wired datapath
validation must use the board port map above and should not infer MACsec
settings from the WAN PHY address.

## Bring-Up Levels

- Level A: host EDMA/PPE only, NSS disabled.
- Level B: NSS DTS and firmware present, no automatic arming.
- Level C: `qca-ppe-nss` loaded, debugfs status visible, no traffic offload.
- Level D: one external wired port armed manually.
- Level E: all external wired ports `1..5` armed manually.
- Level F: bridge, ECM NAT/routing, PPPoE, VLAN, and NSS qdisc enabled one layer at a time.

## Default Safety Policy

For `qualcommax/ipq60xx`, `nss-tools` defaults `nss.general.enabled=0` and
`nss.general.wifi_offload=0`. This prevents automatic NSS firmware arming and
keeps Wi-Fi NSS offload out of the first wired validation images.

Enable wired NSS manually only after the host path is clean:

```sh
uci set nss.general.enabled='1'
uci set nss.general.wifi_offload='0'
uci commit nss
/etc/init.d/nss start
```

## Human Build Report Required

The agent must not compile in this repository. The human build report should
include:

```sh
./scripts/feeds update -a
./scripts/feeds install -a
make menuconfig
make -j$(nproc)
```

Record target, selected NSS packages, firmware package, DTC warnings/errors,
and any package conflicts.

## Hardware Validation Checklist

Level A:

```sh
ip -d link
dmesg | grep -Ei 'qca|edma|ppe|nss|warn|oops|panic'
```

Expected: WAN/LAN work on host EDMA/PPE before NSS is enabled.

Level B/C:

```sh
modprobe qca-ppe-nss
cat /sys/kernel/debug/qca-ppe-nss/status
dmesg | grep -Ei 'qca-nss|NSS|core 0|core 1|qca-nss1'
```

Expected: IPQ60xx reports phys_if `1..5`, no armed ports, no `qca-nss1` or
core1 boot attempt.

Level D:

```sh
echo 0x2 > /sys/kernel/debug/qca-ppe-nss/fw_mask
cat /sys/kernel/debug/qca-ppe-nss/status
nss-status -l2
dmesg | grep -Ei 'qca-ppe-nss|qca-nss|warn|oops|panic'
```

Expected: exactly one external port is armed, no hang, no kernel WARN/Oops.

Level E:

```sh
echo 0x3e > /sys/kernel/debug/qca-ppe-nss/fw_mask
cat /sys/kernel/debug/qca-ppe-nss/status
nss-status -l2
```

Expected: ports `1..5` only; port 6/7 must not be armed on IPQ60xx.

Level F:

Validate in order: LAN bridge, IPv4/IPv6 routing, NAT/ECM, PPPoE, VLAN, then
NSS qdisc/SQM. Capture `nss-status`, ECM counters, `tc -s qdisc`, link flap
logs, and reboot-cycle results for each layer.
