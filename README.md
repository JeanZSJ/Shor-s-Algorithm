# Shor's Algorithm--Classical and Quantum Implementations

This repository contains two implementations of Shor's algorithm for integer factorization.

The first is a purely classical simulation. It also contains a toy of example of RSA to show why factoring is important for encryption.

The second follows IBM's Qiskit tutorial (https://quantum.cloud.ibm.com/docs/en/tutorials/shors-algorithm) but also includes a fix for a bug I discovered in their Quantum Shannon Decomposition step. 
Isometry-based approach could be applied to work around this problem.

## The Bug Fix is as follows:

### Context: 

Shor's algorithm turns factoring into an order-finding problem. More detailed information can be found here (https://en.wikipedia.org/wiki/Shor%27s_algorithm) and in the IBM tutorial.

Modular Exponentiation is an important part in the quantum subroutine of Shor's algorithm. 

It is what allows us to encode the information of the order into the circuit, so that we can later extract the order using inverse quantum fourier transform and continued fractions.

In Qiskit's tutorial, they proposed a way to synthesize a modular exponentiation gate:

```python
def mod_mult_gate(b, N):
    """
    Modular multiplication gate from permutation matrix.
    """
    if gcd(b, N) > 1:
        print(f"Error: gcd({b},{N}) > 1")
    else:
        n = floor(log(N - 1, 2)) + 1
        U = np.full((2**n, 2**n), 0)
        for x in range(N):
            U[b * x % N][x] = 1
        for x in range(N, 2**n):
            U[x][x] = 1
        G = UnitaryGate(U)
        G.name = f"M_{b}"
        return G
```
However, if we run the code block offered in the tutorial, we would run into a problem

```python
# Let's build M2 using the permutation matrix definition
M2_other = mod_mult_gate(2, 15)

# Add it to a circuit
circ = QuantumCircuit(4)

circ.compose(M2_other, inplace=True)

circ = circ.decompose()
circ.draw(output="mpl", fold=-1)
```

Here, running circ.decompose() will raise an error:

```python
PanicException: called `Option::unwrap()` on a `None` value
```
Interestingly, only the specific combination of (2,15) will raise an error. For other combinations, this code block seems to run just fine. 

It turns out that this is a problem with Qiskit's Quantum Shannon Decomposition (QSD). 
QSD is an algorithm for breaking down an arbitrary n-bit unitary operator into basic quantum gates. This process of turning a unitary operator into a series of gates is called decomposition.

It turns out that not all permutation matrices are equally easy to decompose, and the specific combination of (2,15) happens to hit a numerically unstable edge case in QSD (more explanation on that later).

We can use Isometry to work around this problem (more explanation on that later as well).

```python
M2_other = mod_mult_gate(2, 15)

iso_circ = Isometry(M2_other.to_matrix(), 0, 0).definition
transpiled_circ = transpile(iso_circ, basis_gates = ["u", "cx"], optimization_level = 3)
```
(the code fix is provided by Dr. McKinney)

Note: Many thanks to Dr. McKinney for his guidance & help! His explanations made it possible for me to understand what was going on.
