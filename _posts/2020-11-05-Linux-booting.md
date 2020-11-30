### BIOS

- CPU wakes up with default values 
- First instruction it finds at reset vercor (With all the initial values in `CS` there is the address `0xfffffff0`, which is 16 bytes below 4GB called reset vector position.)
- BIOS selects boot device and loads boot loader
- boot loader split between MBR and first sector of the booting device



### Boot loader

- Boot loader fill in some headers in the linker code and loads kernel boot sector market with magic header `MZ`
- 