---
title: "Automating Development Environment with Mise: Comprehensive Guide 💫"
seoTitle: "Automate Dev Environments with Mise Guide"
seoDescription: "Automate consistent dev environments with Mise, a unified tool and task management app. Simplify installations, tasks, and pipelines with ease"
datePublished: Tue Oct 29 2024 09:06:42 GMT+0000 (Coordinated Universal Time)
cuid: cm2u84ec9000d08mbfb74ecrr
slug: dev-env-with-mise
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1730150532839/428f7559-7364-4c3e-9228-f439b6c22a0c.png
tags: tools, devops, makefile, developer-tools, cicd, development-environments

---

When working in a team, it's important for everyone to have a consistent environment. Also, different projects might require different versions of tools.

Additionally, automating routine tasks with the codebase is helpful, so some form of task management is also necessary.

## Background

As always, there are many tools available to solve these problems. For example, to create a development environment, there are:

1. **Language-specific tools**
    
    1. Node.js: nvm / n
        
    2. Python: venv / pyenv / poetry / conda
        
    3. Ruby: rbenv
        
    4. Java: jenv
        
    5. Infrastructure tools:
        
        1. tfenv for Terraform
            
        2. kbenv for kubectl
            
        3. helmenv for Helm
            
    6. Environment variables: [direnv](https://github.com/direnv/direnv)
        
2. **Language-agnostic tools**
    
    1. [asdf](https://asdf-vm.com) - popular all-in-one solution
        
    2. [nix](https://nix.dev) - it’s better to have a lot of free time, lol
        
    3. or even [devcontainers](https://code.visualstudio.com/docs/devcontainers/tutorial) for VS Code: the docker-way
        

As we can see, there are many tools available 🤯, and there are even more for task management. While we can use `make`, it's not very user-friendly. So, there are many alternatives to `make`. Trust me, I've tried most of them

1. [task](https://taskfile.dev)
    
2. [just](https://github.com/casey/just)
    
3. [Earthly](https://github.com/earthly/earthly) - docker inside 🤟
    
4. [bake](https://github.com/trinio-labs/bake)
    
5. [run](https://github.com/TekWizely/run) - last commit year ago
    
6. [**makesure**](https://github.com/xonixx/makesure) - weird syntax
    
7. [foy](https://github.com/zaaack/foy) - if we love js
    
8. [mmake](https://github.com/tj/mmake) - `make` on steroids
    
9. [robo](https://github.com/tj/robo) - last commit 2 years ago
    
10. [redo](https://github.com/apenwarr/redo) - last commit 3 years ago
    
11. Or even [buck2](https://buck2.build) / [bazel](https://github.com/bazelbuild/bazel) / [pants](https://github.com/pantsbuild/pants) / [please](https://please.build) if we already have it in the project
    

I actually really like the first three.

## Introducing Mise: Development Tools and Tasks in One App

[Mise](https://mise.jdx.dev), inspired by [asdf](https://asdf-vm.com) — the multiple runtime version manager, can use asdf’s repository with hundreds of tools as its successor. However, mise goes further:

1. It supports several other "backends" to install tools outside the built-in repository.
    
2. It has a simpler CLI.
    
3. Tasks!
    

### Define a Toolset

Mise is set up using a single file called `.mise.toml`. Here's an example:

```ini
# .mise.toml
[tools]
terraform = "1.9"
terramate = "0.9"
pre-commit = "3"
awscli = "2"
"pipx:detect-secrets" = "1.4"
"go:github.com/containerscrew/tftools" = "0.9.0"
```

I can install it with a single command: `mise install`, see a demo gif in the next sections. Let me explain the contents.

Our team uses this file for the infrastructure repository with Terraform and [Terramate](https://github.com/terramate-io/terramate) manifests. Terraform requires awscli to work with the AWS API provider.

`detect-secrets` is a tool used in a [pre-commit hook](https://github.com/pre-commit/pre-commit) to ensure no secrets are accidentally exposed to git. This tool is installed with `pipx`, and mise supports this backend out of the box.

`tftools` is a tool that summarizes changes in Terraform plans. It's useful if you want to review plans for several environments or stacks at once. As we can see, it uses the Go backend to fetch a binary from the repository.

There are many other backends: [asdf](https://mise.jdx.dev/dev-tools/backends/asdf.html), [cargo](https://mise.jdx.dev/dev-tools/backends/cargo.html), [go](https://mise.jdx.dev/dev-tools/backends/go.html), [npm](https://mise.jdx.dev/dev-tools/backends/npm.html), [pipx](https://mise.jdx.dev/dev-tools/backends/pipx.html), [spm](https://mise.jdx.dev/dev-tools/backends/spm.html), [ubi](https://mise.jdx.dev/dev-tools/backends/ubi.html), [vfox](https://mise.jdx.dev/dev-tools/backends/vfox.html). Honestly, I'm not sure what spm and vfox are 🙂, but ubi is [The Universal Binary Installer](https://github.com/houseabsolute/ubi), which lets you install any binary from a tool’s GitHub release page.

Need to add more tools? Edit the file or run `mise use TOOL_NAME`. To see the built-in tool registry, run `mise registry`. For tools not listed there, we can use the full notation, as I did with *tftools* above.

### **Manage Environments**

By default, mise installs a tool **not system-wide**. This means we can have different tool versions for different directories aka projects. If you prefer a system-wide installation, you can configure it by placing the settings in `~/.config/mise/config.toml`, making tools and tasks (see below) available in any directory.

Mise supports nested configurations. To find out which configuration provides a tool or task, run `mise ls`.

For example, you can have different Python versions for different projects. Mise determines which Python version to use based on the nearest `.mise.toml` file. It even supports [automatic virtualenv activation](https://mise.jdx.dev/lang/python.html#automatic-virtualenv-activation).

You can also set different environment variables for different directories, similar to [direnv](https://github.com/direnv/direnv).

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1730137795481/33c39289-c7bf-4bf4-986b-804e6bd41f35.gif align="center")

### Prepare Dev Env

1. `brew install mise` or [use alternative installation methods](https://mise.jdx.dev/getting-started.html#alternate-installation-methods)
    
2. [Activate mise](https://mise.jdx.dev/getting-started.html#_2a-activate-mise) in your shell. Zsh example:
    
    ```bash
    echo 'eval "$(~/.local/bin/mise activate zsh)"' >> ~/.zshrc
    # restart shell or `source ~/.zshrc`
    ```
    
3. `mise install` in a dir with `.mise.toml` file  
    or use my [VSCode extension](https://marketplace.visualstudio.com/items?itemName=rgeraskin.mise)
    

So, add this simple instruction to the repo's `README.md`, place `.mise.toml` in the root of the repo, and your teammates can simply run `mise install` to get the same toolset. Make sure to pin tool versions to ensure consistency. 🙂

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1730192549906/91e81498-e018-4a3b-bbcd-88fe83ffdbea.gif align="center")

Also, mise is cross-platform, so people using Linux or Mac will follow the same instructions. No more juggling with brew, apt, or yum.

### Run Tasks

Let's define several tasks to make daily routines easier with that infrastructure repo. To run Terraform, `*.tf` files should be generated with Terramate. To create a plan, the *tf-state* must be initialized. To initialize a state, it's best to be logged into AWS. That's a lot to keep track of.

Here's a quick demo:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1730140976742/282babf6-8269-4dce-90d9-9952cc0e9456.gif align="center")

We can use mise tasks to make the process easier. I've removed the Terramate and SSO details to keep the example brief:

```ini
# .mise.toml

[tasks."tf.init"]
alias = "tfi"
description = "`terraform init`"
run = "terraform init"

[tasks."tf.validate"]
alias = "tfv"
depends = ["tf.init"]
description = "`terraform validate`"
run = "terraform validate"

[tasks."tf.plan"]
alias = "tfp"
depends = ["tf.init"]
description = "`terraform plan`"
run = """
#!/usr/bin/env bash

# can use args for this task
terraform plan -out=tfplan $@ 
# ring terminal when complete
tput bel 
"""

[tasks."tf.summarize"]
alias = "tfs"
description = "`tftools summarize` with pre-generated plan file (terraform plan should be generated in advance)"
run = """
#!/usr/bin/env bash

terraform show -json tfplan > plan.json
tftools summarize < plan.json
"""

[tasks."tf.apply"]
alias = "tfa"
description = "`terraform apply` with pre-generated plan file (terraform plan should be generated in advance)"
run = """
#!/usr/bin/env bash

cmd="terraform apply tfplan $@"
read -p "$cmd\n\nAre you sure? (Y/n) " choice
[ "$choice" = "n" ] || ($cmd; tput bel)
"""

[tasks."tf.lock"]
depends = ["tf.init"]
description = "`terraform providers lock` with predefined platform list"
run = "terraform providers lock -platform=linux_amd64 -platform=darwin_amd64 -platform=darwin_arm64 ; tput bel"

[tasks."tf.console"]
alias = "tfc"
depends = ["tf.init"]
description = "`terraform console`"
raw = true
run = "terraform console"
```

To run *plan*, use `mise run tf.plan`. Mise has aliases, so we can use `mise run tfp`. The shell also supports aliases 🙂. So why not create an alias with `alias mr="mise run"` and just use `mr tfp`.

> If you use VSCode, check out my [VSCode extension](https://marketplace.visualstudio.com/items?itemName=rgeraskin.mise) to run tasks for your workspace directly from the IDE.

This is a simple, straightforward flow for Terraform. To see the full example I use daily, you can check [this gist](https://gist.github.com/rgeraskin/88c895e393aa8727464401980482f4e0).

Clean and beautiful syntax, in my opinion.

> Makefile is for C developers, while TOML is for humans 😊

List all available tasks, and you'll get a nice table with descriptions:

```bash
$ mise tasks

Name          Description                               Source
tf.apply      `terraform apply` with pre-generated pl…  ~/.config/mise/config.toml
tf.console    `terraform console`                       ~/.config/mise/config.toml
tf.init       `terraform init`                          ~/.config/mise/config.toml
tf.lock       `terraform providers lock` with predefi…  ~/.config/mise/config.toml
tf.plan       `terraform plan`                          ~/.config/mise/config.toml
tf.summarize  `tftools summarize` with pre-generated …  ~/.config/mise/config.toml
tf.validate   `terraform validate`                      ~/.config/mise/config.toml
```

### CI/CD Pipelines

Tired of writing boilerplate code to install necessary tools for pipelines? With `.mise.toml`, why not use `mise install` in pipelines too? Install mise in advance or use the Docker image `jdxcode/mise`.

There's no need to track tool versions separately for development environments and CI anymore. You no longer need to remember to update tool versions in different places to prevent any issues. Mise serves as a single source of truth.

If you use a similar task flow in CI as you do locally, you can run the same mise tasks there too. If not, you can use [mise profiles](https://mise.jdx.dev/profiles.html) to define CI tasks. Bonus: you can easily use it locally to debug pipeline issues.

See the [docs](https://mise.jdx.dev/tips-and-tricks.html#ci-cd) for a GitHub Actions example.

### Best Practices

1. Place `.mise.toml` in the root of your Git repository to define tools and tasks for the entire repository.
    
2. If a repository contains several different projects (like a monorepo), keep common items (like pre-commit tools) in the root config and project-specific items in the project directories.
    
3. Use your own `~/.config/mise/config.toml` for personal automation tasks.
    

## Conclusion

Mise is a unified tool and task management solution that simplifies the process of installing and managing tools for different projects. It supports various backends for tool installation and allows users to define tasks with custom scripts.

Mise can be used in both local development environments and CI/CD pipelines, serving as a single source of truth for tool versions and task flows.

Mise is an excellent alternative to `make`, `tfenv`, `direnv`, and whatever-else-env. [Try it!](https://mise.jdx.dev)