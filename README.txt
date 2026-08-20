Four-tap Fourier Transform 

The top-level layout can be found in ring.mag

This circuit will implement a fast Fourier transform algorithm. Given four time-domain complex numbers, the circuit outputs four complex, frequency-domain Fourier coefficients.

Use the table below to load in each 6-bit input one at a time by setting the corresponding write address and ensuring WEN is HIGH:
+-----+-----+-----+-----+--------+
| WEN | WA0 | WA1 | WA2 | Output |
+-----+-----+-----+-----+--------+
|  1  |  1  |  1  |  1  | x3,im  |
|  1  |  1  |  1  |  0  | x1,im  |
|  1  |  1  |  0  |  1  | x3,re  |
|  1  |  1  |  0  |  0  | x1,re  |
|  1  |  0  |  1  |  1  | x2,im  |
|  1  |  0  |  1  |  0  | x0,im  |
|  1  |  0  |  0  |  1  | x2,re  |
|  1  |  0  |  0  |  0  | x0,re  |
+-----+-----+-----+-----+--------+

Once the computations are complete and DONE outputs HIGH, use the table below to read out 2 6-bit outputs at a time by setting the corresponding read address and ensuring REN is HIGH

+-----+-----+-----+--------+
| REN | RA0 | RA1 | Output |
+-----+-----+-----+--------+
|  1  |  0  |  0  | F(0)   |
|  1  |  0  |  1  | F(1)   |
|  1  |  1  |  0  | F(2)   |
|  1  |  1  |  1  | F(3)   |
+-----+-----+-----+--------+

To test, run:

(in magic)
extract all
ext2sim alias on
ext2sim

(in terminal)
irsim g180 ring.sim ring.al

(in irsim)
@ ring.irsim
