---
name: mixed-models-inference
description: >
  Design, fit, compare, report, and interpret confirmatory linear and generalized linear mixed-effects models in R. Use whenever a user mentions mixed models, multilevel or hierarchical models, random effects, mixed effects, random slopes, repeated measures, clustered data, nesting, crossed subjects and items, lmerTest, lme4, lmer, glmer, glmer.nb, ICC, or random-effects structures for inference. Apply it even when the user asks only for a formula or random-effects structure. Focus on lme4/lmerTest; the design logic also applies to glmmTMB and brms.
---

# Mixed models for inference

Use this workflow for explanatory or confirmatory inference, not primarily prediction. Translate the data-generating design into a defensible fixed- and random-effects structure; do not infer it from column names. For convergence, singularity, or identifiability problems, use `mixed-models-troubleshooting`.

## Start with the design interview

Before writing a formula, collect missing information. Do not proceed past an unresolved design question.

| Topic | Establish |
|---|---|
| Outcome | Variable, scale, and plausible distribution |
| Observation | What one row represents; timing and repetition |
| Research target | Confirmatory hypothesis/estimand and target population |
| Fixed effects | Exposure(s), covariates, interactions, and nonlinear terms that answer the question |
| Grouping factors | Clustering units and whether their levels support generalization beyond those observed |
| Structure | Whether factors are nested or crossed; observations per group/cell; sparse or empty cells |
| Variation | For each fixed-effect term, whether it varies within each grouping factor |
| Data quality | Missingness, outcome imbalance or separation, and sampling constraints |

A random grouping factor clusters rows and its levels represent a relevant broader population. A fixed categorical predictor represents specific contrasts of substantive interest.

## Four-step random-effects design workflow

### 1. Map the hierarchy

Identify each grouping factor and its relationship to the other grouping factors.

- **Nested:** each lower-level unit belongs to one higher-level unit, for example `(1 | school/classroom)` or `(1 | school) + (1 | school:classroom)`.
- **Crossed:** observations belong to multiple factors simultaneously, for example `(1 | participant) + (1 | item)`.

Do not treat IDs as nested merely because they are unique in the observed data. Confirm the sampling and identifier scheme with the user.

### 2. Classify variable levels

For every candidate fixed-effect term and grouping factor, determine whether the term varies within its groups.

- A constant term is between-group at that level.
- A varying term is within-group at that level.
- An interaction is eligible for a slope only when that interaction varies within the group. An interaction with a between-group term does not get a random slope.

Special care is needed for predictors that might vary both within- and between-groups:when a numeric predictor has both within- and between-group variation, a pooled coefficient can conflate distinct effects ("effect smooshing"). This typically happens when a low-level predictor is _measured_, but can also happen with manipulated predictors if the design is unbalanced (e.g., when some data filtering is affecting some cells more than others).

This can be checked with:

```r
performance::check_group_variation(
  dta_data,
  select = "predictor",
  by = "group"
)
```

Predictors with both within- and between-group variation need special handling: separate into distinct within- and between-group effects:

```r
dta_data <- datawizard::demean(
  dta_data,
  select = "predictor",
  by = "group"
)

mod_lmm <- lmerTest::lmer(
  outcome ~ predictor_within + predictor_between + (1 + predictor_within | group),
  data = dta_data
)
```

`predictor_within` is group-mean centered and `predictor_between` is the group mean. Retain the original predictor and make this transformation only when it serves the estimand.

### 3. Specify the structural hypothesis

Write fixed effects from the substantive question: prespecified main effects, interactions, nonlinearities, and confounder adjustment. Choose a GLMM family and link from the outcome-generating process, not coefficient convenience.

### 4. Derive random effects

For each design-justified grouping factor:

1. Include a random intercept for varying group baselines.
2. Add a random slope for every fixed-effect term that varies within that grouping factor.
3. Do not add a slope for a term constant within that grouping factor; it is non-identifiable there.

For a crossed participant-item experiment where `condition` varies within both factors:

```r
outcome ~ condition +
  (1 + condition | participant) +
  (1 + condition | item)
```

For a nested design when `condition` varies within `schools` but between `students`:

```r
outcome ~ condition +
  (1 + condition | schools) +
  (1 | schools:students)
```

For a Gaussian LMM with exactly one observation per crossed cell, an independent random cell intercept such as `(1 | participant:item)` is structurally confounded with homogeneous level-1 residual variance. This is also true if there is only one observation per within-subject condition-cell in general, making `(1 + condition | participant)` also confounded with homogeneous level-1 residual variance (another example: a fully factorial condition-by-time design with one observation per cell makes the `condition:time` random slope in `(1 + condition + time + condition:time | participant)` unidentifiable). This is not a convergence problem: obtain replication within cells and omit the confounded cell component if needed. (Separate participant and item intercepts are not automatically unidentifiable in such cases. Same for random slopes of "main effects" in a factorial or otherwise interactive design.)

