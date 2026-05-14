---
name: iir-identifiability
description: Use when a user explicitly asks to apply Invariant Image Reparameterisation (IIR) to a new mathematical, statistical, mechanistic, ODE, compartmental, or simulation model. Guides a non-expert user through practical identifiability analysis using the methods of arXiv:2502.04867 and the reparam Julia library, starting with log-monomial coordinates but not treating monomials as the only possible class.
---

# IIR Identifiability Analysis for New Models

Use this skill when a user explicitly asks to apply **Invariant Image Reparameterisation (IIR)** to a new mathematical, statistical, mechanistic, ODE, compartmental, or simulation model. This follows the methods described in <https://arxiv.org/abs/2502.04867> and implemented at <https://github.com/omaclaren/reparam>.

If the user asks only for general modelling or general identifiability advice and does not mention IIR, do not assume they want this workflow. Ask whether they want to use IIR.

## Audience and interaction style

Assume the user may know their model and may have a high-level understanding of identifiability, but may not know IIR, automatic differentiation, invariant images, null spaces, or profile-wise analysis. The goal is to enable identifiability analysis first and foremost.

- Guide the user step by step from their model to interpretable conclusions.
- Prefer practical questions and concrete code over theory-first exposition.
- Explain only the amount of IIR terminology needed for the current decision.
- Translate outputs into statements such as:
  - “these parameters appear separately identifiable”,
  - “the observable behaviour appears to depend on these parameters through this combination”,
  - “this coordinate is weakly informed rather than exactly non-identifiable”,
  - “this conclusion depends on the chosen observations and auxiliary mapping”.
- Do not modify the `reparam` library source unless the user explicitly asks. Use existing functionality wherever possible.

If the model itself needs to be formulated from reactions or compartments, also use the `compartmental-models` skill.

## Core task

The central task is to help the user define an **auxiliary mapping**

```text
φ(θ)
```

from model parameters to observable quantities or data distribution parameters. For an ODE or simulation model, explain this as: “the deterministic map from parameters to the observable behaviour we want to preserve.” This may be a model solution on a fine grid, selected state variables at observation times, summary statistics, or mean/variance parameters of an observation model.

The current implementation works best when the transformed auxiliary mapping

```text
φ*(θ*) = φ(f⁻¹(θ*))
```

can be differentiated with `ForwardDiff.jl`. The agent should also help the user choose suitable initial componentwise transformations `f` and `g`. Multiple choices may need to be tested either separately or sequentially.

Monomials are one important and well-supported class, not the definition of IIR. The core invariant-subspace calculation acts on the transformed auxiliary mapping `φ*` and does not require monomial coordinates. The usual choice `f = log`, `g = exp` gives log-linear / monomial image coordinates. Other choices, such as identity transformations, logit-type transformations, or model-specific transformed coordinates, can still be tested by defining `φ*` explicitly; they may require interpreting the SVD basis directly or writing a custom basis search rather than using the monomial labels.

Sequential IIR can generate richer classes. For example, a first log-monomial pass may produce image coordinates `m(θ)` whose entries are monomials in the original parameters. A second pass using `f = identity` and `g = identity` on `m` gives coordinates of the form `B*m(θ)`, i.e. linear combinations of monomials such as sums and differences. More generally, each stage searches for a sparse linear basis in the current transformed coordinates; the interpretation depends on what the current coordinates and transformation `f` are.

Use this as a simple **IIR Search** philosophy before introducing broad custom transformations. Try a small menu of natural stage types, usually `L = log/monomial` and `I = identity/additive`, and short sequences such as `L`, `I`, `L → I`, and `I → L`. Let IIR classify each stage: if the invariant-null dimension equals the local null dimension, stop; if the stage is exact but non-minimal, pass the retained image coordinates (not the null/free coordinates) to the next simple stage. Treat non-minimal reductions as promising intermediate coordinates, not failures. This covers basic patterns such as `n₁p₁+n₂p₂` via `L → I`, and additive rate combinations such as `α+ω` via `I` or `L → I`.

