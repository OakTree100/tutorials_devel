Tutorial 2.5: The RFDC DAC Interface
====================================

Introduction
------------
In the previous tutorial we introduced the RFDC Yellow block, with configurations
for the dual- and quad-tile RFSoCs with ADCs. It it worth providing a brief
introduction to the DAC interface. This tutorial assumes you have completed through
the RFDC tutorial :doc:`RFDC Interface <./tut_rfdc>`

The Example Design
--------------------
In this example we will configure the RFDC for a dual-tile RFSoC4x2 board.

This design will:
  * Set sample rates
  * Use the internal PLLs to generate the sample clock
  * Output a sinusoidal signal
  * Write and read data from a bram

The final design will look like this for the RFSoC 4x2:

.. image:: tut_dac_simple_layout.png


Section 1: Assembling & Configuring the blocks
----------------------------------------------

You'll need all these blocks
 * System Generator
 * RFSoC 4x2 block
 * RFDC
 * An "enable" software register
 * bram
 * munge
 * counter
 * Xilinx constants

Add your ``System Generator`` and ``RFSoC 4x2`` blocks
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code:: bash

  # RFSoC4x2
  User IP Clock Rate: 245.76, RFPLL PL Clock Rate: 491.52

Add your ``rfdc``
^^^^^^^^^^^^^^^^^
Double click on it, and disable all
available ADCs. Enable the first and second DAC tiles (228, 229), and only
enable the DAC 0 in either. Your ``Required AXI4-Stream Clock (MHz)`` should be 245.76.
Configure the DAC tiles as follows:

.. code:: bash

  # Tile Config
  Sampling Rate   (MHz) - 1966.08
  Clock Out       (MHz) - 122.88
  Reference Clock (MHz) - 491.52
  Enable Tile PLLs      - True
  Output Power          - 20

  # DAC 0 Config
  Analog Output Data    - Real 
  Interpolation Mode    - 1x 
  Samples Per AXI Cycle - 8 
  Mixer Type            - Coarse
  Mixer Mode            - Real -> Real
  Frequency             - 0
  Nyquist Zone          - Zone 1
  Decoder Mode          - SNR Optimized

  CHECK:
  Does your Required AXI4-Stream clock say 245.76?

.. image:: dac_config.png

Add your ``bram``
^^^^^^^^^^^^^^^^^
The bram is where we'll save the data to drive the dac.
Inside of our FPGA PL (Programmable Logic) there are bram memory blocks spread 
throughout the fabric. Each of these memory banks has a specific size,
if we request more capacity than a single bram can provide, we may encounter
timing violations that can be addressed through delay blocks.
We choose a ``Data Width`` of 128 because the ``rfdc`` takes in 8 16-bit samples
every clock cycle.

We'll drive this block's ports as follows:
 * ``addr`` - A counter to loop through our samples,
 * ``we`` - A boolean 0 to prevent this bram from being written to by any PL blocks
 * ``data_in`` - A 128 bit 0 because we need an appropriately sized Xilinx block driving this port

.. code:: bash

  Output Data Type          - Unsigned
  Address width             - 13
  Data Width                - 128
  Register Primitive Output - No
  Register Core Output      - No
  Optimization              - Minimum_Area
  Data Binary Point         - 0
  Initial Values (sim only) - Not important
  Sample rate               - 1

.. image:: tut_dac_bram_config.png


