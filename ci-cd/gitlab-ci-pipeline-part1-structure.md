# Structuring a Clean GitLab CI Pipeline for Dev, QA, Stage, and Production (Part 1: Structure & Rules)

Most GitLab CI pipelines don't start out messy. They start out fine — a `.gitlab-ci.yml` with three or four jobs, a couple of stages, maybe a `deploy` job that pushes straight to production because there was only one environment to worry about. Then QA gets added. Then staging. Then someone asks for manual approval gates before prod, and six months later you've got a 400-line YAML file held together by `only`/`except` rules nobody fully remembers writing.

This is the first of two parts on avoiding that version of the file. Part 1 covers the structural decisions — how environments relate to stages, building once versus rebuilding per environment, and templating jobs so four environments don't mean four copies of the same logic. [Part 2](./gitlab-ci-pipeline-part2-security-rollback.md) covers protected environments, environment-scoped secrets, and treating rollback as a first-class pipeline job.

## The core problem: environments aren't stages

The most common mistake is treating `dev`, `qa`, `stage`, and `production` as **stages** in the pipeline sense. They're not. A stage in GitLab CI (`build`, `test`, `deploy`) describes a phase of work. An environment describes *where* that work lands. Conflating the two is how you end up with a `deploy-to-qa` stage, a `deploy-to-stage` stage, and a `deploy-to-prod` stage sitting side by side — each duplicating the same deploy logic with a different target baked in.

The cleaner mental model:

```
stages:
  - build
  - test
  - deploy

environments:
  dev  → auto-deploy on every merge to develop
  qa   → auto-deploy on every merge to develop (same artifact, different target)
  stage → manual promotion of a tagged/release-candidate artifact
  prod  → manual promotion, protected, requires approval
```

One `deploy` stage, one `deploy` job template, parameterized by environment. Everything else is `rules:` controlling *when* each environment's instance of that job runs.

## Build once, deploy everywhere

If your pipeline rebuilds the application for every environment, that's the second thing to fix before touching the YAML structure. You want exactly one build artifact (a container image, a compiled binary, a packaged bundle) that gets promoted through environments unchanged. Rebuilding per environment means you're no longer testing the thing that ships to prod — you're testing a sibling of it.

```yaml
build:
  stage: build
  script:
    - docker build -t "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA" .
    - docker push "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA"
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "develop"'
    - if: '$CI_COMMIT_TAG'
```

The tag `$CI_COMMIT_SHORT_SHA` becomes the single identifier that travels from dev through prod. When someone asks "is this the same build that passed QA," the answer should be a diff of one string, not an archaeology project.

## A template-first job structure

GitLab's `extends` keyword is the single biggest lever for keeping a multi-environment pipeline readable. Define the shape of a deploy job once, then extend it per environment with only the parts that differ.

```yaml
.deploy_template:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl set image deployment/$APP_NAME
        $APP_NAME=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
        --namespace=$K8S_NAMESPACE
    - kubectl rollout status deployment/$APP_NAME --namespace=$K8S_NAMESPACE

deploy-dev:
  extends: .deploy_template
  environment:
    name: dev
    url: https://dev.example.com
  variables:
    K8S_NAMESPACE: dev
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'

deploy-qa:
  extends: .deploy_template
  environment:
    name: qa
    url: https://qa.example.com
  variables:
    K8S_NAMESPACE: qa
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'

deploy-stage:
  extends: .deploy_template
  environment:
    name: stage
    url: https://stage.example.com
  variables:
    K8S_NAMESPACE: stage
  rules:
    - if: '$CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+-rc/'
      when: manual

deploy-prod:
  extends: .deploy_template
  environment:
    name: production
    url: https://example.com
  variables:
    K8S_NAMESPACE: production
  rules:
    - if: '$CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/'
      when: manual
```

Notice what's different across the four jobs: the namespace, the environment URL, and the rule governing when they're eligible to run. Everything else — the actual deploy mechanics — lives in one place. When your deploy logic changes (say, you move from `kubectl set image` to a Helm upgrade), you change it once.

## Rules, not only/except

If your pipeline still uses `only:`/`except:`, that's worth migrating away from independent of the environment question. `rules:` gives you ordered, composable conditions and — more importantly — lets you attach `when: manual`, `when: on_success`, and `allow_failure` per condition instead of bolting them onto the job as a whole.

The pattern worth internalizing: **auto-deploy pre-production, gate production behind an explicit human action.** Dev and QA should update on every merge without anyone touching a button — that's the point of having them. Stage and prod should require someone to look at what's about to happen and press go.

| Environment | Trigger | Approval | Typical audience |
|---|---|---|---|
| dev | merge to `develop` | none | engineers, fast feedback |
| qa | merge to `develop` | none | QA team, automated test suites |
| stage | release-candidate tag | manual | QA sign-off, stakeholder demos |
| production | release tag | manual, protected | on-call, release manager |

That last column — protected — is where Part 1 hands off to Part 2. Marking a job `when: manual` stops accidental deploys; it doesn't stop *unauthorized* ones. Locking that down properly means pairing job-level rules with GitLab's protected environments and protected tags, which is where we pick up next.

---

*Continue to [Part 2: Security & Rollback](./gitlab-ci-pipeline-part2-security-rollback.md). Part of the [DevOps Field Notes](https://github.com/sandeepk24/devops-field-notes) series.*