The sparse search helpers in the library are most directly interpretable as monomial searches when the current transformed coordinates are logs of positive parameters. In other transformed coordinates, the same candidate vectors can still be useful as sparse basis vectors, but their labels should be generated or rewritten according to the current transformation. For example, the vector `[1, 1]` means a product under `f = log`, but a sum under `f = identity`. When available, use `basis_labels(candidates, coord_names; representation=:linear)` for identity-stage or additive interpretations, and `representation=:monomial` for log-monomial interpretations.

## Standard workflow

1. **Clarify the modelling problem**
   - What are the parameters `θ`?
   - What are their units, bounds, signs, and constraints?
   - What state variables or outputs are observed?
   - Is there data and a likelihood, or only nominal parameter values?
   - Is the goal structural identifiability, practical identifiability, prediction uncertainty, or a first diagnostic screen?

2. **Define the auxiliary mapping `φ(θ)`**
   - For likelihood-based work, `φ(θ)` can map to data distribution parameters.
   - For deterministic temporal/spatial models, `φ(θ)` can be the predicted observable solution.
   - The auxiliary mapping should match the observation operator or prediction target relevant to the question. Changing the observation map can change the identifiability result.

3. **Choose simple transformations `f` and `g` first**
   - Default log-monomial case for positive or fixed-sign parameters:
     ```text
     θ* = f(θ) = log(θ)
     ψ(θ) = g(Af(θ)) = exp(A log(θ))
     ```
   - Treat this as one first practical class to try, especially for scale-based mechanistic models, not as the only possible IIR class.
   - Also try `f = identity`, `g = identity` when additive structure is plausible or when a previous non-minimal log stage has produced image coordinates that may combine additively.
   - Before hand-designing clever model-specific transformations, try a small IIR Search menu such as `L`, `I`, `L → I`, and `I → L`, where `L` is log/monomial and `I` is identity/additive.
   - If parameters may be negative but their signs are known or can be estimated, use the fixed-sign approach below.
   - If parameters are probabilities or otherwise bounded, consider an appropriate one-to-one componentwise transformation only when it is genuinely natural for the parameter domain. Explain that this is already a broader class than the basic `L/I` search; for example, `logit(p)` is a log-ratio of `p` and `1-p`, not something automatically generated by ordinary log and identity stages on `p` alone.
   - Non-componentwise transformed coordinates can be used if the user has a scientifically meaningful invertible transformation, but then the agent should define `φ*` explicitly and avoid claiming monomial interpretation unless it is actually present. Do not present hand-designed coordinates as if they were discovered by the simple search.

4. **Find a reference point**
   - Prefer a maximum likelihood estimate `θ̂` when data and a likelihood are available.
   - Otherwise use a scientifically meaningful nominal value or fitted parameter value.
   - IIR is a local numerical calculation used to recover globally meaningful reparameterisations under the method’s assumptions, so repeat or perturb the reference point when possible.

5. **Construct and test `φ*(θ*)`**
   - Define `φ*(θ*) = φ(f⁻¹(θ*))`.
   - Before running IIR, test that `φ*` works with `ForwardDiff.jl`.

6. **Run the existing IIR routines**
   - Use `find_invariant_subspace(φ_star, θstar_hat)`.
   - Inspect singular values, numerical rank, invariant null basis `N`, and complementary basis `N_perp`.
   - Compute the local null dimension `p - rank_J` and the minimality defect `Δ_min = (p - rank_J) - size(N, 2)`.
   - Interpret the result as minimal image (`Δ_min == 0`), non-minimal image (`Δ_min > 0` but useful invariant-null directions/image coordinates found), or rank-deficient with no invariant-null direction found in this transformed representation.