Add your ``munge``
^^^^^^^^^^^^^^^^^^
On the output of our ``bram`` we're using a munge to reorder data for compatibility between the ``rfdc`` 
and other casper blocks. We'll study this block more in depth for Tutorial 3. This block takes a bus of 
some width (128 bits in our case), and separates it into pieces (some number of divisions, with some size for each)
(8 16-bit samples for us), and then reorders them (we're just reversing things for DAC compatibility here).
In hardware, this routes wires and costs nothing.

``din`` should connect to the ``bram`` ``data_out``. 

``dout`` should connect to both ``s00_axis_tdata`` and ``s10_axis_tdata`` on the ``rfdc``

.. code:: bash

  Number of divisions       - 8
  Division size (bits)      - 16*ones(1,8)
  Division packing order    - [7 6 5 4 3 2 1 0]
  Output arithmetic type    - Unsigned
  Output binary point       - 0

.. image:: tut_dac_munge_config.png


Add your ``Counter``
^^^^^^^^^^^^^^^^^^^^
Connect the output of this block to the ``bram``'s ``addr`` port.

This block will loop through all of the addresses in our bram, 
playing our signal on repeat. If you add separate control
logic, you can set a specific counter value to restart playback,
for now we don't need that level of control to play a sine wave.

.. code:: bash

  Counter type              - Free running
  Count direction           - Up
  Initaial value            - 0
  Step                      - 1
  Output type               - Unsigned
  Number of bits            - 13
  Binary point              - 0
  Provide load port         - No
  Provide sync reset port   - Yes
  Provide enable port       - Yes
  Sample period source      - Explicit
  Sample rate               - 1

.. image:: tut_dac_counter_config.png


Add some ``Constant`` blocks
^^^^^^^^^^^^^^^^^^^^^^^^^^^^
We need 3 Xilinx Constant blocks.

.. code:: bash

  bram constants:
    we
      Constant Value    - 0
      Output Type       - Boolean
      Sampled Constant  - Yes
      Sample period     - 1

    data_in
      Constant Value    - 0
      Output Type       - Fixed Point
      Number of Bits    - 128
      Binary point      - 0
      Sampled Constant  - Yes
      Sample period     - 1

  counter constant:
    rst
      Constant Value    - 0
      Output Type       - Boolean
      Sampled Constant  - Yes
      Sample period     - 1      

Add your ``Enable``
^^^^^^^^^^^^^^^^^^^^
Connect the input of this block to a Simulink constant
Connect the output of this block to the ``Counter``'s ``en`` port.
This block enables the playing of our sine wave and looks really cool
while doing it.

.. code:: bash

  I/O direction             - From processor
  I/O delay                 - 0
  Initial Value             - dec2hex(0)
  Sample period             - 1
  Bitfield names [msb..lsb] - reg
  Bitfield widths           - 1
  Bitfield binary pts       - 0
  Bitfield types            - 2 (bool)

.. image:: tut_dac_enable_config.png


Optional: Add a waveform length ``wf_len`` register
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
To keep track of how many addresses our counter iterates over, we can 
add register wf_len1. This block is primarily useful for debugging. We'll
connect its output to a scope, so we can run a simulation in simulink.

.. code:: bash

  I/O direction             - To processor
  I/O delay                 - 0
  Initial Value             - dec2hex(0)
  Sample period             - 1
  Bitfield names [msb..lsb] - reg
  Bitfield widths           - Equal to counter width
  Bitfield binary pts       - 0
  Bitfield types            - 0 (ufix)

Once we've added this register, we'll be able to check it's value from ipython
For now, we can press run, and watch our counter iterate over the data.
In our scope, if we right click, we can find ``Signals & Ports``, and set the
Number of Input Ports to 2. 
We can connect the either input to the bram or munge and see the data change. 



Section 2: Generating your signal
---------------------------------

For this tutorial we will generate a sine wave in software. You can use 
the provided code, we would recommend that you add it to a file, which
you can run in ipython with ``run sine.py``

``sine.py``

.. code:: python

  import numpy as np
  import struct
  
  # bram parameters - need to match our yellow block's values
  block_size = 128  # <bram data_width>
  blocks = 2**13    # 2**<bram address_width>
  bits_per_val = 16 # <rfdc input data size> 16 bits for rfsoc4x2
  # We need to make sure our output data size matches the bram's
  # capacity, so we don't fail on writes
  num_vals = int(block_size / bits_per_val * blocks)
  
  # sine wave parameters
  fs = 1966.08e6      # Sampling frequency
  fc = 393.216e6      # Carrier frequency
  dt = 1/fs           # Time length between samples
  tau = dt * num_vals # Time length of bram 
  
  # Useful info if running as a script
  print(f"fs = {fs / 1e6} MHz")
  print(f"fc = {fc / 1e6} MHz")
  
  # Setup our array
  t = np.arange(0,tau,dt)
  
  # Generate our sine wave
  # frequency fc
  # range 0, 1
  x = 0.5*(1+np.cos(2*np.pi* fc *t))
  # scale our function to use the whole DAC range
  maxVal = 2**14-1
  x *= maxVal
  # set each value to a 16 bit integer, for DAC compatibility
  x = np.round(np.short(x))
  # Shift right, DAC is 14 bits
  x <<= 2

  # Save our array x as a set of bytes  
  buf = bytes()
  for i in x:
    buf += struct.pack('>h',i)

  # # Code used to create plots shown below code block 
  # # python3 sine.py
  # # ^ run from the terminal
  # import matplotlib.pyplot as plt
  # plt.plot(np.ushort(x[:100]))
  # plt.title(f"fs = {fs / 1e6} MHz; fc = {fc / 1e6} MHz")
  # plt.show()

  # We're done!, we can now write buf to our
  # bram. To make sure it exists, enter len(buf)
  # in your ipython terminal

  # If needed we can save it as a file 
  # for later use or transferability  
  f = open("sine.txt", "bw")
  f.write(buf)

.. image:: sine_py_plot.png

.. image:: sine_py_plot_2.png

These images plot or sine wave data points that
we wrote to our bram. In some cases, the wave will
not be continuous between the last element of the bram
and the first element, causing some noise. Additional 
logic can reset our counter on a sample which will provide
a smooth transition, but for this tutorial we've elected to
keep things as simple as possible.

Note that these sine wave data points are simply samples passed
into our bram. In order to convert these to a voltage, we would
need to consider the output power of our dac


Section 3: Sending your signal out
----------------------------------

0) Start an ipython session
1) Connect to and program your board normally
2) Program your DAC clocks as you did for the ADCs in tutorial 2
3) Generate your sine wave as shown above. This has to be done within your ipython 
   session or in the same script to that your values are available in buf
