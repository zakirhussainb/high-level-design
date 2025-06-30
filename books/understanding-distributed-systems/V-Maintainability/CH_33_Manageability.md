---
documentclass: extarticle
fontsize: 14pt
linestretch: 1.5
geometry: "margin=0.8in"
header-includes: |
  \usepackage[T1]{fontenc}
  \usepackage{mlmodern}
  \usepackage{float}
  \newenvironment{centerfigure}{\begin{figure}[H]\centering}{\end{figure}}
  \usepackage{caption}
  \captionsetup{font=small}
  \usepackage{etoolbox}
  \AtBeginEnvironment{tabular}{\footnotesize}
  \AtBeginEnvironment{longtable}{\footnotesize}
---

# Chapter 33: Manageability

Manageability refers to the need for operators to modify an application's behavior without requiring code changes. While observability tools help operators understand behavior, manageability provides the mechanisms to change it, most commonly through configuration.

## Application Configuration

Applications rely on various configuration settings that can be categorized as:

- **Behavioral Settings**: These affect how the application functions, such as the maximum size of an internal cache.
- **Secrets**: These include sensitive information like credentials needed to access external data stores.

### Decoupling Configuration from Code

It is critical that settings are **not hardcoded** into the application. This is because settings often vary between different environments (e.g., development, production) and can contain sensitive data.

To decouple the application from its configuration, settings can be stored in a dedicated, external configuration store like **AWS AppConfig** or **Azure App Configuration**.

### Configuration Update Strategies

1.  **Static Configuration (At Deployment)**:

    - In this approach, the Continuous Delivery (CD) pipeline reads the configuration from the store during deployment.
    - The settings are then passed to the application, often through environment variables.
    - **Drawback**: Any change to the configuration requires a full redeployment of the application.

2.  **Dynamic Configuration (At Run Time)**:
    - For an application to react to configuration changes without being redeployed, it must periodically re-read its configuration during run time.
    - When a setting is updated in the store, the application detects the change and applies it.
    - _Example_: If an HTTP handler's behavior depends on a specific setting, the handler must be re-created when that setting changes.

### Benefits of Dynamic Configuration

Once the ability to change settings dynamically is in place, it enables powerful release strategies:

- **Feature Toggles (or Feature Flags)**:

  - New features can be deployed to production in a disabled state by default.
  - This allows the code to be integrated and released safely without affecting users.

- **Progressive Rollouts & Canary Releases**:

  - After deployment, a feature can be enabled for a small fraction of application instances or users.
  - This helps build confidence that the feature works as intended in the production environment before it is rolled out to everyone.

- **A/B Testing**:
  - The same dynamic configuration mechanism can be used to perform A/B tests by showing different versions of a feature to different user segments and measuring the impact.