7. **If useful, continue IIR Search sequentially**
   - If `Δ_min == 0`, the current transformed representation is locally minimal for the chosen auxiliary map; stop unless the user wants alternative coordinate choices or validation.
   - If `Δ_min > 0` and the stage gives a simpler exact image (for example, products such as `n₁p₁` and `n₂p₂`), treat those retained image coordinates as a lower-dimensional starting parameterisation for a second simple stage.
   - Pass image/retained coordinates to the next stage; do not pass invariant-null/free coordinates as the next image parameterisation.
   - Prefer simple follow-up stages first: identity/additive after log/monomial (`L → I`) and log/monomial after identity/additive (`I → L`).
   - If no short simple sequence improves the defect, report that the structure was not found in the tested `L/I` search class rather than forcing a coordinate.

8. **Choose interpretable image coordinates**
   - Use the default SVD result for diagnostics.
   - For log-monomial interpretation, use sparse monomial basis search:
     - local information from the Jacobian on the image side (`N_perp`),
     - simplicity on the invariant-null side (`N`).
   - For other transformed coordinates or sequential stages, treat the same idea as sparse linear basis selection in the current transformed coordinates. Label coordinates according to the current transformation, rather than automatically using monomial/product labels. For identity-stage additive reductions, use linear labels when available, e.g. `basis_labels(result.selected, current_names; representation=:linear)`.
   - Remind the user that sparse bases are not unique; they are interpretable coordinate choices spanning the same invariant image or invariant-null subspace.

9. **Validate**
   - Check sensitivity to numerical tolerances.
   - Repeat at nearby or alternative reference points when feasible.
   - Perturb along proposed invariant-null directions and verify that observable behaviour is unchanged or nearly unchanged.
   - Use profile likelihoods for important coordinates when practical identifiability is the main question.

10. **Optional PWA / prediction analysis**
   - Once useful image coordinates are chosen, profile them if uncertainty or prediction effects matter.
   - Push accepted/profile parameter values through prediction functions to see which coordinates carry uncertainty in observed or unobserved outputs.

## Rank, local null space, and invariant null space

Be careful with rank-deficient outputs. A local null space is not the same thing as an invariant null space.

- If `φ*` has `m` outputs and `p` parameters, the Jacobian rank is at most `m`; scalar auxiliary mappings are always locally rank-limited when `p > 1`.
- The local null-space dimension is `p - rank_J`.
- `find_invariant_subspace` returns only the invariant component of that local null space as `N`.
- The minimality defect is `Δ_min = (p - rank_J) - size(N, 2)`. Use `Δ_min == 0` as the local stopping condition for a candidate transformed representation. Use `Δ_min > 0` as a warning that the current representation is non-minimal or does not capture the full local null structure.
- If the Jacobian is rank deficient but `size(N, 2) == 0`, report this as a local ridge or local non-identifiability with no constant invariant-null direction found in the chosen transformed coordinates. Do not call the individual parameters identifiable just because `N` is empty.
- In non-minimal image cases, `N_perp` contains both locally informed directions and local null directions that failed the invariance test. Do not label every `N_perp` coordinate as “identified”. Prefer “image/complement coordinate”, “retained coordinate”, or “non-minimal image coordinate” unless profile likelihoods or rank information support “identified”.
- Distinguish image coordinates from invariant-null/free coordinates. If the observable behaviour depends on `β/γ`, then `β/γ` is an image coordinate; a complementary coordinate such as `β*γ` parameterises variation the observable does not see.

When a low-dimensional auxiliary mapping leaves a curved ridge, IIR may correctly return no sparse invariant-null coordinate. In that case, use profile likelihoods, level-set tracing, or PWA rather than forcing a monomial interpretation. Unless a separate mathematical argument is supplied, phrase this as “no invariant-null coordinate was found in the chosen transformed coordinates / basis class”, not “no possible reparameterisation exists”.

## Known design variables and lifted maps

Sometimes it is useful to temporarily treat known design variables, observation times, initial conditions, or input settings as entries of an enlarged parameter vector. This can expose scaling or dimensional invariants of the family of experiments. However:

- Clearly distinguish this enlarged analysis from identifiability of the unknown model parameters under a fixed experimental design.
- If a design variable is known or directly observed, either keep it fixed when interpreting the original problem or include it as an observed component of the auxiliary mapping.
- Do not present an invariant from a lifted map as a non-identifiability of the original fixed-design problem without checking how the restriction changes the result.

