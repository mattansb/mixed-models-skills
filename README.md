# Mixed Models Skills

Agent skills for designing, fitting, interpreting, and troubleshooting confirmatory mixed-effects models in R.

The skills focus on fitting mixed models with `{lme4}` (and `lmerTest`), but are also compatible with `{glmmTMB}`, and `{rstanarm}`/`{brms}`.

## Skills

See examples of the skills in use [here](examples.md).

### Mixed models for inference

[`mixed-models-inference`](mixed-models-inference-r/SKILL.md) supports confirmatory linear and generalized linear mixed-effects modeling. It guides a design-first workflow: establish what each observation represents, distinguish nested from crossed grouping factors, determine which predictors vary within and between groups, and derive defensible fixed and random effects specifications. It also covers model family selection, fitting with `lme4`/`lmerTest`, and comparing nested models. This skill also supports best practices for reporting ICC and $R^2$ (with [`{performance}`](https://easystats.github.io/performance/)) and producing coefficient tables (with [`{parameters}`](https://easystats.github.io/parameters/)).

### Mixed-model troubleshooting

[`mixed-models-troubleshooting`](mixed-models-troubleshooting-r/SKILL.md) helps diagnose the most commonly occurring warnings:
_convergence_ (`Model failed to converge ...`) and _singular fit_ (`boundary (singular) fit: see help('isSingular')`). It distinguishes problems the design cannot resolve from numerical issues, and provides a staged workflow that preserves design-justified random effects.

## Install

Install both skills with the [Skills CLI](https://skills.sh):

```bash
npx skills add mattansb/mixed-models-skills
```
