# Guides for Coding Agents

This repository contains guides for coding agents.

The guides are used as grounded truth about certain services, APIs, or workflows so coding agents can implement features cleanly.

## How to Use

- They are not intended to be used directly, but rather use as context for planning agents to come up with engineering docs/plans.
- The way I use this is to have an agent read the guides and explore the codebase to come up with engineering docs/plans. Then execute on the plans.
- Ideally the docs contain ASCII diagrams to help human readers understand the flow of the system.

## Contents

- Currently we have guides for the wechat ecosystem, like wechat OAuth, wechat miniprogram logins, wechat pay, etc. The first-class online docs are hard for LLMs to read, as they contain lots of images in the original docs, and top LLM services (like OpenAI) are banned from accessing them. So we have to convert them to text format via other means.
  - see [wechat/](wechat/) for more details.
- I also have guides for other services, like Supabase, PostgreSQL, Apple Pay, Sign-in-with-Apple, I will add them gradually.

## Todo
- [ ] add README.md to each guide directory. e.g. a [wechat/README.md](wechat/README.md)
- [ ] add more guides for other services, like Supabase, PostgreSQL, Apple Pay, Sign-in-with-Apple, etc.

## Contributions

Contributions are welcome! Please feel free to submit a pull request to add new guides or improve existing ones.

Keep this in mind: 
1. the guides are mainly intended for coding agents to read, not for humans.
2. They should be able to propose a complete plan to implement a feature after reading the guides and exploring the codebase.
3. Ideally the docs should contain ASCII diagrams to help human readers understand the flow of the system.
