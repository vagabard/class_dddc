Here is the technical specification document detailing the necessary updates, structural conflicts, and numerical mitigation strategies required to force the CLASS Einstein-Boltzmann solver to accommodate Density Driven Dilation Cosmology (DDDC).

***

# CLASS Modification Guide: DDDC Compliance & Soft Transitions

### PART 1: The Core Architectural Conflict
CLASS is fundamentally hardcoded around the Friedmann-Lemaître-Robertson-Walker (FLRW) metric. The most critical divergence between FLRW and DDDC occurs at the origin of the universe.

* **FLRW Assumption:** The universe begins at a mathematical singularity. The scale factor $a \to 0$, and CLASS routinely initializes its background integrations at deeply microscopic scale factors (e.g., $a \approx 10^{-14}$).
* **DDDC Assumption:** The universe has a hard physical floor based on the total mass $M_0$. Time dilation is governed by the Schwarzschild interior metric, hitting infinite dilation ($D \to \infty$) at the Buchdahl limit: $a_{min} = \frac{9}{8} a_{rs}$.
* **The Failure State:** If CLASS attempts to evaluate the DDDC background equations at any $a < a_{min}$, the square root in the dilation denominator $\left(\sqrt{1 - \frac{a_{rs}}{a}}\right)$ will produce imaginary numbers, and the denominator itself will cross zero, causing fatal `NaN` (Not a Number) cascade failures in the C-code integrators.

To allow CLASS to compute initial conditions (like Big Bang Nucleosynthesis and the initial perturbation spectrum) without crashing, a **Soft Transition** must be engineered.

---

### PART 2: The Soft Transition Engine

To satisfy CLASS's requirement to integrate back to $a \to 0$ while preserving DDDC's mechanics for the observable universe, the code must dynamically blend the two frameworks during the deep radiation epoch.

**1. The Transition Threshold ($a_{trans}$)**
Define a scale factor $a_{trans}$, comfortably larger than $a_{min}$ but deep enough in the past that it occurs before any relevant observational benchmarks (e.g., before recombination at $z \approx 1100$).

**2. The Blending Function ($S(a)$)**
Use a smooth, continuously differentiable sigmoid function to transition the physics. This prevents sharp discontinuities in the derivatives, which would crash CLASS's internal `ndf15` numerical integrator.
$$S(a) = \frac{1}{1 + e^{-k_{blend}(a - a_{trans})}}$$
* For $a \gg a_{trans}$, $S(a) \to 1$ (Pure DDDC).
* For $a \ll a_{trans}$, $S(a) \to 0$ (Pure FLRW).
* $k_{blend}$ controls the sharpness of the transition.

**3. The Effective Dilation Factor ($D_{eff}$)**
In `background.c`, rewrite the dilation function to output a blended metric:
$$D_{eff}(a) = S(a) \cdot D_{DDDC}(a) + (1 - S(a)) \cdot 1.0$$
*(Note: $1.0$ is the standard FLRW undilated baseline).*

**4. The Effective Spatial Expansion**
If your scale factor equation $a(t_c) = k_{dil} t_c^n + a_{min}$ strictly forbids $a \to 0$, you must also blend the spatial expansion rate $H_{spatial}$ back into standard radiation-dominated FLRW scaling ($H \propto a^{-2}$) below $a_{trans}$.

---

### PART 3: High-Risk CLASS Modules & Required Updates

Beyond the background engine, several specific modules in CLASS will error out or produce unphysical results if not explicitly guided through the DDDC framework.

#### A. Thermodynamics (`thermodynamics.c`)
* **The Error:** CLASS uses the physical cooling of the primordial plasma to trigger recombination (the decoupling of photons and baryons). If $D(a) \to \infty$, the Hubble friction term becomes infinite, freezing the thermodynamics and preventing recombination from ever completing in the code.
* **The Fix:** The soft transition $D_{eff}(a)$ must fully resolve to $1.0$ before the code integrates forward to the thermodynamics initialization. Ensure that physical temperature is calculated as $T = T_{cmb} \cdot (1 + Z_{total})$, but bound $Z_{dilation}$ so it does not exceed the transition threshold.

#### B. Perturbations & Tight Coupling (`perturbations.c`)
* **The Error:** In the early universe, photons and baryons are tightly coupled. CLASS uses a specialized "Tight Coupling Approximation" (TCA) to avoid taking infinitely small integration steps. TCA relies heavily on conformal time ($\eta$). If $\eta$ stalls because $d\eta = \frac{c \cdot D(a) dt_c}{a}$, the TCA triggers will misfire, causing perturbation integrations to diverge.
* **The Fix:** The initial conditions for perturbations must be set in the FLRW regime (where $S(a) \to 0$). The transition to DDDC ($S(a) \to 1$) must occur *after* the initial perturbation modes have entered the horizon, but before the epoch of observation.

#### C. Nonlinear Structure Formation (`nonlinear.c` / Halofit)
* **The Error:** CLASS uses semi-empirical fitting functions (like Halofit or HMcode) to calculate the non-linear matter power spectrum. These are heavily calibrated against FLRW N-body simulations. They rely purely on standard spatial $a$ and standard $\Omega_m$.
* **The Fix:** Halofit will not inherently understand DDDC. You must feed the non-linear module the "effective" FLRW scale factor derived from your $Z_{total}$ equation. If you pass the raw DDDC $a$, the non-linear collapse thresholds will trigger at the wrong physical epochs.

---

### PART 4: C-Code Implementation Targets for Soft Transition

When writing the C code, target these specific parameters to enforce the transition logic safely.

**1. `background_init()` (File: `background.c`)**
* **Intercept:** Before establishing the `a` array, enforce the boundary checks.
* **Logic:** ```c
  if (pba->a_ini < pba->a_min) {
  // Force soft transition logic to initialize FLRW background
  pba->use_soft_transition = _TRUE_;
  }
  ```

**2. `background_solve()` (File: `background.c`)**
* **Intercept:** The root-finding algorithm converting $z$ to $a$.
* **Logic:** If the requested $z$ corresponds to an epoch where $a < a_{trans}$, the root finder must gracefully swap from the combined $Z_{total}$ equation back to the simple $1/a - 1$ equation to avoid sending the solver into the complex plane.

**3. `background_derivs()` (File: `background.c`)**
* **Intercept:** The differential equations defining $H$ and $d\tau$.
* **Logic:** Apply the sigmoid blending function here so the numerical integrator sees a smooth, continuous derivative vector across the transition boundary.

By implementing this soft transition, you preserve the strict, strong-field DDDC mechanics for the late universe and observable data (Supernovae, BAO), while giving CLASS the FLRW-like mathematical sandbox it absolutely requires to initialize the Big Bang.