---
name: mixed-models-troubleshooting-r
description: >
  Diagnose and resolve convergence warnings, singular or boundary fits, non-positive-definite Hessians, optimizer failures, and identifiability problems in confirmatory lme4/lmerTest mixed models. Use whenever a user reports a mixed-model warning, asks whether to simplify random effects, needs help with maximal random-effects structures, or is fitting a sparse or fully crossed design. Preserve random intercepts and slopes justified by the design.
---

# Mixed-model troubleshooting

Use this skill after deriving the random-effects structure from the design. Do not cure these warnings by dropping design-justified random intercepts or slopes. For confirmatory inference, the design-supported maximal structure protects fixed-effect inference; see Barr et al. (2013).

## First classify the problem

Distinguish these before changing a model:

- **Structural non-identifiability:** the design cannot separate parameters. In a Gaussian fully crossed design with one observation per cell, `(1 | participant:item)` or the `condition:time` random slope in `(1 + condition + time + condition:time | participant)` is confounded with residual variance. Optimizer changes cannot solve this.
- **Singular/boundary fit:** the likelihood optimum has a random-effects variance-covariance matrix that is only positive semi-definite: at least one variance direction is effectively zero. This can arise from a zero variance, a correlation at a boundary, or a more complex linear combination that is not obvious in `VarCorr()`. It is not automatically a failed convergence.
- **Numerical non-convergence:** the optimizer or derivative checks did not establish that the algorithm reached a stable likelihood optimum. It can produce unreliable or non-estimable estimates and standard errors, but an `lme4` warning can also be overly conservative.
- **Data/model failure:** coding errors, separation, empty cells, wrong outcome family, severe scaling problems, or a proposed slope that does not vary within its grouping factor.

## Diagnose singularity

A singular model is statistically defined, but may indicate an over-complex covariance structure and weaken ordinary Wald or likelihood-ratio inference. Do not equate it with structural non-identifiability: the latter is determined by the design before fitting.

Rather than relying solely on printed variances or correlations use:

```r
performance::check_singularity(mod_lmm)
```

It returns `TRUE` for a singular fit. Its default `tolerance = 1e-5` controls how close to a boundary is considered singular; a larger tolerance is stricter. Inspect `VarCorr(mod_lmm)` as a complement, not a substitute. For `glmmTMB` models, `check = "terms"` can identify affected random-effects terms.

When inference is required for a singular fit, treat random-effect estimates and uncertainty cautiously. Ordinary Wald standard errors may be fragile. For fixed effects, profile-likelihood intervals are one alternative (can be slow):

```r
parameters::model_parameters(
  mod_lmm,
  effects = "fixed",
  ci_method = "profile"
)
```

Appropriate bootstrap-based uncertainty assessment is another option. Do not claim that a singular fit automatically leaves fixed-effect inference unaffected. For sparse or weakly identified estimation, consider `brms` with meaningful weakly informative priors.

## Diagnose and investigate convergence

Alongside the original `lme4` warning, use:

```r
res_converged <- performance::check_convergence(mod_lmm)
```

It returns `TRUE` when its alternative check finds convergence acceptable and `FALSE` when it is suspicious; its convergence value is returned as an attribute. The default `tolerance = 0.001` can be made stricter by making it smaller.

Work in this order:

### 1. Verify design and model spec

First verify the design, nesting/crossing, contrasts, outcome coding, missingness, empty cells, family/link, offsets, separation issues, and within-group variation for every slope, and the distinction from a structurally confounded term. See the `r-mixed-models-inference` skill for guidance.

If a component is truly non-identifiable from the design, explain that structural reason and confirm any revised model with the user.

### 2. Optimizer

Increase the optimizer iteration budget and try a suitable optimizer, for example `lme4::lmerControl(optimizer = "bobyqa")` or `lme4::glmerControl(optimizer = "bobyqa")`.

If changing an optimizer does not work, use `lme4::allFit()` to refit a fitted `lmer` or `glmer` model with its available optimizers:

```r
fits_all <- lme4::allFit(mod_lmm, maxfun = 1e5)
summary(fits_all)
```