## Model family and fitting function

Choose the outcome family after validating the design:

| Outcome | Function |
|---|---|
| Approximately Gaussian continuous outcome | `lmerTest::lmer()` |
| Binary or binomial outcome | `lme4::glmer(..., family = binomial)` |
| Poisson count | `lme4::glmer(..., family = poisson)` |
| Negative-binomial count | `lme4::glmer.nb()` |

Prefer `lmerTest::lmer()` to `lme4::lmer()` for LMM inference. It uses the same `lme4` framework and formula syntax while adding denominator degrees of freedom and p-values, including Satterthwaite and Kenward–Roger methods. The design logic transfers to `glmmTMB` and `brms`, which should be preferred for more complex models (e.g., zero-inflation, heteroscedasticity, additional families, etc.); mention/use them only when their features or Bayesian inference are needed.

## Validate before fitting

Always summarize the design in plain language and as a formula, then ask the user to confirm or correct it before fitting:

> I understand that each row is **[observation]**; **[factor A]** and **[factor B]** are **[nested/crossed]**; **[term]** varies within **[group]**, while **[term]** is between **[group]**. The confirmatory fixed-effects question is **[question]**. I therefore propose **[formula]**. Is that design correct before I fit it?

If the needed design information is unavailable, state what prevents a defensible decision and ask a targeted question.

## Fitting, reporting, and interpretation

### Baseline model

Start with the empty random-intercepts model justified by the hierarchy to describe baseline clustering. This does not select or justify the final random-effects structure, which is always justified by the data and design.

```r
mod_empty <- lmerTest::lmer(
  outcome ~ 1 + (1 | participant) + (1 | item),
  data = dta_data
)

res_icc <- performance::icc(mod_empty, by_group = TRUE)
```

Use `by_group = TRUE` where multiple grouping levels are present.

### Full model

Fit the full model with all planned fixed and random effects. For dealing with convergence and singularity issues, see the `r-mixed-models-troubleshooting` skill.

```r
mod_lmm <- lmerTest::lmer(
  outcome ~ condition * time +
    (1 + condition * time | participant) +
    (1 + condition | item),
  data = dta_data
)

tab_parameters <- parameters::model_parameters(
  mod_lmm,
  ci_method = "satterthwaite"
)
```

Use `parameters::model_parameters()` for clean summaries. For LMMs, use `ci_method = "satterthwaite"` (or documented shorthand `"s"`) or `ci_method = "kenward"`. `ci_random = TRUE` is available only when users specifically need confidence intervals for random effects; do not use it by default. For interactions, nonlinearities, or link-scale GLMM coefficients, define an estimand and use `marginaleffects` predictions, comparisons, or slopes on an interpretable scale.

Discourage standardized mixed-model coefficients by default. Level-specific variation, centering choices, link functions, and random slopes make standardization non-trivial and can obscure the estimand. Prefer naturally interpretable units or prespecified meaningful rescaling.

```r
res_r2 <- performance::r2(mod_lmm)
res_icc <- performance::icc(mod_lmm)
```

Use `performance::r2()` for model $R^2$. Both `performance::icc()` and `performance::r2()` accept `ci`, but intervals are typically bootstrap-based and slow; omit `ci` by default and add it only when requested or required by the reporting plan. For GLMMs, ICC and $R^2$ depend on distribution-specific variance approximations, so state the scale/approximation and interpret cautiously.

## Compare nested models

Use `anova()` for nested-model likelihood-ratio comparisons; it also reports AIC and BIC. Ensure models use identical rows and differ only in the terms being tested.

For LMMs, `lmer()` fits with `REML = TRUE` by default. When models differ only in random effects, use `anova(mod_reduced, mod_full, refit = FALSE)`. When they differ in fixed effects, use `anova(mod_reduced, mod_full)` so the default `refit = TRUE` refits with ML. Treat tests of variance components cautiously because zero is on the boundary of the parameter space.

## Sources

- Bolker, *GLMM FAQ*: https://bbolker.github.io/mixedmodels-misc/glmmFAQ.html
- Mattan S. Ben-Shachar, *Specifying Random Effects Structure for Hypothesis Testing or Estimation of Fixed Effects*: https://docs.google.com/presentation/d/1use1vdfqUpFssg70VfQl6U4ZaKdO9Jdi8F6fFqg4Xc8
- `performance::check_group_variation()`: https://easystats.github.io/performance/reference/check_group_variation.html
- `datawizard::demean()`: https://easystats.github.io/datawizard/reference/demean.html
