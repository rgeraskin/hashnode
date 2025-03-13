---
title: "Mastering Terraform Debugging: Tips and Techniques 🔧"
seoTitle: "Terraform Debugging: Essential Tips and Techniques"
seoDescription: "Guide to Terraform debugging with these essential tips and techniques for using the Terraform console and more"
datePublished: Sun Jun 02 2024 12:50:13 GMT+0000 (Coordinated Universal Time)
cuid: clwxjhx89000108ju66k67cbn
slug: terraform-expressions-debugging
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1717332404288/ec29589f-831b-4e56-bb6c-9daa2f494ee5.jpeg
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1717335039636/d682c6a4-4802-46e9-8ef9-c573006ec9d6.jpeg
tags: devops, terraform

---

There is a wealth of information available about Terraform and the various tools within the Terraform ecosystem. However, there seems to be a noticeable gap when it comes to resources on debugging techniques specific to Terraform.

Let's address this issue and expand the knowledge base to include detailed debugging methods.

## Terraform Console

The built-in Terraform feature can make your life much simpler. It's often been avoided by engineers and is not so popular in blogs.

What can you do with the console? Basically, it can be used not only to show info about a resource in a state.

1. ### Drop state info
    

When you run `terraform console` in a Terraform project directory, it tries to use the state information to get actual details. It's useful for getting information about created resources or data objects.

But it sometimes slows down the debugging process. For example, if you place your state in an S3 bucket, it fetches it first. Tools like `terraform-repl` run `terraform console` under the hood for every command, making the process really slow. Also, `console` sets a lock on the state while you work with it.

To work with `console` without an S3 state, you can comment out the Terraform `backend` section, remove the `.terraform` folder, and rerun `terraform init`.

2. ### Test your expressions
    

Often we have to convert data between various formats to meet the requirements of module interfaces or for other cases. And we usually do it blindly, evaluating complicated expressions in our minds only. That leads to unexpected issues in some corner cases.

Just test it:

```plaintext
> coalesce(["", "b"]...)
b
```

Yes, it's a simple example from the tf docs, but you can use `console` for much more complicated things.

> Did you know that terraform console can be launched not only in a directory with a terraform project? To test expressions, you can start it in any directory.

3. ### Define your own `locals` interactively
    

One of the common drawbacks of terraform console is that it doesn't allow you to set your own variables or locals. To compose a complicated expression, it's handy to use intermediate local variables.

To make it possible, you can use terraform console wrappers such as [terraform-repl](https://github.com/paololazzari/terraform-repl). It also adds a tab-completion feature.

```plaintext
> local.a=1
> local.a+1
2
```

By using the `local` command, you can display all local variables.

There is another tool called [tfrepl](https://github.com/ysoftwareab/tfrepl) with similar functionality. However, it doesn't have tab-completion, and it can't show all of your defined `locals`.

4. ### Debug your modules
    

Modules are the way to keep your tf code [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself). But it's a pain in the neck when you debug and refactor it. Use console tools to make it simpler.

If you have default values for your variables defined, just run `terraform console` or `terraform-repl` to debug expressions.

If you have no default values, it's safe to place `terraform.tfvars` just inside the module folder. Terraform will ignore these vars when this module is used somewhere in a parent project.

5. ### Test your modules
    

The practice of testing Terraform code is not widely used in the infrastructure world. Actually, I've never seen tests in any single company :)

> Personally, I don't see any significant benefit from tests in the way they are meant to be used. See [this](https://developer.hashicorp.com/terraform/language/tests#example) example from the official docs: the test checks if the bucket's name is created as expected.
> 
> Do we really need it? We just use the variable in the bucket name, so it's obvious that nothing will happen to it later!

But we can use tests to make sure that we will not break a module's 'interface' sometime in the future while refactoring. Just write a test and do anything with local expressions and variables.

Stupid simple test example:

```bash
# main.tftest.hcl
run "test" {
  command = plan
  assert {
    condition = jsonencode(local.users) == jsonencode({
      "john" = {
        "grants" = null
        "login"  = true
        "member_of" = [
          "admin",
        ]
        "password_enabled" = false
      }
    })
    error_message = "Wrong format of local.users : ${jsonencode(local.users)}"
  }
}
```

```bash
❯ terraform test
main.tftest.hcl... in progress
  run "test"... pass
main.tftest.hcl... tearing down
main.tftest.hcl... pass

Success! 1 passed, 0 failed.
```

Again, if you already have default values in your `variables.tf`, you are ready to use tests. If you don't, place `terraform.tfvars` as stated above.

I don't recommend placing variables inside `*.tftest.hcl` in simple cases because you will not be able to use those variables with `console` later. Also, avoid complicated tests. Simple tests are easier to support. So, there is a chance that they will be supported in the future at least.

Testing expressions is the way to be confident that you will have the expected value in your resource property.

## Conclusion

I shared a few tricks on debugging Terraform expressions. Do you have any to add? Share them in the comments.