Inspect `which.OK`, log-likelihoods, fixed effects, and random-effect standard deviations/correlations from the summary. If all optimizers yield essentially the same likelihood and substantively equivalent estimates and uncertainty, the original warning is less concerning; material disagreement means the fit remains unresolved. `allFit()` needs `optimx` and `dfoptim` to use every available optimizer. It translates `maxfun` to optimizer-specific controls, but that setting is only respected by some methods. It refits through `update()`, so the original call's data and referenced objects must remain available. Do not use parallel fitting by default.

Here too, a Bayesian model can go a long way - suggest switching to `rstanarm::stan_glmer()` or `brms::brm()` with meaningful weakly informative priors.

### 3. Change feature specification

#### Centering and recoding

Center or rescale continuous predictors and sum-to-zero code factors where this preserves the target estimand (design-equivalent reparameterization); document changes to intercept and interaction interpretation.

```r
# Center a continuous predictor
dat$trial_c <- datawizard::center(dat$trial)

# Center and scale if needed
dat$trial_cs <- datawizard::standardize(dat$trial)

# Sum-to-zero code the factor
contrasts(dat$condition) <- contr.sum

mod_centered <- lme4::lmer(
  response ~ condition * trial_c + (1 + condition + trial_c | participant),
  data = dat
)
```

#### Remove random covariances

Preserve the design-supported intercepts and slopes while reducing covariance parameters, for example `(1 + condition || participant)` or the more general `diag(1 + condition | participant)` (works when `condition` is a factor). Do not automatically delete a design-justified slope - only revise a term after establishing that it is structurally non-identifiable or after a documented, substantive decision (option 1 above).

Using `||` is the same same as fitting separate random effects, which results in no covariance between them.

```
response ~ condition * trial_c +
  (1 + trial_c || participant) +
  (1 + condition | item),
```

is the same as:

```
response ~ condition * trial_c +
  (1 | participant) +
  (0 + trial_c | participant) +
  (1 + condition | item),
```

This is only appropriate for numeric slopes - when used with a factor slope it gives per-category intercepts (R's standard `0 + factor` syntax). Instead, the more general `diag()` should be used:

```
response ~ condition * trial_c +
  diag(1 + condition | participant) +
  (1 + condition | item),
```

#### Complex random intercepts

Complex random intercepts (CRIs) can reduce computational burden, but random effects are less directly interpretable. This option is expecially useful with factorial designs or designs with random slopes for factors.

For example, we have `X` (a continuous predictor) and `condition` (a factor) - both within participants. This standard specification:

```
response ~ X * condition + (1 + X * condition | participant)
```

Can be expanded to:

```
response ~ X * condition +
  (1 | participant) +
  (0 + X | participant) +
  (0 + condition | participant) +
  (0 + X:condition | participant)
```

Which can then be converged to CRIs:

```
response ~ X * condition +
  (1 + X | participant) +
  (1 + X | participant:condition)
```

Keep the same CRI parameterization across compared models because it changes degrees of freedom and model structure.

### 4. Other

Turning off derivative calculations or convergence checks, for example `lme4::lmerControl(calc.derivs = FALSE)` (only for LMMs), can help diagnose which check produced a warning. It does not show that the model converged and is not a remedy.

## Sources

- Barr, D. J., Levy, R., Scheepers, C., & Tily, H. J. (2013). *Random effects structure for confirmatory hypothesis testing: Keep it maximal.* Journal of Memory and Language, 68(3), 255–278. https://doi.org/10.1016/j.jml.2012.11.001
- Scandola, M., & Tidoni, E. (2024). *Reliability and feasibility of linear mixed models in fully crossed experimental designs.* Advances in Methods and Practices in Psychological Science, 7(1), 25152459231214454. https://doi.org/10.1177/25152459231214454
- Bolker, *GLMM FAQ*: https://bbolker.github.io/mixedmodels-misc/glmmFAQ.html
- `performance::check_singularity()`: https://easystats.github.io/performance/reference/check_singularity.html
- `performance::check_convergence()`: https://easystats.github.io/performance/reference/check_convergence.html
- `lme4::allFit()`: https://lme4.github.io/lme4/reference/allFit.html
- Ben-Shachar, convergence and singularity examples: https://github.com/mattansb/Hierarchical-Linear-Models-foR-Psychologists/blob/main/09%20misc/02%20convergance.R
