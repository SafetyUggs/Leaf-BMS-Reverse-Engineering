4FA2B is eNV-200 24kwh FW, without battery heater.

XML contains all tables within 4FA2B firmware. Use ECUFLASH to edit. Checksums must be corrected before flashing a modified firmware. Capacity changes are not part of this XML, there are many constants that need modifying at the ASM level, as well as coulomb counter functions and integrators that need scaling to prevent overflow of AH*S variables. If Scaled, any functions that use or report the scaled variables must also be modified.

UDS services are complete as possible

Cyclic Can frames are documented as complete as possible

Find this useful? Buy me a coffee - help support development : https://ko-fi.com/safetyuggs
