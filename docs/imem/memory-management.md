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

$$ \sigma_L: \text{Loc}_L \rightarrow \text{Val}_L $$

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

$Par$ is the function that maps linear locations to their parent, meaning:

$$
\text{Par}(l) \in P(l)
$$

***Reachability of Every Linear Value:***
The variables in the environment must reach all linear locations in memory.

In other words, for every linear location \(l\), the following condition must hold:

$$
\text{WF}(\rho_L, \sigma_L) \implies
\forall l \in \text{dom}(\sigma_L), \exists n \ge 0 \text{ s.t. } Par^{n}(l) \in \text{range}(\rho_L)
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

To allow the program to access the second element (for example, for reading) without ever creating a non-well-formed memory state,
it must deconstruct the first element of the linear list, store that value in a separate variable, and then retain a linear variable that refers to the second element.

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

---------------------------------------------------------------------------------------------------------------
THE INTERMEDIATE STATE BEGIN

## imem memory

imem extends linear memory by introducing new kinds of references.
To explain the imem memory model more clearly, the first part describes these additional references, how a program uses them, the expected well-formedness, and the need for lifetimes.
Then, the section introduces lifetimes and explains their role in tracking the availability of the additional references.

### Memory With imem References

The program state is the same as the linear program state:

$$
(\rho, \sigma)
$$

imem extends linear values by introducing two kinds of linear references:

$$
\begin{aligned}
  \text{Val}_L =
  &\ \{ \langle v_1, \dots, v_k \rangle \mid v_i \in \text{Loc}_{L} \cup \text{Loc}_{NL} \cup \mathbb{Z} \} \\
  &\ \cup \{ \text{box}(l) \mid l \in \text{Loc}_{L} \cup \text{Loc}_{NL} \} \\
  &\ \cup \{ \text{mref}(l) \mid l \in \text{Loc}_{L} \cup \text{Loc}_{NL} \}
\end{aligned}
$$

These linear references are:

- A \(\text{box}(l)\) is a general reference that the program can borrow immutable and mutable references from.
- A \(\text{mref}(l)\) is a mutable reference, which provides read-write access to location \(l\).

In addition, imem extends nonlinear values by adding immutable references:

$$
\begin{aligned}
  \text{Val}_{NL} =
  &\ \{ \langle v_1, \dots, v_k \rangle \mid v_i \in \text{Loc}_{NL} \cup \mathbb{Z} \} \\
  &\ \cup \{ \text{iref}(l) \mid l \in \text{Loc}_{L} \cup \text{Loc}_{NL} \}
\end{aligned}
$$

An immutable reference \(\text{iref}(l)\) s similar to a mutable reference but provides read-only access to the location and is a nonlinear value.

#### Program Required Operations

To make these references practical, the program must be able to create references from one another and use them to access specific values in memory.
The following list of operations models the features required by a program to create and use imem references.
These operations are defined as functions that take a program state and additional arguments, and either validate that the requested operation is valid or return a new state that results from applying the operation to the memory state.

#### Operations on Boxes

- ***Create a New Box***: The operation gets a location \(l\) and creates a box that points to \(l\). 
                          A fresh linear location \(k \in \text{Loc}_L\) points to the new box.

$$
\mathsf{newBox}\bigl((\rho,(\sigma_L,\sigma_{NL})),\ l,\ k\bigr)
=
\bigl(\rho,(\sigma_L[k \mapsto \text{box}(l)],\sigma_{NL})\bigr)
$$

- ***Borrow a Box Immutably***: \(\mathsf{borrowImmutBox}\) gets a box and creates an immutable reference that points to the same location. 
                      A fresh nonlinear location \(k_{ir} \in \text{Loc}_{NL}\) points to the new immutable reference.
                      Formally, If \(\sigma_L(k_b)=\text{box}(l)\) and \(k_{ir} \notin \mathrm{dom}(\sigma_{NL})\):

$$
\mathsf{borrowImmutBox}\bigl((\rho,(\sigma_L,\sigma_{NL})),\ k_b,\ k_{ir}\bigr)
=
\bigl(\rho,(\sigma_L,\sigma_{NL}[k_{ir} \mapsto \text{iref}(l)])\bigr)
$$

- ***Borrow a Box Mutably***: The \(\mathsf{borrowMutBox}\) operation behaves similar to \(\mathsf{borrowImmutBox}\), but it creates a mutable reference that points to the same location as the given box.
                      As a result, a fresh linear location \(k_{mr} \in \text{Loc}_{L}\) points to the new mutable reference.
                      The operation is valid if \(\sigma_L(k_b)=\text{box}(l)\) and \(k_{mr} \notin \mathrm{dom}(\sigma_{L})\):

$$
\mathsf{borrowMutBox}\bigl((\rho,(\sigma_L,\sigma_{NL})),\ k_b,\ k_{mr}\bigr)
=
\bigl(\rho,(\sigma_L[k_{mr} \mapsto \text{mref}(l)],\sigma_{NL})\bigr)
$$

#### Operations on Immutable References

