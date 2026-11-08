# Mixed Models Skills

Agent skills for designing, fitting, interpreting, and troubleshooting confirmatory mixed-effects models in R.

The skills focus on `lme4` and `lmerTest` (but are also compatible with `glmmTMB`, and `rstanarm`/`brms`) and `easystats` packages such as [`performance`](https://easystats.github.io/performance/) and [`parameters`](https://easystats.github.io/parameters/).

## Skills

### Mixed models for inference

`mixed-models-inference` supports confirmatory linear and generalized linear mixed-effects modeling. It guides a design-first workflow: establish what each observation represents, distinguish nested from crossed grouping factors, determine which predictors vary within groups, and derive defensible fixed and random effects before fitting. It also covers model family selection, fitting with `lme4`/`lmerTest`, reporting ICC and $R^2$, and comparing nested models.

### Mixed-model troubleshooting

`mixed-models-troubleshooting` helps diagnose convergence warnings, singular or boundary fits, non-positive-definite Hessians, optimizer failures, and structural non-identifiability. It distinguishes problems the design cannot resolve from numerical issues, preserves design-justified random effects, and provides a staged workflow for checking specifications, trying optimizers, and using equivalent reparameterizations when appropriate.

## Install

Install both skills with the [Skills CLI](https://skills.sh):

```bash
npx skills add mattansb/mixed-models-skills
```
