# Governance Templates for AI-Assisted Development

This directory contains governance artifacts that define how AI-assisted tools
must behave during software development. These templates apply the Open
Epistemic Protocol (OEP) and the Genesis Key Specification (GKS) as constraints
on reasoning, generation, and transformation.

These artifacts are intended for any project using AI to generate code,
documentation, or architectural components. They ensure that tools operate with
clear epistemic boundaries, deterministic structure, and enforceable constraints.

## Structure

- `templates/`  
  Reusable mission files and development profiles for IDEs, agents, and
  automation systems. These instruct AI tools on how to apply OEP/GKS during
  code generation.

## What These Templates Do

- enforce observability limits  
- prevent fabricated awareness or unverifiable claims  
- ensure consistent epistemic classification  
- require deterministic behavior  
- embed constraint-set discipline into the development flow  
- trigger lineage and structural checks when appropriate  
- guide tools toward safe transformation and away from generative ambiguity  

## How to Use

- Copy a template into your AI assistant or IDE agent configuration.  
- Reference the OEP and GKS repositories from within your development workflow.
- Modify templates only through documented governance processes (e.g., RFCs).
- Use these constraints as the default profile for model-driven development.

## Notes

These templates are intentionally minimal. They are expected to grow over time
as more enforcement strategies, validation tools, and pipeline stubs are added.
