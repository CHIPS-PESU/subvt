Toolflow
========

Choosing Transistor Variant
---------------------------

A Process Design Kit (PDK) includes a variety of transistors. These may have different supply and threshold voltages. These are usually `LVT`, `SVT`, `HVT`.
Some PDKs offer `ULVT` and `UHVT` versions too. Higher the threshold voltage, lesser is the leakage current in the transistor. If a system is not timing
critical, `HVT` transistos are suitable, whereas when speed is the primary concern, `LVT` cells are preferable. Multi-Vt algorithm is used during optimization
to reduce leakage power of the design while meeting the timing/delay requirement at same time. The HVT cells are used on less timing critical path to reduce
leakage power whereas `LVT` cells are used for more timing critical paths. This flow also takes care of Noise. This method not only reduces leakage power during
the standby mode, but also during active mode operation of the device.

Schematic Genaration
--------------------

We follow the CMOS logic topology. All multiplications in the boolean expression represent transistors connected in series, and all additions represent parellel
connections. We use the `Cadence Virtuoso Schematic Editor` to instantiate transistors from a said PDK. Post connecting all nets in the schematic, we create pins
to interface the input, output, power and ground signals. These are packaged as a symbol to be used as a design under test (DUT). We use a testbench schematic
to drive inputs to the combinational/sequential standard cell. All cells assume a minimum fan-out of `20 fF` for their minimum drive strength variant.
We import this schematic to the `Cadence Analog Design Environment` to define transient and DC analyses. These analyses are then exported as a `spectre` netlist.
These netlists contain all aspect ratios and the supply voltage as free design parameters, hence enabling some level of automation.

Computing the Optimal Supply voltage
----------------------------------

We design the schematic for a basic CMOS inverter, sized for symmetric operation, and vary the supply voltage in order to maintain a static `logic high` for
computing leakage. Static energy `Estat` is measured as `Vleak/Rout` while performing a DC analysis, and Dynamic energy `Edyn` is measured as the integral of
total current at the output node for one complete transition (l-h-l). It can be observed that `Edyn` increases, and `Estat` with an increase in the supply voltage.
We compute the total energy `Etot` as `Estat + Edyn`. We plot Static, Dynamic, and Total energy against supply voltage to identify the supply voltage at which the minimum energy point is achieved. This value of
supply voltage is considered to be the optimal supply voltage for energy efficient operation.

Transistor Sizing
-----------------

We utilise the netlists exported in `Schematic Genaration` to perform a multi-variable multi-objective optimisation. Transistor sizing is treated as a 
non-linear curve fitting problem. Genetic algorithms would be the most common approach, but we use the `Gauss-Newton method using a trust region approach`.
Given a set of `m` empirical pairs `(xi, yi)` of independant and dependant variables, find the parameters `B` of a model cirve `f(x, B)` such that the sum
of the squares of the deviations is minimized. We use `Cadence SpectreMDL` to write scripts that can interface the `Spectre Circuit Simulator`.

We define 2 `measurement alias` within the MDL control file, and export the following variables for computation:

| fall = falltime(sig=V(OUT), initval=vdd, inittype='y, finalval=0.0, finaltype='y, theta1=90, theta2=10)
| rise = risetime(sig=V(OUT), initval=0.0, inittype='y, finalval=vdd, finaltype='y, theta1=10, theta2=90)
| tphl = cross(sig=V(OUT), dir='fall, n=1, thresh=vdd/2) - cross(sig=V(IN), dir='rise, n=1, thresh=vdd/2)
| tplh = cross(sig=V(OUT), dir='rise, n=1, thresh=vdd/2) - cross(sig=V(IN), dir='fall, n=1, thresh=vdd/2)
| delay = max(rise, fall) + max(tphl, tplh)
| energy = integ(I$INAME:pwr, from=starttime, to=stoptime)

All these parameters are normalised to lie between 0 and 1, and compute the tradeoff between energy and delay as described in [3].

We use the `mvarsearch` construct within `SpectreMDL` to perform the optimization.

```
mvarsearch
 {
    
option
 {
        options_statements
    }
    
parameter
 {
        parameter_statements
    }
    
exec
 {
        exec_statement -- run statement to compute goal values.
    }
    
zero
 {
        zero_statements
    }
}
```

The option_statements include:

```
[ method = method ]
[ accuracy = conv_tol ]
[ deltax = diff_tol ]
[ maxiter = maxiter ]
[ restoreParam = restoreParam ]
```

The parameter_statements include:

```
{param_name, init_val, lower_val, upper_val}
```

In the following example design parameters para_pw and para_nw are varied by the optimization algorithm starting at an initial value of 1.2 microns
with a maximum value of 10 microns and a lower limit of 0.1 microns. At each iteration, the measurement alias trans is run after the design parameter
value is set. The zero values tmp1 and tmp2 are then computed using the results from the measurement alias. This iteration continues until one of the
following happens:

| -tmp1 and tmp2 satisfy the conv_tool criteria determined by the following equation: (tmp1*tmp1 + tmp2*tmp2) < 1.0e-03
| the maxiter parameter value is exceeded

```
alias measurement trans {
run tran( stop=1u, autostop='yes )
    export real rise=risetime(sig=V(d), initval=0, inittype='y, finalval=3.0, 
       finaltype='y, theta1=10, theta2=90) // measured from 10% to 90% 
    export real fall=falltime(sig=V(d), initval=3.0, inittype='y, finalval=0.0,
       finaltype='y, theta1=90, theta2=10) // measured from 10% to 90% 
}
mvarsearch {
    option {
       accuracy = 1e-3     // convergence tolerance of trans->rise
       deltax = 1e-3       // numerical difference % of design variables
       maxiter = 100       // limit to 100 iterations
    }
    parameter {
       {para_pw, 1.2u, 0.1u, 10u}
       {para_nw, 1.2u, 0.1u, 10u}
    }
    exec {
       run trans
    }
    zero {
       tmp1 = trans->rise - 3ns
       tmp2 = trans->fall - 3ns 
    }
}
```

Layout Generation
-----------------



Netlist Extraction
------------------

Cell Charecterization
---------------------

Synthesis
---------

Post-Synthesis Simulation
-------------------------

Place and Route
---------------

.. autosummary::
   :toctree: generated

