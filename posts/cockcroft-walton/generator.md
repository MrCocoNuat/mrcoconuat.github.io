# Nice Cockcroft-Walton

Ok, jokes aside - the [Cockcroft-Walton Multiplier](https://en.wikipedia.org/wiki/Cockcroft%E2%80%93Walton_generator) is a design that generates DC from AC, charging up `N` stages of capacitors sequentially while keeping them in series so that the final output voltage achieved is as high as `N*Vpp`. To understand this, think of those old toys where figurines would slide down a track until they hit a staircase, and like magic, a constantly oscillating motion could move them all the way to the top for them to slide down again.

![the toy in question](assets/toy.jpg)

In this circuit, the input AC is the oscillating motion and electron front is the penguins - as each capacitor is charged by one half-wave (`C1` first), the next half-wave pushes the particular stack ending with that capacitor to a high enough voltage to exceed that of the other stack, and so the next capacitor in sequence can charge (here, `C2`). After many iterations (remember that `C1` will lose voltage charging `C2`, so the overall process isn't quite that fast), each of the capacitors in the output stack (`C2`, `C4`) holds `Vpp`, and the output is generated!

![schematic](assets/cw-schematic.png)

With a output frequency around 20kHz, the ZVS driver is almost ideally suited for supplying this generator. You can send power through a HV transformer first, so that `Vpp = 10-20kV` and use corresponding high voltage diodes and capacitors, or pipe the `60Vpp` directly into a low voltage but higher power version of the circuit. That is what I did.

