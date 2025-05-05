---
title: "Introducing GoDiffYAML Tool 💪"
datePublished: Mon May 05 2025 23:04:33 GMT+0000 (Coordinated Universal Time)
cuid: cmabou0vu00050ajlfhgu9ahu
slug: godiffyaml
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1746485562551/5124cff8-014b-4550-bea3-43f79a0050a7.png
tags: kubernetes, devops, diff, yaml, k8s

---

## The YAML Diffing Dilemma

YAML files are everywhere in software development — think Kubernetes manifests, Ansible playbooks, or any configuration management system. But when these files contain multiple documents (separated by `---`), comparing changes with traditional diff tools becomes a mess.

These tools treat the file as a single blob of text, often producing confusing or outright incorrect diffs. This problem gets worse with large YAML files packed with many documents, making it nearly impossible to spot specific changes.

That’s why I created [**godiffyaml**](https://github.com/rgeraskin/godiffyaml), a tool designed to solve this exact issue. Whether you’re a developer, DevOps engineer, or system admin, **godiffyaml** makes diffing multi-document YAML files painless.

## Why Standard Diff Tools Struggle

Here’s what makes diffing multi-document YAML tricky with standard tools:

1. **Document Order Awareness:** If a document is moved within a YAML file, standard diff tools display it as a completely new document. Any changes inside it might be overlooked.
    
2. **Document Awareness**: Tools like `diff` don't recognize YAML's `---` separators, so changes across documents get mixed up.
    
3. **Values Notation Awareness**: According to the YAML specification, the way values are noted is quite flexible. For example, strings can be quoted or not. However, for diff tools, these differences still appear as changes.
    
4. **Nested Structure Awareness**: YAML’s hierarchical structure—nested keys and values—means small changes can drown in a flood of irrelevant differences.
    

I built **godiffyaml** to address these pain points head-on, offering a smarter way to compare multi-document YAML files.

Let's explain the issues and how **godiffyaml** solves them in detail.

### Document Order Awareness

A basic YAML file with random Kubernetes manifests:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  DB_HOST: "localhost"
  DB_PORT: "5432"
  CACHE_TTL: "3600"
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
  namespace: default
spec:
  ports:
    - port: 80
      targetPort: 3000
  selector:
    app: frontend
```

Now reverse the order of the documents: place `kind: Service` before `kind: ConfigMap` and then compare the differences.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1746474180299/fd0e8f09-ffba-4eb3-b8b7-5319b6c2e6ec.png align="center")

Can you quickly tell if the `ConfigMap` has just been moved in the YAML, or if there are additional changes? Imagine having many such "movements" inside; reviewing them would become a nightmare.

With **godiffyaml**, you can sort YAML files by path, such as `.kind`, and compare them with any standard tool.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1746477055917/6532afa8-ce08-4e98-a46f-86320b51b7cf.png align="center")

Or just

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1746478594428/7ff63695-2407-4837-abbc-5d08eadd3a28.png align="center")

### Document Awareness

Let's change the value of `PGHOST` from `host2` to `host-changed` in `app2-secret` within a YAML file containing two secrets, and then compare the original with the modified version.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app1-secret
  namespace: default
stringData:
  PGDATABASE: db1
  PGHOST: host1
  PGPASSWORD: pass1
  PGUSER: user1
---
apiVersion: v1
kind: Secret
metadata:
  name: app2-secret
  namespace: default
stringData:
  PGDATABASE: db2
  PGHOST: host2 # this will be changed
  PGPASSWORD: pass2
  PGUSER: user2
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1746478871250/4e08f4ef-b012-482b-9e84-62e828db6263.png align="center")

Can you guess which secret has changed? To find its name, you need to expand the context for the difft tool or check the full diff in your favorite diff tool. Now, imagine there are many such changes across several documents.

**godiffyaml** solves this problem:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1746479236924/708c9492-da21-4f9f-bf40-9394558b54df.png align="center")

See the name in a rectangle? It's templated from the `-paths` flag: `<.kind>_<.metadata.name>.yaml`.

### Values Notation Awareness

Let’s compare the same YAML files, but the first one will have unquoted strings and the second one will have quoted strings. According to the YAML specification, they are considered the same, but your IDE probably doesn't see it that way.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1746475336123/d5ea3a57-38cb-499c-af9d-e254a3201185.png align="center")

It's just distracting noise because it's YAML with the same values.

Just like the *'No Document Order Awareness'* example above, we can either use `godiffyaml sort` and compare with a usual tool or simply use `godiffyaml diff`.

But now let's use the `k8s` subcommand: `godiffyaml k8s` is a shortcut for `godiffyaml diff -paths apiVersion,kind,metadata.namespace,metadata.name`. Notice that the dot at the beginning of each path is optional.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1746480194333/52dadd48-ffe2-46e9-97ec-0e9ae552118d.png align="center")

Note that we don’t use the `-order` flag for the `sort` subcommand. Details are provided below in the *‘Magic’* section.

### Nested Structure Awareness

Typical diff tools show changes for lines, not values. However, it's useful to see specific differences. [Difftastic](https://github.com/Wilfred/difftastic) addresses this issue, and **godiffyaml** uses it behind the scenes.

> Difftastic — a structural diff tool that compares files based on their syntax. It’s a great tool that shows differences only for elements that changed, not the lines.

## How GoDiffYAML Works

It has two subcommands: `sort` and `diff`. Let's break them down first.

### Sort subcommand

1. Reads a YAML file.
    
2. Sorts documents inside by YAML paths **if** the `-order` flag is provided.
    
3. Outputs the YAML file with all documents to stdout.
    

### Diff subcommand

1. Reads YAML files.
    
2. Saves each document in a temporary directory with names based on path values.
    
    *E.g. for* `paths=apiVersion,kind,metadata.namespace,metadata.name` *document name will be like* `apps_v1_Deployment_default_backend-api.yaml`
    
3. Runs **difftastic** on the directories to compare them.
    

### 💫 Magic 💫

All the magic inside is based on two ideas:

1. By reading and then dumping YAML with the same Go YAML library, we ensure that YAML documents have consistent value notation and key ordering. This applies to both `sort` and `diff` modes.
    
2. By saving files with templated names, we can compare directories with **difftastic**, which will display the file names. The file names help us identify documents due to the templating.
    

The structural diff relies entirely on **difftastic** because I don't want to reinvent the wheel. So, we need **difftastic** to use the `diff` or `k8s` subcommands.

Additionally, any extra flags for **godiffyaml** are passed to the **difftastic** backend.

## Why You’ll Love GoDiffYAML

* **Saves Time**: No more digging through messy diffs to find what changed.
    
* **Reduces Errors**: Clear, structured output means fewer mistakes when reviewing updates.
    
* **Handles Scale**: Works seamlessly with large, complex YAML files.
    

## Wrap-Up

If multi-document YAML files are part of your daily grind, **godiffyaml** is here to make your life easier. By respecting YAML’s structure and delivering precise, readable diffs, it turns a frustrating task into a breeze. Give it a try — I think you’ll wonder how you ever managed without it!

So go diff that YAML💪