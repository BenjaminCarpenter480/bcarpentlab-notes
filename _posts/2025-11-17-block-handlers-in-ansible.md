---
title: Block Handlers in Ansible (Why they don't work and what you can do instead)
date: 2025-11-17 20:00:00 +0000
#categories: [TOP_CATEGORY, SUB_CATEGORY]
tags: [ansible, ansible-handlers, ansible-block, ansible-gotchas, ansible-playbook, ansible-tutorial, ansible-listen, ansible-include-tasks, devops, configuration-management]     # TAG names should always be lowercase
---

Block handlers are as it turns out not a thing, lets go through what a block handler is, why they as of writing do not work:

Say you have a block of code you would like to run as a handler in `roles/config/handlers/main.yml`

```yaml
# This example does NOT work
- name: Restart service
  block:
    - name: Stop Service
      ansible.builtin.service:
        name: my-service
        state: stopped

    - name: Start Service
      ansible.builtin.service:
        name: my-service
        state: started
```

Intuitively you would think that adding `notify: Restart Service` to the task you want to run the handler with would work but this is not the case due to how Ansible works with Blocks.  

>[*"Block names are for documentation only, they do not apply to the contained tasks"*](https://github.com/ansible/ansible/issues/77461#issuecomment-1088274545)

As such there are two choices you can follow, and I think the one you choose is mostly based on whereabouts the handler code in use is stored

## Listen attribute

Use of the `listen` attribute allows setting what should respond to a notify:

```yml
# This handler 'name' is now just for documentation.
# The 'listen' topic is what matters.
- name: Restart service
  block:
  - name: Stop Service
    <...>
    listen: Restart Service Handler
    
  - name: Start Service
    <...>
    listen: Restart Service Handler
```

Things marked listener will be respond when notified e.g. any tasks with `notify: Restart service handler` will call the above tasks

```yml
- name: Task which needs a restart after
  <...>
  notify: Restart Service Handler
```

## Include_tasks

If you have an Restart Service `tasks.yml` file then you can do the following:

`roles/tasks/restart_service.yml`

```yml
- name: Stop Service
  <...>
  
- name: Start Service
  <...>
```

Then in your handlers file you can have
`roles/config/handlers/main.yml`

```yml
- name: Restart Service
  include_tasks: roles/tasks/restart_service.yml
```

And then be able to use in your code and plays

```yml
- name: Task which needs a restart after
  <...>
  notify: Restart Service
```

## Conclusions

 Both these both seem quite convoluted so there is a lot of interest in updating Ansible to allow calls to a block as a handler (see discussion on [Stack Overflow](https://stackoverflow.com/questions/71578834/using-block-in-handler-ansible) and [GitHub](https://github.com/ansible/ansible/issues/14270) ) but in the meantime they are both standard patterns for grouping multiple tasks into a singular handler action.