## ForwardDiff compatibility is a preflight requirement

The current implementation evaluates derivatives of the transformed auxiliary mapping using `ForwardDiff.jl`. This is required not only for the Jacobian `D_{θ*}φ*(θ̂*)`, but also for the first-order invariance calculation used to extract the invariant component of the local null space.

Therefore `φ*(θ*)` must work when `θ*` is a vector whose element type is `ForwardDiff.Dual`, not only `Vector{Float64}`.

### Do

- Write functions against generic vectors, e.g. `AbstractVector{T}` where useful.
- Allocate arrays using the parameter/state element type, e.g. `T = eltype(θstar); zeros(T, n)`.
- Convert fixed initial conditions to the active numeric type when they interact with Dual-valued parameters.
- Keep ODE right-hand sides type-generic.
- Return a vector of finite values from `φ*`.
- Keep `φ*` deterministic and smooth where possible.

### Avoid

- Hard-coded `Vector{Float64}` outputs inside `φ*`.
- `zeros(Float64, n)` followed by assignment of Dual-valued expressions.
- `Float64(x)`, `convert(Float64, x)`, or `Float64.(...)` inside `φ*`.
- Random simulation inside `φ*`.
- Non-smooth thresholds, discontinuous events, callbacks, or branch changes unless handled deliberately.
- External black-box code that strips Dual numbers.

### Important limitation

Passing a custom finite-difference Jacobian is not a full escape hatch for the current `find_invariant_subspace` implementation, because the invariance test differentiates through the Jacobian calculation. If `φ*` cannot be made `ForwardDiff`-compatible, pause before making structural claims. Options include rewriting/simplifying `φ*`, using a differentiable surrogate, doing only local finite-difference SVD diagnostics, or implementing a separate finite-difference invariance calculation.

## Fixed-sign log-monomial handling

For log-monomial IIR, positive parameters are simplest. If parameters may be negative but their signs are known or can be estimated, first find a reference point, preferably a maximum likelihood estimate `θ̂`. Fix signs

```julia
signs = sign.(θhat)
```

and work in the fixed-sign region using

```julia
θstar_to_θ(θstar) = signs .* exp.(θstar)
θ_to_θstar(θ) = log.(abs.(θ))
φ_star(θstar) = φ(θstar_to_θ(θstar))
```

This is equivalent to applying the log function to the absolute value of each entry and reintroducing the estimated signs in the auxiliary mapping. Do not apply fractional monomial powers directly to negative parameters. If an estimated parameter is near zero, the fixed-sign log transformation is not appropriate without further modelling decisions.

Bounds must stay within a fixed-sign region. If a bound crosses zero, ask the user to choose a different transformation, refit with sign constraints, or split the analysis into sign-specific regions.

## Minimal Julia scaffold

Use the existing `reparam` library rather than reimplementing IIR. Ask the user for the path to their local checkout, or clone/use <https://github.com/omaclaren/reparam>.

