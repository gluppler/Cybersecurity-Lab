  ┌────────┐
  │Internet│
  └───┬────┘
      │
┌─────┴──────┐
│Host Machine│ Kernel Virtual Machine (KVM) on DMZ
└─────┬──────┘
      │
    ┌─┴─┐
    │NAT│ (default)
    └─┬─┘
      │
  ┌───┴───┐ ┌────────┐
  │PFSense├─┤Isolated│ LAN 10.0.1.1/24 (vmbr0)
  └───────┘ └───┬────┘
                │
             ┌──┴───┐
             │Remnux│ Static Analysis and Interception
             └──┬───┘
                │
            ┌───┴────┐
            │Isolated│ Analysis LAN 10.0.2.1/24 (vmbr1)
            └─┬──────┤
              │      │
      ┌───────┴───┐ ┌┴──────┐
      │Analysis VM│ │Windows│ Dynamic Analysis
      └───────────┘ └───────┘
