---
title: "Terramate meets Atlantis 🚀"
seoTitle: "Terramate and Atlantis"
seoDescription: "Integrate Terramate with Atlantis for seamless Terraform code generation and pull request automation. Enhance your DevOps workflow effortlessly"
datePublished: Wed Apr 03 2024 20:10:50 GMT+0000 (Coordinated Universal Time)
cuid: cluk8tfw3000308kz0xwb5g0v
slug: terramate-atlantis
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1712174792674/d98e45b2-88d5-42d7-8744-ca64ecd39f3f.jpeg
tags: devops, terraform, atlantis, terramate

---

[Atlantis](https://www.runatlantis.io) is a pull request automation tool that works well with plain Terraform right away. But what if we're already using [Terramate](https://github.com/terramate-io/terramate) to generate Terraform code?

> Below, I assume the use of the official Atlantis Helm chart for deployment.

**1\. Add Terramate binary**

Add the following to the Atlantis chart's `values.yaml` to download the binary and mount it to the Atlantis pod. This allows Atlantis to use it for generating Terraform code.

```yaml
initContainers:
  - args:
      - >-
        curl -L https://github.com/terramate-io/terramate/releases/download/v${TERRAMATE_VERSION}/terramate_${TERRAMATE_VERSION}_linux_x86_64.tar.gz | tar xz
    command:
      - sh
      - -c
    env:
      - name: TERRAMATE_VERSION
        value: 0.4.5
    image: curlimages/curl
    name: get-terramate
    volumeMounts:
      - mountPath: /home/curl_user/
        name: terramate
    workingDir: /home/curl_user/
  extraVolumes:
    - name: terramate
      emptyDir: {}
  extraVolumeMounts:
    - name: terramate
      mountPath: /usr/local/bin/terramate
      subPath: terramate
      readOnly: true
```

**2\. Use server-side config**

Let's use server-side configuration for our repository. Therefore, add more values to the `values.yaml`:

```yaml
environment:
  ATLANTIS_REPO_CONFIG: /etc/atlantis/server-side-config.yaml
extraVolumes:
  - name: atlantis-server-side-config
    configMap:
      name: atlantis-server-side-config
extraVolumeMounts:
  - name: atlantis-server-side-config
    mountPath: /etc/atlantis/server-side-config.yaml
    subPath: server-side-config.yaml
    readOnly: true
```

**3\. Deploy server-side config**

Now, place the server-side configuration in a configMap and deploy it to the Atlantis namespace:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: atlantis-server-side-config
  labels:
    # I deploy it with the Atlantis chart, so here is helm templating
    app: {{ .Chart.Name }}
    chart: {{ .Chart.Name }}-{{ .Chart.AppVersion }}
    release: {{ .Release.Name }}
    heritage: {{ .Release.Service }}
data:
  server-side-config.yaml: |
    repos:
    # this steps run before every workflow. 
    # So we will generate tf-code here
    - pre_workflow_hooks:
        # to run 'safeguards' terramate wants origin master 
        #   and some previous commits
        - run: |
            git config --add remote.origin.fetch +refs/heads/master:refs/remotes/origin/master
            git fetch --depth=1 origin master
            git fetch --depth=2
          description: Fetch origin master branch
        
        # this step is optional
        # I use .tm-run-order later to made Atlantis 
        #   to run plan/apply in a specified order
        # Also, I generate an atlantis.yaml config 
        #   with terramate too, so I exclude it from run order: 
        #   it's just a yaml, not tf-code
        - run: terramate list --run-order -c --no-tags atlantis -B origin/master > .tm-run-order
          description: Get changed stacks
        
        # And the obvious final step =)
        - run: terramate generate
          description: Generating tf code
      # some other options, offtopic
      id: github.com/XXX/YYY
      apply_requirements: [mergeable, approved, undiverged]
      delete_source_branch_on_merge: true
```

**4\. (Optional) Generate** `atlantis.yaml` with Terramate too

Add this 'stack' to your Terramate repository:

```ini
stack {
  name        = "atlantis"
  description = "atlantis"
  id          = "your-uniq-id"

  tags = [
    "atlantis"
  ]
}

generate_file "atlantis.yaml" {
  condition = (
    # I place it in the root repo dir so I want to avoid 
    #   file generation with child stacks, only with
    #   atlantis stack and with not empty .tm-run-order
    terramate.stack.name == "atlantis" && tm_length(let.run_order) > 0
  )

  lets {
    run_order = tm_try(
      tm_compact(tm_split("\n", tm_trim(tm_file(".tm-run-order"), "\n"))),
    [])
    # dirty magic to fill config options by my folder structure
    projects = [for x in let.run_order : {
      name = tm_replace(x, "/^terraform/(projectX/)?/", "")
      dir  = x
      autoplan = {
        # I want to trigger Atlantis for every project
        #   because I already know that every project 
        #   here is modified
        when_modified = (
          tm_substr(x, 0, 14) == "terraform/projectX/" ? ["../../../**/*"] : ["../**/*"]
        )
        enabled = false
      }
    }]
    config = {
      version        = 3
      automerge      = true
      parallel_plan  = false
      parallel_apply = false
      projects       = let.projects
    }
  }

  content = tm_yamlencode(let.config)
}
```

Done! Now, for every PR, Atlantis will:

1. Run pre-workflow hooks to generate Terraform code.
    
2. (Optional) Generate its own configuration to run projects in the desired order.
    

If you're looking for more tips and useful info, definitely swing by my blog post [10 Useful Aliases and Functions for Terramate and Zsh Users](https://rgeraskin.hashnode.dev/terramate-zsh).