```julia
include("/path/to/reparam/ReparamTools.jl")
using .ReparamTools
using ForwardDiff
using LinearAlgebra

# User/model-specific pieces:
param_names = ["β", "γ", "N"]
θ_lower = [...]
θ_upper = [...]
θ_initial = [...]

# Define φ in original parameters.
# It should return observable quantities or data distribution parameters.
φ(θ) = begin
    # model-specific deterministic calculation
    return [...]
end

# If a likelihood is available, define it in original parameters.
lnlike_θ(θ) = begin
    # model-specific log-likelihood
    return 0.0
end

# Positive-parameter default. For fixed-sign parameters, replace this
# with θstar_to_θ(θstar) = signs .* exp.(θstar).
θ_to_θstar(θ) = log.(θ)
θstar_to_θ(θstar) = exp.(θstar)
φ_star(θstar) = φ(θstar_to_θ(θstar))
lnlike_star(θstar) = lnlike_θ(θstar_to_θ(θstar))

θstar_lower = θ_to_θstar(θ_lower)
θstar_upper = θ_to_θstar(θ_upper)
θstar_initial = θ_to_θstar(θ_initial)

# MLE in transformed coordinates if likelihood is available.
θstar_hat, ll_hat = profile_target(
    lnlike_star, Int[], θstar_lower, θstar_upper, θstar_initial;
    grid_steps=[100],
)

# ForwardDiff preflight.
y0 = φ_star(θstar_hat)
@assert y0 isa AbstractVector
@assert all(isfinite, y0)
J = ForwardDiff.jacobian(φ_star, θstar_hat)
@assert all(isfinite, J)

# IIR calculation.
S, N, N_perp, rank_J = find_invariant_subspace(φ_star, θstar_hat; verbose=true)
J = compute_ϕ_Jacobian(φ_star, θstar_hat)

println("Singular values: ", S)
println("Numerical rank: ", rank_J)
println("Invariant-null dimension: ", size(N, 2))
println("Image/complement dimension: ", size(N_perp, 2))

# Sparse monomial basis search for interpretable coordinates.
p = length(param_names)
identified = informed_monomial_basis_search(
    N_perp, J' * J, S[1]^2, param_names;
    s_max=2, c_max=1, residual_cap=1e-2,
)

if size(N, 2) > 0
    null = simple_monomial_basis_search(
        N, param_names;
        s_max=2, c_max=1, residual_cap=1e-2, retry_support=true,
    )
    A_cols = hcat(
        monomial_basis_matrix(identified.selected, p),
        monomial_basis_matrix(null.selected, p),
    )
    labels = vcat(basis_labels(identified.selected), basis_labels(null.selected))
else
    A_cols = monomial_basis_matrix(identified.selected, p)
    labels = basis_labels(identified.selected)
end

println("Selected coordinates:")
for (i, label) in enumerate(labels)
    println("  ψ_", i, " = ", label)
end

# For the positive-parameter default, reparam expects columns = parameter combinations.
θ_to_ψ, ψ_to_θ = reparam(A_cols)

# For fixed-sign or custom transformations, define the maps explicitly instead.
# In the fixed-sign log-monomial case:
# A = A_cols'
# θ_to_ψ(θ) = exp.(A * θ_to_θstar(θ))
# ψ_to_θ(ψ) = θstar_to_θ(A \ log.(ψ))
```

Adapt the scaffold to avoid likelihood/MLE steps when no data are available; in that case use a nominal `θstar_hat` and clearly describe the analysis as a reference-point structural/numerical diagnostic.

## ODE / simulation model pattern

For ODE models, keep the right-hand side and solution extraction compatible with Dual-valued parameters.

```julia
using DifferentialEquations

function rhs!(du, u, θ, t)
    # no Float64 assumptions here
    du[1] = -θ[1] * u[1]
    du[2] =  θ[1] * u[1] - θ[2] * u[2]
    return nothing
end

function φ(θ)
    T = eltype(θ)
    u0_T = T.(u0)             # important for ForwardDiff
    prob = ODEProblem(rhs!, u0_T, tspan, θ)
    sol = solve(prob, Tsit5(); saveat=t_obs, abstol=1e-8, reltol=1e-8)
    Y = Array(sol)
    return vec(Y[observed_state_indices, :])
end
```

If the ODE solve fails with Dual-valued parameters, first check for type instabilities and hard-coded `Float64`. If time points or final times are treated as parameters, make `tspan` and `saveat` construction type-compatible as well, e.g. use `zero(T)` rather than `0.0` when the final time may be Dual-valued. If type fixes are not enough, consider whether the chosen solver, events, callbacks, or external simulation code are incompatible with automatic differentiation.

## Interpreting common outputs

Use manuscript-aligned terminology, but explain it accessibly.

