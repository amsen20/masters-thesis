# Soundness


<!-- TODO: Fix this -->
Also, `WC^` and `MC^` has to be type parameters to 
because the program has access to only one instance of `Context` that is provided to it by imem.
Furthermore, The program is not able to create an instance of `Context`.
So each function has to have two type parameters that the instantiate the `Context` type parameters.

<!-- TODO: Explain `NeverUsableKey` -->
<!-- TODO: Mention that Box is the only source of mutation -->
