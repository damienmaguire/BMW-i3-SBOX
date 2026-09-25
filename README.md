# BMW-i3-SBOX
Reverse engineering of the BMW i3 EV HV Contactor box (SBOX) for use in EV Conversion Projects.

<img width="4096" height="2304" alt="sboxonbench" src="https://github.com/user-attachments/assets/29b625ea-692f-41de-bc9c-d615abcd308e" />

The SBOX connects to the SME (bms) via a white 12 pin connector. 

Pin 1 - yellow - Positive contactor +12v supply.

Pin 2 - black - Positive contactor 12v negative. Connect to Pos contactor pin on Zombieverter VCU.

Pin 3 - grey - Precharge relay +12v supply.

Pin 4 - green - Precharge relay 12v negative. Connect to precharge contactor pin on Zombieverter VCU.

Pin 5 - blue - Negative contactor +12v supply.

Pin 6 - brown - Negative contactor 12v negative. Connect to neg contactor pin on Zombieverter VCU.

Pin 7 - orange - +12v supply to SBOX electronics.

Pin 8 - violet - Ground.

Pin 9 - pink - CAN Low. Connect to zombieverter shunt can. NOTE - SBOX has 120R termination resistor.

Pin 10 - yellow - CAN High. Connect to zombieverter shunt can. NOTE - SBOX has 120R termination resistor.

Pin 11 - No connection.

Pin 12 - No connection.

SBOX CAN bus is 500k.