- `S`: singular values of the transformed auxiliary-map Jacobian. Large separations suggest strong/weak information separation, but a scalar auxiliary mapping only reports one non-zero singular value.
- `rank_J`: numerical rank at the reference point.
- `N`: basis for the invariant null space in transformed coordinates. These are directions that appear to leave the auxiliary mapping unchanged under the invariance calculation.
- `N_perp`: canonical orthonormal complementary basis for the invariant image. This is used to construct the default reduction matrix `A_SVD = N_perp'`. In non-minimal cases, it may include non-invariant local-null directions as well as locally informed directions.
- Full local null space invariant: minimal image reparameterisation.
- Only part of the local null space invariant: non-minimal image reparameterisation.
- Rank deficient but `N` empty: local non-identifiability exists, but no invariant-null subspace was found for the chosen transformed coordinates.
- Full rank but very uneven singular values: structurally identifiable at this reference point, but some image coordinates may be weakly informed.

Avoid saying simply “these are the non-identifiable parameters.” Prefer “this is an invariant-null coordinate” or “this parameter combination is non-identifiable for the chosen auxiliary mapping.” Also avoid saying “invariant combination” without specifying whether it is an image coordinate or an invariant-null/free coordinate.

## Validation checklist

Before presenting final conclusions, check as many as practical:

- Does `φ*` pass the ForwardDiff preflight?
- Are singular values and ranks stable under reasonable tolerances?
- Does the result persist at nearby or alternative reference points?
- Are proposed sparse monomial coordinates actually close to the target subspaces? Inspect basis residuals and `basis_ok`.
- Do perturbations along invariant-null coordinates leave `φ` unchanged or nearly unchanged?
- If the Jacobian is rank deficient but `N` is empty, can a profile or level-set diagnostic show the local ridge and whether it curves in the chosen transformed coordinates?
- If a result appears only for a large finite time, saturated output, or limiting regime, describe it as numerical/approximate unless the limiting auxiliary mapping has been analysed directly.
- If using data, do profile likelihoods support the strong/weak or identifiable/non-identifiable interpretation?
- Does changing the observation operator change the result? If yes, report that clearly.
- If design variables were temporarily included as parameters, does the conclusion still hold after returning to the fixed-design problem?

## Profile likelihood and PWA guidance

Use profile likelihoods when the user asks about practical identifiability, uncertainty intervals, or prediction uncertainty.

- Reparameterise the likelihood in the chosen image coordinates.
- Treat selected image coordinates as interest parameters and optimise over the remaining nuisance parameters.
- For prediction uncertainty, push accepted/profile parameter values through the prediction map.
- Explain that PWA shows how variation along particular parameter combinations affects predictions; bands need not be uniform-width intervals around the maximum-likelihood prediction.

When using likelihood thresholds for accepted sets, be explicit about the degrees of freedom and why they were chosen. In IIR settings, the rank of the transformed auxiliary-map Jacobian may be relevant for accepted-set pushforwards, while lower-dimensional visualisations may use the dimension of the displayed interest plane.

## Reporting template

End with a concise, user-facing report.

```markdown
## IIR identifiability summary

Auxiliary mapping used:
- φ(θ) = ...
- Observed quantities / data distribution parameters: ...

Reference point:
- θ̂ = ...
- θ̂* = f(θ̂) = ...

Numerical result:
- rank(Dφ*) = ...
- local null dimension = ...
- invariant-null dimension = ...
- singular values = ...
- interpretation: minimal image / non-minimal image / rank-deficient with no invariant-null subspace / full-rank practical case

Selected coordinates:
| coordinate | expression | interpretation | evidence | caveat |
|---|---|---|---|---|
| ψ₁ | ... | image / strongly informed / retained | ... | ... |
| ψ₂ | ... | invariant-null / free / weakly informed | ... | ... |

Validation:
- ForwardDiff check: pass/fail
- tolerance/reference-point checks: ...
- profile/PWA checks if done: ...

Plain-language conclusion:
- ...
```

Keep caveats visible: IIR conclusions are for the chosen auxiliary mapping, transformations, parameter domain, reference point, and numerical tolerances.
