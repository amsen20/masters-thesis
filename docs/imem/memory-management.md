- [x] Define the memory definition and the env, references and linear and nonlinear values
- [x] Then proceed to define well formed-ness
- [x] Then define a minimal two element list as an example:

  - [x] First show how hard it is to iterate it, map it.
  - [x] Second show how it is impossible to have a immutable view to the second element.

- [x] Mention that in a well formed linear memory, cascade deleting a variable is safe.
- [x] Then mention that `@HideLinearity` allows non well form memories to form in runtime.
- [x] Then start defining the imem memory overview:
- [x] Define boxes, value holders, immutable reference and mutable references.
- [ ] Define what a well-formed memory overview in this definition looks like.
- [ ] Then again, define a minimal two element list as an example.

  - [ ] Easy to iterate.
  - [ ] Easy to have reference to the second element.

- [ ] It's safe to delete after lifetime is passed (in lifetime order).

- [ ] Prove that a program that always has a well formed memory, never violates the stacked borrows model

---

<!-- TODO: Check all the definitions -->

# Memory Management in imem

This section defines the meaning of memory management in imem, the safety guarantees that imem statically ensures, and the reasons why imem is more expressive than pure linear data structures.

## Pure Linear Memory

A pure linear program that follows all linearity rules described in the [original linearity paper](https://cs.ioc.ee/ewscs/2010/mycroft/linear-2up.pdf), or in scinear terms, that is, a program without `@HideLinearity`, structures memory as a tree-like data structure.

### Linear Memory Definition

The following presents a formal definition of a linear program’s memory at a given point during execution.
The environment splits into linear and nonlinear environments.

$$ \rho = \rho_L \cup \rho_{NL} $$

The linear environment is a mapping from linear variables to linear locations.

$$ \rho_L: \text{Var}_L \rightarrow \text{Loc}_L $$

The nonlinear environment is a mapping from nonlinear variables to nonlinear locations or integer values.

$$ \rho_{NL}: \text{Var}_{NL} \rightarrow \{ \text{Loc}_{NL} \cup \mathbb{Z} \} $$

The memory is also divided into linear and nonlinear.

$$ \sigma_L: \text{L}_L \rightarrow \text{Val}_L $$

$$ \sigma_{NL}: \text{Loc}_{NL} \rightarrow \text{Val}_{NL} $$

Linear values consist of linear references, nonlinear references, and integer values.

$$
\text{Val}_L = \{ \langle v_1, \dots, v_k \rangle \vert v_i \in \text{Loc}_{L} \cup \text{Loc}_{NL} \cup \mathbb{Z} \}
$$

Nonlinear values consist of nonlinear references and integer values.

$$
\text{Val}_{NL} = \{ \langle v_1, \dots, v_k \rangle \vert v_i \in \text{Loc}_{NL} \cup \mathbb{Z} \}
$$

### Well-Formedness

The memory is well-formed if the linear part consists of disjoint trees, each rooted at a linear environment variable.

Formally, every linear value has exactly one other linear value or linear variable whose location serves as a reference, and linear variables can reach all linear values.

***Reference Uniqueness:***
Every linear location $l \in \text{dom}(\sigma_L)$ has to be mentioned exactly by its parent, which can be a linear value or environment variables.

Parents of \(l\) are defined as:

$$
P(l) = \{ x \in \text{dom}(\rho_L) \mid \rho_L(x) = l \} \cup \{ k \in \text{dom}(\sigma_L) \mid l \in \text{Refs}(\sigma_L(k)) \}
$$

Where $\text{Refs}(v)$ is the set of references linear value $v = \langle v_1, \dots, v_k \rangle$ mentions:

$$
\text{Refs}(v) = \{ v_i \vert v_i \in \text{Loc}_{L} \}
$$

In a well-formed memory $\text{WF}(\rho_L, \sigma_L)$ every location has one parent.

$$
\text{WF}(\rho_L, \sigma_L) \implies \forall l \in \text{dom}(\sigma_L), |P(l)| = 1
$$

$par$ is the function that maps linear locations to their parent, meaning:

$$
\text{par}(l) \in P(l)
$$

***Reachability of Every Linear Value:***
The variables in the environment must reach all linear locations in memory.

In other words, for every linear location \(l\), the following condition must hold:

$$
\text{WF}(\rho_L, \sigma_L) \implies
\forall l \in \text{dom}(\sigma_L), \exists n \ge 0 \text{ s.t. } par^{n}(l) \in \text{range}(\rho_L)
$$

TODO: A DIAGRAM OF HOW LINEAR MEMORY LOOKS LIKE

### An Example

The following is a two-item list expressed with linear values:

$$
\rho_L = \{ \texttt{list} \mapsto l_1 \}
$$

$$
\sigma_L = \{
  l_1 \mapsto \langle 42, l_2 \rangle,
  \quad
  l_2 \mapsto \langle 99, 0 \rangle
\}
$$

In Scala, using Scinear, the following code constructs the memory described above:

```Scala
TODO: SCINEAR CODE THAT CONSTRUCTS THE LIST
```

To allow the program to access the second element (for example, for reading) without ever creating a non-well-formed memory state;
It must deconstruct the first element of the linear list, store that value in a separate variable, and then retain a linear variable that refers to the second element.

```Scala
TODO: SCINEAR CODE THAT ACCESSES THE SECOND ELEMENT
```

After accessing the second element, the memory would become the following:

$$
\rho'_L = \{ \texttt{tail} \mapsto l_2 \}
$$

$$
\rho'_{NL} = \{ \texttt{head} \mapsto 42 \}
$$

To change the first element of the list, for example, by multiplying it by two, the program must reconstruct the list after deconstructing the first element and replacing it with the new value.

```Scala
TODO: SCINEAR CODE THAT RECONSTRUCTS THE LIST
```

This results in:

$$
\rho''_L = \{ \texttt{list'} \mapsto l_3 \}
$$

$$
\sigma''_L = \{
  l_3 \mapsto \langle 84, l_2 \rangle,
  \quad
  l_2 \mapsto \langle 99, 0 \rangle
\}
$$

Overall, in pure linear data structures, accessing a node in the tree requires deconstructing the entire path from the root to that node.
Moreover, when updating a nonlinear field of a linear value, the value must first be accessed, and then the entire path from the root must be reconstructed.

### Memory Management

At any point in a well-formed memory, deleting a linear variable and, accordingly, freeing all linear locations reachable from that variable is safe.
The notion of safety is defined as follows:

- No double-free occurs, because each linear location appears exactly once in the memory, either in another linear value or in a variable.
- No dangling references remain after deletion, because each linear location is only referenced by one other linear value or variable.
  As a result, freeing a location propagates through its parent structure, which is also freed.

Based on linearity rules, each linear variable is mentioned exactly once in the program.
Therefore, each variable is either used in another expression or safely freed, resulting in no memory leaks and enabling static memory management for linear data structures.
<!-- TODO: Link to a paper on memory management of linear types -->

### Effect of `@HideLinearity`

In Scinear, `@HideLinearity` allows a program to construct memories whose linear part is not well formed.
For example, the following code creates a cyclic data structure between two linear values.

```Scala
TODO: AN EXAMPLE OF `@HiderLinearity` CREATING CYCLIC DATA STRUCTURES
```

imem uses `@HideLinearity` to provide a more expressive approach while still performing safe memory management statically.

## imem Memory

imem extends linear memory with new kinds of references that make working with a well-formed memory more expressive than using pure linear values that follow linearity rules.
To ensure the safety of well-formed memories and to enable static memory management, imem introduces lifetimes.

### imem Memory Definition

imem's new reference kinds are a box, immutable reference, mutable reference and a value holder.
They all hold a linear or nonlinear location.

To make sense of these new references, a box is a general reference that a immutable or mutable reference can be borrowed from it.
An immutable reference, provides the program with a read access to the location its referring to.
A mutable reference is similar to an immutable reference, only it provides read-write access.
A value holder reference, makes sure that when the program accesses the location its referring to, some other references does not exists.

Boxes, immutable reference, and mutable reference have a set of lifetimes assigned to them.
Each lifetime will expire sometime during the program execution.
And with the lifetime, all the references mentioning this lifetime in its set will expire too.
The following will give more explanation on what expiration actually means.

A value holder has a lifetime assigned to it, meaning that the program cannot access to the location it is holding unless the lifetime is expired.

At each point of the execution, lifetimes are divided into expired and available lifetimes.

boxes, mutable references, and value holders are linear values, and immutable references are nonlinear.


A lifetime is an entity that becomes available at some point during execution and expires later.  
The set of all lifetimes is \(\mathcal{L}\).

imem extends the memory state by recording which lifetimes are currently available:

$$
(\rho, \sigma, \Lambda)
$$

Where $\Lambda \subseteq \mathcal{L}$ is the set of available lifetimes.

imem extends the set of linear values by introducing three kinds of references:

$$
\begin{aligned}
  \text{Val}_L =
  &\ \{ \langle v_1, \dots, v_k \rangle \mid v_i \in \text{Loc}_{L} \cup \text{Loc}_{NL} \cup \mathbb{Z} \} \\
  &\ \cup \{ \text{box}(l, \tau) \mid l \in \text{Loc}_{L} \cup \text{Loc}_{NL},\ \tau \subseteq \mathcal{L} \} \\
  &\ \cup \{ \text{mref}(l, \tau) \mid l \in \text{Loc}_{L} \cup \text{Loc}_{NL},\ \tau \subseteq \mathcal{L} \} \\
  &\ \cup \{ \text{hold}(l, \alpha) \mid l \in \text{Loc}_{L} \cup \text{Loc}_{NL},\ \alpha \in \mathcal{L} \}
\end{aligned}
$$

These new linear references are:

- A \(\text{box}(l, \tau)\) is a general reference that the program can borrow immutable and mutable references from.
- \(\text{mref}(l, \tau)\) is a mutable reference, which provides read-write access to location \(l\).
- \(\text{hold}(l, \alpha)\) is a value holder, which prevents access to location \(l\) until lifetime \(\alpha\) expires.

In addition, imem extends nonlinear values by adding immutable references:

$$
\begin{aligned}
  \text{Val}_{NL} =
  &\ \{ \langle v_1, \dots, v_k \rangle \mid v_i \in \text{Loc}_{NL} \cup \mathbb{Z} \} \\
  &\ \cup \{ \text{iref}(l, \tau) \mid l \in \text{Loc}_{L} \cup \text{Loc}_{NL},\ \tau \subseteq \mathcal{L} \}
\end{aligned}
$$

An immutable reference \(\text{iref}(l, \tau)\) is similar to a mutable reference but provides read-only access to the location and is a nonlinear value.

imem associates a set of lifetimes \(\tau \subseteq \mathcal{L}\) with the references \(\text{box}(l, \tau)\), \(\text{mref}(l, \tau)\), and \(\text{iref}(l, \tau)\).  
If any lifetime in \(\tau\) expires, the reference becomes unavailable.
Formally, a value \(v \in \{ \text{box}(l, \tau), \text{mref}(l, \tau), \text{iref}(l, \tau) \}\) is available iff:

$$
\text{Available}(v, \Lambda) \iff \tau \subseteq \Lambda
$$

Availability of a reference is used later in the definition of well-formedness.

A resource is a location that has an imem \(\text{box}\) reference pointing to it.

$$
l \in \text{Res} \iff \exists \tau \subseteq \mathcal{L} \text{ s.t. } \text{box}(l, \tau) \in \text{Val}_L
$$

The provided is the definition and well-formedness definition for the linear memory.

### Well-formedness

#### Additional Definitions

***Dereferencing:***
The function $\text{loc}(v)$ returns the location pointed to by a reference value.
Meaning for any value $v \in \{ \text{box}(l, \tau), \text{mref}(l, \tau), \text{iref}(l, \tau), \text{hold}(l, \alpha) \}$:

$$
\text{loc}(v) = l
$$

***Reachability:***
A value $v'$ is reachable from value $v$, denoted $v \rightsquigarrow v'$, if there exists a sequence of the following operations that leads from $v$ to $v'$:
- TODO: Simply define each operation (dereferencing a box or accessing a field of a linear value)

***Available Linear Environment:***
The set of available linear variables, $\text{Var}_{Avail}$, consists of all variables in $\rho_L$ that the first reference that they reach is an available reference.

***Direct Boxes:***
The set of direct boxes of a location $l$ is called $D(l)$, which consists of all boxes in the memory that point to $l$.

$$
D(l) = \{ b \in \text{Val}_L \mid \exists \tau, b = \text{box}(l, \tau) \}
$$

***Paths:***
A path $\pi$ is a sequence of values starting from a variable in the environment $\rho$ and ending at a specific value using the reachability operations.

#### Properties

A memory state $(\rho, \sigma, \Lambda)$ is well-formed if it satisfies the following.

***Direct Box Uniqueness:***
Every resource reachable from an available variable must have exactly one reachable direct box reference pointing to it.
Let $\text{B}_{Avail}$ be the set of all boxes reachable from $\text{Var}_{Avail}$.

$$
\forall l \in \text{Res}, \quad | D(l) \cap \text{B}_{Avail} | = 1
$$

***No cyclic box***:
A box's resource must not reach itself through dereferencing:

$$
\forall b \in \text{B}_{Avail}, \quad \neg (\sigma(\text{loc}(b)) \rightsquigarrow b)
$$

***No dangling boxes:***
If a box $b_1 = \text{box}(l_1, \tau_1)$ reaches another box $b_2 = \text{box}(l_2, \tau_2)$, then if $b_2$ becomes unavailable, $b1$ gets unavailable too.

$$
b_1 \rightsquigarrow b_2 \implies \tau_2 \subseteq \tau_1
$$

This ensures that all boxes reachable from the available linear variables are available.

***Borrowing Validity:***
If a mutable or immutable reference exists for a location, there must be a corresponding box for that location with an assigned lifetime set that is a subset of the reference's.
For any reference $r \in \{ \text{mref}(l, \tau_r), \text{iref}(l, \tau_r) \}$, there exists a box $b = \text{box}(l, \tau_b)$ such that $\tau_b \subset \tau_r$.

This ensures that a box, always outlives the references that are borrowed from it.

***Box reaching Reference:***
If a box $b$ reaches a mutable or immutable reference $r$, then every path from the environment to $b$ must contain a value holder $\text{hold}(l, \alpha)$ such that $\alpha \in \tau_r$.

***Mutable Reference reaching Immutable Reference:***
If a mutable reference $m$ reaches an immutable reference $ir$, then every path from the environment to $m$ must contain a value holder $\text{hold}(l, \alpha)$ such that $\alpha \in \tau_{ir}$.

***Mutable Reference reaching mutable Reference:***
If a mutable reference $m_1$ reaches another mutable reference $m_2$:

- If $\text{loc}(m_1) \neq \text{loc}(m_2)$.
  Every path from the environment to $m_1$ must contain a value holder $\text{hold}(l, \alpha)$ such that $\alpha \in \tau_{m2}$.
- If $\text{loc}(m_1) = \text{loc}(m_2)$.
  Either every path from the environment to $m_1$ contains a holder for $m_2$ (as above), or the path to $m_2$ contains a holder for $m_1$.

***Immutable Reference not reaching Mutable Reference:***
No available immutable reference should reach available mutable reference.

<!-- TODO: CHECKOUT THE EDITED VERSION -->