- ***Borrow an Immutable Reference***: This operation re-borrows an immutable reference and creates a new immutable reference that points to the same location as the given immutable reference.
                                       The memory stores the new immutable reference at a fresh nonlinear location \(k_{ir} \in \text{Loc}_{NL}\).
                                       As a formal definition, If \(\sigma_{NL}(k_{ir})=\text{iref}(l)\) and \(k'_{ir} \notin \mathrm{dom}(\sigma_{NL})\), then:

$$
\mathsf{borrowImmut}_{\mathit{imm}}\bigl((\rho,(\sigma_L,\sigma_{NL})),\ k_{ir},\ k'_{ir}\bigr)
=
\bigl(\rho,(\sigma_L,\sigma_{NL}[k'_{ir} \mapsto \text{iref}(l)])\bigr)
$$

- ***Reading an Immutable Reference***: This operation accesses the resource of an immutable reference through that immutable reference.
                                        It does not change the memory state. Instead, it only validates that the program's request to use this reference.
                                        This access is valid as long as the immutable reference is reachable from the program environment.
                                        The definition uses \(\text{Reachable}_{Ref}(\rho,\sigma)\), which the later subsection define.
                                        Briefly, \(\text{Reachable}_{Ref}(\rho,\sigma)\) is the set of all boxes and references that the environment can reach.
                                        Formally, if \(k_{ir} \in \mathrm{dom}(\sigma_{NL})\) and \(\sigma_{NL}(k_{ir})=\text{iref}(l)\), then:

$$
\mathsf{readImmut}\bigl((\rho,(\sigma_L,\sigma_{NL})),\ k_{ir}\bigr)
=
\begin{cases}
(\rho,(\sigma_L,\sigma_{NL}))
& \text{if }\text{iref}(l) \in \text{Reachable}_{Ref}(\rho,\sigma) \\[4pt]
\mathsf{nonvalid}
& \text{otherwise.}
\end{cases}
$$

#### Operations on Mutable References

- ***Borrow a Mutable Reference Mutably***: The operation \(\mathsf{borrowImmut}_{\mathit{mut}}\) gets a mutable reference and creates an immutable reference that points to the same location.
                                            The new immutable reference is stored at a fresh nonlinear location \(k_{ir} \in \text{Loc}_{NL}\).
                                            To define it formally, if \(\sigma_{L}(k_{mr})=\text{mref}(l)\) and \(k_{ir} \notin \mathrm{dom}(\sigma_{NL})\):

$$
\mathsf{borrowImmut}_{\mathit{mut}}\bigl((\rho,(\sigma_L,\sigma_{NL})),\ k_{mr},\ k_{ir}\bigr)
=
\bigl(\rho,(\sigma_L,\sigma_{NL}[k_{ir} \mapsto \text{iref}(l)])\bigr)
$$

- ***Borrow a Mutable Reference Immutably***: This operation is similar to \(\mathsf{borrowImmut}_{\mathit{mut}}\), but it creates a new mutable reference that resides at a fresh location \(k'_{mr} \in \text{Loc}_{L}\).
                                              If \(\sigma_{L}(k_{mr})=\text{mref}(l)\) and \(k'_{mr} \notin \mathrm{dom}(\sigma_{L})\) then the operation is:

$$
\mathsf{borrowMut}\bigl((\rho,(\sigma_L,\sigma_{NL})),\ k_{mr},\ k'_{mr}\bigr)
=
\bigl(\rho,(\sigma_L[k'_{mr} \mapsto \text{mref}(l)],\sigma_{NL})\bigr)
$$

- ***Writing a Mutable Reference***: Similar to reading an immutable reference, this operation does not change the memory state and only validates the program’s access to a mutable reference.
                                     Also, similar to \( \mathsf{readImmut} \), the operation is valid if the mutable reference is reachable from the program environment.
                                     Formally, if \(k_{mr} \in \mathrm{dom}(\sigma_{L})\) and \(\sigma_{L}(k_{mr})=\text{mref}(l)\), then:

$$
\mathsf{writeMut}\bigl((\rho,(\sigma_L,\sigma_{NL})),\ k_{mr}\bigr)
=
\begin{cases}
(\rho,(\sigma_L,\sigma_{NL}))
& \text{if }\text{mref}(l) \in \text{Reachable}_{Ref}(\rho,\sigma) \\[4pt]
\mathsf{nonvalid}
& \text{otherwise.}
\end{cases}
$$

#### An Example

The following example demonstrates how a program can reach a simple memory state by using the defined operations.
It also shows how certain uses of these operations can lead the memory state to an incorrect form.
The example constructs a memory that contains two boxes: an outer box and an inner box.
The outer box points to the inner box, and the inner box points to a nonlinear integer.

Let \(k_{\mathsf{outer}}, k_{\mathsf{inner}} \in \text{Loc}_L\) and \(n_{42} \in \text{Loc}_{NL}\).
Assume the initial nonlinear memory already contains the integer:

$$
(\rho_0,(\sigma_{L_0},\sigma_{NL_0}))
=
\bigl(\emptyset,\ (\emptyset,\ \{n_{42} \mapsto 42\})\bigr)
$$

First, the program creates a box that points to the nonlinear integer:

$$
(\rho_1,(\sigma_{L_1},\sigma_{NL_1}))
=
\mathsf{newBox}\bigl((\rho_0,(\sigma_{L_0},\sigma_{NL_0})),\ n_{42},\ k_{\mathsf{inner}}\bigr)
$$

Resulting in:

$$
\sigma_{L_1} = \{k_{\mathsf{inner}} \mapsto \text{box}(n_{42})\},
\qquad
\sigma_{NL_1} = \{n_{42} \mapsto 42\}
$$

Second, the program creates a second box that points to the inner box:

$$
(\rho_2,(\sigma_{L_2},\sigma_{NL_2}))
=
\mathsf{newBox}\bigl((\rho_1,(\sigma_{L_1},\sigma_{NL_1})),\ k_{\mathsf{inner}},\ k_{\mathsf{outer}}\bigr)
$$

Which results in:

$$
\sigma_{L_2} =
\{
k_{\mathsf{inner}} \mapsto \text{box}(n_{42}),
\quad
k_{\mathsf{outer}} \mapsto \text{box}(k_{\mathsf{inner}})
\}
\qquad
\sigma_{NL_2} = \{n_{42} \mapsto 42\}
$$

Assume that the linear environment contains a variable \( \texttt{outer} \) that points to the outer box:

$$
\rho_{L_2} = \{\texttt{outer} \mapsto k_{\mathsf{outer}}\},
\qquad
\rho_{NL_2} = \emptyset
$$

Let \(k_{\mathsf{outerIr}} \in \text{Loc}_{NL}\) be fresh.
In the next step, the program borrows the outer box immutably:

$$
(\rho_3,(\sigma_{L_3},\sigma_{NL_3}))
=
\mathsf{borrowImmutBox}\bigl((\rho_2,(\sigma_{L_2},\sigma_{NL_2})),\ k_{\mathsf{outer}},\ k_{\mathsf{outerIr}}\bigr)
$$

So:

$$
\sigma_{NL_3} =
\{
n_{42} \mapsto 42,
\quad
k_{\mathsf{outerIr}} \mapsto \text{iref}(k_{\mathsf{inner}})
\}
$$

Assume that the nonlinear environment stores the borrowed reference in the variable \( \texttt{outerImm} \):

$$
\rho_{NL_3} = \{\texttt{outerImm} \mapsto k_{\mathsf{outerIr}}\}
$$

Next, the program accesses the \( \texttt{outerImm} \) reference using \( \mathsf{readImmut} \).
This operation is valid if \( \text{iref}(k_{\mathsf{inner}}) \) is reachable from the environment.

$$
\mathsf{readImmut}\bigl((\rho_3,(\sigma_{L_3},\sigma_{NL_3})),\ k_{\mathsf{outerIr}}\bigr)
=
(\rho_3,(\sigma_{L_3},\sigma_{NL_3}))
$$

To illustrate that this operation can produce a memory state that appears incorrect, the following presents two cases in which the operation result contains faults.
The later well-formedness definition rules out these incorrect states.

Let \(k'_{\mathsf{outer}} \in \text{Loc}_L\) be fresh.
The program can create a second box that also points to \(k_{\mathsf{inner}}\):

$$
(\rho_4,(\sigma_{L_4},\sigma_{NL_4}))
=
\mathsf{newBox}\bigl((\rho_3,(\sigma_{L_3},\sigma_{NL_3})),\ k_{\mathsf{inner}},\ k'_{\mathsf{outer}}\bigr)
$$

Resulting in:

$$
\sigma_{L_4} =
\{
k_{\mathsf{inner}} \mapsto \text{box}(n_{42}),
\quad
k_{\mathsf{outer}} \mapsto \text{box}(k_{\mathsf{inner}}),
\quad
k'_{\mathsf{outer}} \mapsto \text{box}(k_{\mathsf{inner}})
\}
\qquad
\sigma_{NL_4} =
\{
n_{42} \mapsto 42,
\quad
k_{\mathsf{outerIr}} \mapsto \text{iref}(k_{\mathsf{inner}})
\}
$$

In this state, two boxes point to the same inner box location \(k_{\mathsf{inner}}\).
This is not aligned with the linear tree memory structure that imem tries to preserve through the boxes.
The later well-formedness definition rules out this state.

In addition, the program can borrow the inner box mutably while the immutable reference to the outer box remains reachable.
Let \(k_{\mathsf{innerMr}} \in \text{Loc}_{L}\) be fresh.
Given the third memory state, \( (\rho_5, (\sigma_{L_5}, \sigma_{NL_5})) \), as input, the program can borrow the inner box mutably:

$$
(\rho_5,(\sigma_{L_5},\sigma_{NL_5}))
=
\mathsf{borrowMutBox}\bigl((\rho_3,(\sigma_{L_3},\sigma_{NL_3})),\ k_{\mathsf{inner}},\ k_{\mathsf{innerMr}}\bigr)
$$

Resulting in:

$$
\sigma_{L_5} =
\{
k_{\mathsf{inner}} \mapsto \text{box}(n_{42}),
\quad
k_{\mathsf{outer}} \mapsto \text{box}(k_{\mathsf{inner}}),
\quad
k_{\mathsf{innerMr}} \mapsto \text{mref}(n_{42})
\}
\qquad
\sigma_{NL_5} =
\{
n_{42} \mapsto 42,
\quad
k_{\mathsf{outerIr}} \mapsto \text{iref}(k_{\mathsf{inner}})
\}
$$

In this state, the program can access the immutable reference \(\texttt{outerImm}\), and it can also access the mutable reference \( \text{mref}(n_{42}) \).
This combination is not acceptable because the mutable reference can update the resource that the immutable reference can reach.
The later well-formedness definition rules out this situation.

#### Well-formedness

##### Additional Definitions

***References Mentioned in a Value:***
The references that appear in a value \(v\), denoted \(\text{Refs}(v)\), are defined as follows:

$$
\text{Refs}(v) =
\begin{cases}
\{\, v_i \mid v = \langle v_1, \dots, v_k \rangle
\;\wedge\; v_i \in \text{Loc}_L \cup \text{Loc}_{NL} \,\}
& \\[4pt]
\{l\}
& \text{if } v \in \{\text{box}(l),\ \text{iref}(l),\ \text{mref}(l)\} \\[4pt]
\emptyset
& \text{otherwise.}
\end{cases}
$$

***Resource:***
A resource is a location that has an imem \(\text{box}\) reference pointing to it:

$$
l \in \text{Res}(\sigma)
\iff
\exists k \in \mathrm{dom}(\sigma_L).\ \sigma_L(k)=\text{box}(l)
$$

***Reachability:***
The domain of the full memory is:

$$
\mathrm{dom}(\sigma)=\mathrm{dom}(\sigma_L)\ \cup\ \mathrm{dom}(\sigma_{NL})
$$

For any \(k \in \mathrm{dom}(\sigma)\), \(\sigma(k)\) denotes \(\sigma_L(k)\) if \(k \in \mathrm{dom}(\sigma_L)\), and \(\sigma_{NL}(k)\) if \(k \in \mathrm{dom}(\sigma_{NL})\).

A location \(l\) reaches another location \(l'\) in one step if and only if:

$$
l \to l'
\iff
l \in \mathrm{dom}(\sigma)\ \wedge\ l' \in \text{Refs}(\sigma(l))
$$

A location \(l\) reaches a location \(l'\) if and only if:

$$
l \to^{*} l'
\iff
\exists n \ge 1,\ \exists l_0,\dots,l_n\ \text{such that}\
l_0 = l\ \wedge\ l_n = l'\ \wedge\
\forall i \in \{0,\dots,n-1\}.\ l_i \to l_{i+1}
$$

A location \(l\) is reachable from a linear variable \(x \in \mathrm{dom}(\rho_L)\) iff:

$$
x \rightsquigarrow l \iff \rho_L(x) \to^{*} l
$$

A reaching path \(\pi \in \text{Paths}(x,l)\) from variable \(x\) to location \(l\) is a finite sequence of locations \((l_1,\dots,l_n)\) such that:

- \(l_1 = \rho_L(x)\),
- \(l_n = l\),
- and \(l_i \to l_{i+1}\) for all \(1 \le i < n\).

***Reachable References:***
The set of reachable references, \(\text{Reachable}_{Ref}\), contains every box, immutable reference, and mutable reference that is stored at a location reachable from a linear environment variable.
In addition, \(\text{Reachable}_{Ref}\) contains every immutable reference stored at a location that a nonlinear environment variable points to.

$$
\begin{aligned}
\text{Reachable}_{Ref}(\rho,\sigma)
=
&\ \{\, r
\mid
\exists x \in \mathrm{dom}(\rho_L).\ \exists k \in \mathrm{dom}(\sigma).\
x \rightsquigarrow k
\ \wedge\
\sigma(k)=r
\ \wedge\
r \in \{\text{box}(\_),\ \text{iref}(\_),\ \text{mref}(\_)\}
\,\} \\
&\ \cup\
\{\, r
\mid
\exists y \in \mathrm{dom}(\rho_{NL}).\ \exists k \in \mathrm{dom}(\sigma_{NL}).\
\rho_{NL}(y)=k
\ \wedge\
\sigma_{NL}(k)=r
\ \wedge\
r \in \{\text{iref}(\_)\}
\,\}.
\end{aligned}
$$

***Direct Boxes:***
The set of direct boxes of a location \(l\) is called \(D(l)\).
It contains all linear locations that store a box that points to \(l\):

$$
D(l,\sigma) = \{\, k \in \mathrm{dom}(\sigma_L) \mid \sigma_L(k)=\text{box}(l)\,\}
$$

##### Properties

An imem memory state \((\rho,\sigma)\) is well formed if it satisfies the following properties.
As indicated by the formal definitions of these properties, well-formedness mainly concerns about the portion of memory that is reachable from the environment.

***Direct Box Uniqueness:***
Every linear location \(l \in \text{Loc}_{L} \) has at most one reachable direct box that points to \(l\).
In other words, every linear location is either:

- Is not reachable from environment.
- Or exactly one linear composite value stores it.
- Or exactly one reachable box points to it.

Formally:

$$
\forall l \in \text{Loc}_{L}.\
\Rightarrow
\left|
\left\{
k \in D(l,\sigma)
\ \middle|\ 
\sigma_L(k) \in \text{Reachable}_{Ref}
\right\}
\right|
\leq 1
$$

***No Cyclic Box:***
A box \(b = \text{box}(l)\) must not reach itself through dereferencing:

$$
\neg \exists k \in \mathrm{dom}(\sigma).\
\bigl(\sigma(k)=\text{box}(l)\bigr)\ \wedge\ \bigl(l \to^{*} k\bigr)
$$

***Borrowing Validity:***
For every mutable or immutable reference that is reachable in memory, there exists a reachable box that points to the same location.

Formally:

$$
\forall k \in \mathrm{dom}(\sigma_L).\ \forall l.\
\Bigl(
\sigma_L(k)=\text{mref}(l) \in \text{Reachable}_{Ref}
\Bigr)
\Rightarrow
\Bigl(
\exists k_b \in \mathrm{dom}(\sigma_L).\ \sigma_L(k_b)=\text{box}(l) \in \text{Reachable}_{Ref}
\Bigr)
$$

$$
\forall k \in \mathrm{dom}(\sigma_{NL}).\ \forall l.\
\Bigl(
\sigma_{NL}(k)=\text{iref}(l) \in \text{Reachable}_{Ref}
\Bigr)
\Rightarrow
\Bigl(
\exists k_b \in \mathrm{dom}(\sigma_L).\ \sigma_L(k_b)=\text{box}(l) \in \text{Reachable}_{Ref}
\Bigr)
$$

This property means that every immutable and mutable reference is borrowed from a box that points to the same location.

***Immutable Reference Not Reaching Mutable Reference:***
A reachable immutable reference \(ir = \text{iref}(l_i)\) should not reach a location that a reachable mutable reference points to:

$$
\forall l_i, l_m.\
\Bigl(
\text{iref}(l_i) \in \text{Reachable}_{Ref}(\rho,\sigma)
\ \wedge\
\text{mref}(l_m) \in \text{Reachable}_{Ref}(\rho,\sigma)
\Bigr)
\Rightarrow
\neg (l_i \to^{*} l_m)
$$

This property results in immutable references reaching a constant portion of memory as long as they remain reachable.

#### Example Cont.

---------------------------------------------------------------------------------------------------------------

TODO: CONTINUATION OF THE EXAMPLE BUT SHOW THAT THE NOT WANTED OPERATIONS NOW MAKE THE MEMORY NOT WELL-FORMED AND IT'S OK

---------------------------------------------------------------------------------------------------------------

#### Well-formed But Not Correct

---------------------------------------------------------------------------------------------------------------

TODO: AN MIMAL EXAMPLE WHERE THE PROGRAM HAS TWO MUTABLE REFERENCE TO THE SAME RESOURCE, ONE TIME IT WRITE THE FIRST AND THEN THE SECOND AND ITS OK, BUT IF IT AGAIN WRITE THE FIRST ONE IT IS WRONG AND THAT IS WHERE LIFETIMES ARE NEEDED

---------------------------------------------------------------------------------------------------------------

### Memory with imem References And Lifetimes

To address the discussed issues, imem must relate the availability of some references to the unavailability of other references.
For example, borrowing a box mutably should invalidate all immutable references that are borrowed from the same box.
To achieve this, imem introduces lifetimes and a new type of reference, called a value holder, and augments existing references with lifetime sets.  
In addition, lifetimes enable static memory management.

A lifetime is an entity that becomes available once during program execution and then expires.
The memory state tracks the lifetimes that are currently available:

$$
(\rho, \sigma, \Lambda)
$$

Where $\Lambda \subseteq \mathcal{L}$ is the set of available lifetimes.
The set of linear values is:

$$
\begin{aligned}
  \text{Val}_L =
  &\ \{ \langle v_1, \dots, v_k \rangle \mid v_i \in \text{Loc}_{L} \cup \text{Loc}_{NL} \cup \mathbb{Z} \} \\
  &\ \cup \{ \text{box}(l, \tau) \mid l \in \text{Loc}_{L} \cup \text{Loc}_{NL},\ \tau \subseteq \mathcal{L} \} \\
  &\ \cup \{ \text{mref}(l, \tau) \mid l \in \text{Loc}_{L} \cup \text{Loc}_{NL},\ \tau \subseteq \mathcal{L} \} \\
  &\ \cup \{ \text{hold}(l, \alpha) \mid l \in \text{Loc}_{L} \cup \text{Loc}_{NL},\ \alpha \in \mathcal{L} \}
\end{aligned}
$$

Box and mutable references are augmented with a lifetime set \(\tau\).
A reference is available as long as all lifetimes in its associated set have not expired.
In addition, \(\text{Val}_L\) now includes a new reference \(\text{hold}(l, \alpha)\), called a value holder, which prevents access to location \(l\) until lifetime \(\alpha\) expires.

Furthermore, immutable references are also augmented with a lifetime set.
As a result, the set of nonlinear values takes the following form:

$$
\begin{aligned}
  \text{Val}_{NL} =
  &\ \{ \langle v_1, \dots, v_k \rangle \mid v_i \in \text{Loc}_{NL} \cup \mathbb{Z} \} \\
  &\ \cup \{ \text{iref}(l, \tau) \mid l \in \text{Loc}_{L} \cup \text{Loc}_{NL},\ \tau \subseteq \mathcal{L} \}
\end{aligned}
$$

#### New and Changed Program Needed Operations

---------------------------------------------------------------------------------------------------------------

The introduction of lifetimes adds two new operations: \(\mathsf{addLifetime}\) and \(\mathsf{removeLifetime}\).
TODO: MATHEMATICAL AND EXPLAINATION OF ADDLIFETIME AND REMOVELIFETIME

TODO: MAKE THE FOLLOWING CHANGES TO THE OPERATIONS:
- BORROWING OPERATIONS ARE VALID AS LONG AS THE BORROWING REFERENCE PATH TO ENVIRONMENT DOES NOT ENCOUNTER ANY VALUE HOLDERS THAT THE LIFETIME IS NOT EXPIRED
- THE BORROW OPERATIONS CREATE VALUE HOLDERS FOR THE BORROWED
- THE LIFETIME SET OF BORROWED REFERENCES IS MONOTONE MEANING A SUPERSET OF ALL REFERENCES REACHING IT.

---------------------------------------------------------------------------------------------------------------

These operations ensure that the memory remains well formed, as formally defined in the following subsections.

#### Well-formedness

Similar to the previous version, memory well-formedness concerns only the part that is reachable from the environment and, in addition, available along the entire path from the environment to the location.

##### Additional Definitions

The *Resource*, *Reachability*, and *Direct Boxes* definitions remain the same.  
The definition of *References Mentioned in a Value* changes slightly by adding value holder references:

$$
\text{Refs}(v) =
\begin{cases}
\{\, v_i \mid v = \langle v_1, \dots, v_k \rangle
\;\wedge\; v_i \in \text{Loc}_L \cup \text{Loc}_{NL} \,\}
& \\[4pt]
\{l\}
& \text{if } v \in \{\text{box}(l,\tau),\ \text{ref}(l,\tau),\ \text{mref}(l,\tau),\ \text{hold}(l,\alpha)\} \\[4pt]
\emptyset
& \text{otherwise.}
\end{cases}
$$

The following presents the new definitions.

***Availability of References:***
imem associates a set of lifetimes \(\tau \subseteq \mathcal{L}\) with the references \(\text{box}(l, \tau)\), \(\text{mref}(l, \tau)\), and \(\text{iref}(l, \tau)\).  
If any lifetime in \(\tau\) expires, the reference becomes unavailable.
Formally, a value \(v \in \{ \text{box}(l, \tau), \text{mref}(l, \tau), \text{iref}(l, \tau) \}\) is available iff:

$$
\text{Avail}(v, \Lambda) \iff \tau \subseteq \Lambda
$$

***Available Linear Environment:***
The set of available linear variables, $\text{Var}_{Avail}$, consists of all linear variables in $\rho_L$ for which all references they reach are available:

$$
\text{Var}_{\mathit{Avail}}(\rho,\sigma,\Lambda) =
\{\, x \in \mathrm{dom}(\rho_L) \mid
\forall r \in \text{RefsReach}(x,\rho,\sigma).\ \text{Avail}(r,\Lambda) \,\}
$$

Where $\text{RefsReach}$ is defined as follows:

$$
\text{RefsReach}(x,\rho,\sigma) = \{\, r \mid \exists k.\ x \rightsquigarrow k\ \wedge\ \sigma(k)=r\ \wedge\ r \in \{\text{box}(\_,\_),\text{ref}(\_,\_),\text{mref}(\_,\_)\} \,\}
$$

***Available Reachable References:***
The set of available reachable references, \(\text{AR}_{Ref}\), contains every available box, immutable reference, and mutable reference that is stored at a location reachable from an available linear environment variable:

$$
\begin{aligned}
\text{AR}_{Ref}(\rho,\sigma,\Lambda)
=
\{\, r \mid\ &
\exists x \in \text{Var}_{\mathit{Avail}}(\rho,\sigma,\Lambda).\
\exists k \in \mathrm{dom}(\sigma).\
x \rightsquigarrow k
\ \wedge\ \sigma(k)=r \\
&\wedge\ r \in \{\text{box}(\_,\_),\ \text{iref}(\_,\_),\ \text{mref}(\_,\_)\}
\ \wedge\ \text{Avail}(r,\Lambda)
\,\}.
\end{aligned}
$$

***Access Expiration:***
Accessing a reference \(r \in \{\text{box}(l,\_), \text{iref}(l,\_), \text{mref}(l,\_)\}\) expires a lifetime set \(\tau\) if either \(\tau\) is already expired, or every path that starts from an available linear variable and reaches \(r\) passes through a value holder \(\text{hold}(\_, \alpha)\) such that \(\alpha \in \tau\).  
Formally:

$$
\begin{aligned}
  \text{AExp}_{(\rho,\sigma,\Lambda)}(r,\tau) \iff\ &
  \tau \cap \Lambda \neq \emptyset \vee
  \forall x \in \text{Var}_{\mathit{Avail}}(\rho,\sigma,\Lambda).\ 
  \forall k \in \text{dom}(\sigma).\ 
  \Bigl(\sigma(k)=r \wedge x \rightsquigarrow k \Rightarrow \\
    &\qquad \forall \pi \in \text{Paths}(x,k).\ 
    \exists \alpha \in \tau.\ 
    \exists i.\ \sigma(\pi_i)=\text{hold}(\_,\alpha)
  \Bigr)
\end{aligned}
$$

#### Properties

An imem memory state $(\rho, \sigma, \Lambda)$ is well-formed if it satisfies the following properties.

***Direct Box Uniqueness:***
Every linear location \(l \in \text{Loc}_{L} \) has at most one reachable available direct box that points to \(l\):

$$
|D(l,\sigma) \cap \text{Box}_{AR}(\rho,\sigma,\Lambda)| \leq 1
$$

***No cyclic box***:
The same as previous definition.

***No dangling references:***
For any two references \(r_1 \in \{\text{box}(l_1, \tau_1), \text{iref}(l_1, \tau_1), \text{mref}(l_1, \tau_1)\}\) and  \(r_2 \in \{\text{box}(l_2, \tau_2), \text{iref}(l_2, \tau_2), \text{mref}(l_2, \tau_2)\}\), and for any location \(k\) such that \(\sigma(k) = r_2\), if \(l_1\) reaches \(k\) and \(r_2\) becomes unavailable, then \(r_1\) also becomes unavailable.

$$
l_1 \to^{*} k \implies \tau_2 \subseteq \tau_1
$$

This property ensures that all references reachable from available references are available.

***Borrowing Validity:***
For every reference \(r \in \{\text{mref}(l,\tau_r), \text{iref}(l,\tau_r)\}\), there exists a box \(b=\text{box}(l,\tau_b)\) that $\tau_b \subseteq \tau_r$.

This property means that every immutable and mutable reference is borrowed from a box, and that box always outlives the reference.

***Box reaching Reference:***
If a box $b = \text{box}(l_b, \tau_b)$ reaches a location that a mutable or immutable reference $r \in \{\text{iref}(l_r, \tau_r), \text{mref}(l_r, \tau_r)\}$ points to it, then accessing $b$ should expire $r$:

$$
l_b \to^{*} l_r \implies \text{AExp}_{(\rho,\sigma,\Lambda)}(b,\tau_r)
$$

***Mutable Reference reaching Immutable Reference:***
If a mutable reference $mr = \text{mref}(l_m, \tau_m)$ reaches a location that an immutable reference $ir = \text{iref}(l_i, \tau_i)$ points to it, then accessing $mr$ should expire $ir$:

$$
l_m \to^{*} l_i \implies \text{AExp}_{(\rho,\sigma,\Lambda)}(mr,\tau_i)
$$

***Mutable Reference reaching Mutable Reference:***
If a mutable reference \(m_1 = \text{mref}(l_1,\tau_1)\) reaches another mutable reference \(m_2 = \text{mref}(l_2,\tau_2)\):

- If $l_1 \neq l_2$:
  Accessing $m_1$ should expire $m_2$, meaning: $(l_1 \to^{*} l_2)\ \wedge\ (l_1 \neq l_2) \implies \text{AExp}_{(\rho,\sigma,\Lambda)}(m_1,\tau_2)$.

- If $l_1 = l_2$:
  Either accessing $m_1$ should expire $m_2$ or vice versa, formally: $(l_1 \to^{*} l_2)\ \wedge\ (l_1 = l_2) \implies \Bigl( \text{AExp}_{(\rho,\sigma,\Lambda)}(m_1,\tau_2) \vee\ \text{AExp}_{(\rho,\sigma,\Lambda)}(m_2,\tau_1) \Bigr)$.

The three reaching properties ensure that references follow the [Stacked Borrows Model](../background/stacked-borrows.md).
Accessing a box expires all references borrowed from it, as well as references borrowed from reachable boxes that point to locations reachable from the box.
Accessing a mutable reference expires all mutable and immutable references borrowed from it or borrowed from its reachable boxes.

***Immutable Reference not reaching Mutable Reference:***
An available reachable immutable reference $ir = \text{iref}(l_i, \tau_i)$  should not reach a location that an available reachable mutable reference points to:

$$
\forall ir, mr.\ 
\Bigl(
ir \in \text{AR}_{Ref}(\rho,\sigma,\Lambda) \wedge mr \in \text{AR}_{Ref}(\rho,\sigma,\Lambda)
\wedge ir=\text{iref}(l_i,\tau_i) \wedge mr=\text{mref}(l_m,\tau_m)
\Bigr)
\implies
\neg (l_i \to^{*} l_m)
$$

### Overview

An imem memory is a linear memory extended with [additional references](#memory-with-imem-references) and [lifetimes](#memory-with-imem-references-and-lifetimes), with a specific definition of [well-formedness](#well-formedness-2).
In a well-formed imem memory, the structure formed by connecting all available boxes to the values stored at the locations they point to is a tree.
This structure arises because the *Direct Box Uniqueness* property guarantees that each value is pointed to by exactly one box, and the *No Cyclic Box* property prevents boxes from forming cycles.
As a result, this structure closely resembles the tree of linear values in a well-formed linear memory.

imem allows programs to borrow immutable and mutable references from boxes.
When edges from immutable and mutable references to the values at their target locations are added, the tree becomes a directed acyclic graph.
This graph remains acyclic because references do not introduce cycles.
Moreover, due to the *No Dangling References* property, following any available reference whose lifetime has not expired never leads to an unavailable reference.
In addition, every available immutable or mutable reference is ultimately borrowed from an available box, which is ensured by the *Borrowing Validity* property.

The reaching properties further restrict how references can appear in this graph.
If a variable points to an available box, then there are no available immutable or mutable references pointing to any location within the subtree of that box.
If a variable points to an available mutable reference, then accessing the boxes that reach this reference, as well as the mutable references it is borrowed from, necessarily goes through a value holder that expires the mutable reference.
As a result, there are no mutable or immutable references pointing to any location within the subtree of the box from which the mutable reference is borrowed.

Finally, due to the Immutable Reference Not Reaching Mutable Reference property, if a variable points to an available immutable reference, then no mutable reference points to a location whose value lies in the subtree of the corresponding box.
Accessing any mutable reference or box that reaches the location targeted by this immutable reference causes the immutable reference to expire.

The following diagram illustrates the available and reachable part of a well-formed imem memory:

TODO: ADD A DIAGRAM FOR IMEM MEMORY

### An Example

The following example shows a two-element list encoded in imem.

The set of active lifetimes is:

$$
\Lambda = \{\alpha\}
$$

The linear environment contains a single variable that points to the list:

$$
\rho_L = \{ \texttt{list} \mapsto l_{\mathsf{list}} \}
$$

The linear memory is composed of boxes and linear values that represent the structure of the list and its elements:

$$
\begin{aligned}
\sigma_L = \{&
l_{\mathsf{list}} \mapsto \text{box}(l_1,\{\alpha\}), \\[2pt]
&l_1 \mapsto \langle l_{\mathsf{h1}},\ l_{\mathsf{t1}} \rangle, \\
&l_{\mathsf{h1}} \mapsto \text{box}(n_{42},\{\alpha\}), \\
&l_{\mathsf{t1}} \mapsto \text{box}(l_2,\{\alpha\}), \\[2pt]
&l_2 \mapsto \langle l_{\mathsf{h2}},\ l_{\mathsf{t2}} \rangle, \\
&l_{\mathsf{h2}} \mapsto \text{box}(n_{99},\{\alpha\}), \\
&l_{\mathsf{t2}} \mapsto \text{box}(l_{\texttt{none}},\{\alpha\}), \\[2pt]
&l_{\texttt{none}} \mapsto \langle\ \rangle
\}
\end{aligned}
$$

The nonlinear memory stores the integers:

$$
\sigma_{NL} =
\{
n_{42} \mapsto 42,
\quad
n_{99} \mapsto 99
\}
$$

The variable \(\texttt{list}\) points to a location that contains a box referencing the first element of the list.  
This first element is stored at \(l_1\) and is a linear value composed of two boxed fields.  
The first field, located at \(l_{\mathsf{h1}}\), is a box that points to a nonlinear integer value.  
The second field, located at \(l_{\mathsf{t1}}\), is a box that points to the second element of the list.

TODO: A DIAGRAM of how it looks

The following example extends the previous list by adding a mutable reference to the first element and an immutable reference to the second element, while keeping the memory well formed.

The set of available lifetimes now includes three elements:

$$
\Lambda = \{\alpha,\beta,\gamma\}
$$

The lifetimes \(\beta\) and \(\gamma\) are newly introduced. Lifetime \(\beta\) is added to the lifetime set of the mutable reference, \(\tau_m = \{\alpha,\beta\}\), while lifetime \(\gamma\) is added to the lifetime set of the immutable reference, \(\tau_i = \{\alpha,\beta,\gamma\}\).

The environments now contain three variables:

$$
\rho_L =
\{
\texttt{list} \mapsto l_{\mathsf{list}},
\quad
\texttt{firstMut} \mapsto l_{\mathsf{mut}}
\}
\qquad
\rho_{NL} =
\{
\texttt{secondImm} \mapsto n_{\mathsf{secondImm}}
\}
$$

To support these references, the linear memory is extended with additional holds and a mutable reference:

$$
\begin{aligned}
\sigma_L = \{&
\underbrace{l_{\mathsf{list}} \mapsto \text{hold}(l_{\mathsf{listBox}},\beta)}_{\texttt{list} \text{ reaches the root box through a }\beta\text{-hold}}, \\
&l_{\mathsf{listBox}} \mapsto \text{box}(l_1,\{\alpha\}), \\[2pt]
&l_1 \mapsto \langle l_{\mathsf{h1}},\ l_{\mathsf{t1}} \rangle, \\
&l_{\mathsf{h1}} \mapsto \text{box}(n_{42},\{\alpha\}), \\
&l_{\mathsf{t1}} \mapsto \text{box}(l_2,\{\alpha\}), \\[2pt]
&l_2 \mapsto \langle l_{\mathsf{h2}},\ l_{\mathsf{t2}} \rangle, \\
&l_{\mathsf{h2}} \mapsto \text{box}(n_{99},\{\alpha\}), \\
&l_{\mathsf{t2}} \mapsto \text{box}(l_{\texttt{none}},\{\alpha\}), \\
&l_{\texttt{none}} \mapsto \langle\ \rangle, \\[6pt]
&\underbrace{l_{\mathsf{mut}} \mapsto \text{hold}(l_{\mathsf{mutCell}},\gamma)}_{\texttt{firstMut} \text{ reaches the mref through a }\gamma\text{-hold}}, \\
&l_{\mathsf{mutCell}} \mapsto \text{mref}(n_{42},\{\alpha,\beta\})
\}
\end{aligned}
$$

The nonlinear memory is also extended to include an immutable reference:

$$
\sigma_{NL} =
\{
n_{42} \mapsto 42,
\quad
n_{99} \mapsto 99,
\quad
n_{\mathsf{secondImm}} \mapsto \text{iref}(n_{99},\{\alpha,\beta,\gamma\})
\}
$$

In this configuration, imem allows the program to hold a mutable reference to the first list element and an immutable reference to the second element at the same time, without requiring the list to be deconstructed.

If the program accesses the box that points to the first element, the access must pass through the hold at location \(l_{\mathsf{list}}\). This action expires lifetime \(\beta\), which in turn causes both the mutable reference and the immutable reference to become unavailable.

Similarly, accessing the mutable reference passes through the hold at location \(l_{\mathsf{mut}}\). This access expires lifetime \(\gamma\), which results in the immutable reference becoming unavailable.

TODO: A DIAGRAM THAT THE LIST IN IMEM WITH THE REFERENCES

### Memory Management

In a well-formed imem memory, it is safe to free a location that points to an unavailable box, as well as the location that the unavailable box points to, if no available direct box still points to that location.
This safety follows from the fact that once a box becomes unavailable, it is no longer reachable from any available linear variable, and all references to it are also unavailable.

It is also safe to free a location that points to an immutable or mutable reference.
For the same reason, no available variable can reach such references once they become unavailable.

Moreover, as in a purely linear memory, when a linear location that points to a composite linear value is freed, the locations contained within that value can also be safely freed, unless the locations point to a box, an immutable reference, or a mutable reference.

In this way, all memory regions reachable through references are managed safely and statically.

## imem Implementation

The Scala library imem enforces static rules to ensure that the part of program memory it manages remains well formed throughout execution.  
These static rules include [ownership](./ownership.md) rules and [borrow-checking](./borrow-checking.md) rules.
Because Scala provides many features that may interfere with these rules, the [soundness](./soundness.md) section presents guidelines that a program must follow to keep its memory well formed.