4) Write your sine wave to your bram, and a 1 to your enable register

.. code:: python

  In [9]: rfsoc.listdev()
  Out[9]: 
  ['rfdc',
  'sys',
  'sys_board_id',
  'sys_clkcounter',
  'sys_rev',
  'sys_rev_rcs',
  'sys_scratchpad',
  'wf_bram_0',
  'wf_en']

  In [10]: rfsoc.write('wf_bram_0', buf)

  In [11]: rfsoc.write_int('wf_en', 1)

5) Connect a network analyzer or oscilloscope to your output.

Your signal should be output on the DAC labed DAC_B. Why? Who knows
Your signal in an network analyzer should look something like this:

.. image:: spectrum_output.jpg

Be aware, that if ``wf_en`` is disabled, you may still have signals
at 491.52 MHz and 245.76 MHz. Your DAC Reference Clock and 
your User IP Clock Rate. We use wf_en to run our counter block. If 
we stop counting, we won't stop playing data, we'll just loop the 
same 8 samples forever. If we set those samples to 0s, we lose those
signals.


Errors
------
If you get an error like the following, make sure that your constant block driving
data_in on your bram has ``Number of Bits == 128``

.. code:: bash

  Width of slice (number of bits) is set ot a value of 32, but the value 
  must be less than or equal to 16. The input signal bit-width, 16,
  determines the upper bound for the width of the slice.
  Error occurred during "Rate and Type Error Checking"

  Reported by:
    'design/shared_bram/munge_in/split/slice3'


If you get an error like the following, make sure your bram address width in your
simulink model matches the bram address width in your ``sine.py`` script (the script
in Section 2)

.. code:: python

  UnicodeDecodeError                        Traceback (most recent call last)
  Cell In[7], line 1
  ----> 1 rfsoc.write('shared_bram', buf)

  ...
  ...

  File ~/.conda/envs/enmotion/lib/python3.8/site-packages/katcp/core.py:384, in Message.__str__(self)
      382     return byte_str
      383 else:
  --> 384     return byte_str.decode('utf-8')

  UnicodeDecodeError: 'utf-8' codec can't decode byte 0x88 in position 21: invalid start byte

