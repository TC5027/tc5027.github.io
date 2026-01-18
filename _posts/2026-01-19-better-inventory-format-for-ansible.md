---
layout: post
title:  "Better inventory format for Ansible"
date:   2026-01-19
---

One thing I really like about Ansible is the **inventory** and being able to define **group** for hosts sharing some configuration. It provides flexibility but preserving it as inventory involves can be challenging. We all know how fun it is to review yaml (jk).
In this post I want to convince you that the existing inventory formats available for Ansible are not good enough because they only encode hierarchy and not intent.

## Storytime

Not so long ago I was part of a team managing Ceph clusters. We used Ansible for many tasks like post-boot configuration, cluster bootstrap etc. A fleet of hundreds of servers to manage and take care of.

## Variables

When you set up your inventory (and playbooks) you need to first establish what will be the **variables**. This could be:

* rack_id
* rack_slot
* datacenter_contact
* ceph_version

When registering a host you specify **values** for the variables and this is the **definition** of your hosts in your system.

## Groups

Using **groups** and listing hosts under it allows us to factorize many definitions. For example, defining a group corresponding to a rack and listing all the hosts in it, sharing same value for rack_id, makes sense.

## Poly-definition

In our example we notice that a host has several definitions to it like:

* its physical location 
* the cluster it belongs to

It is very likely that this is also the case for your own hosts !

Because of this __poly-definition__ our hosts are mentioned at different places in the inventory file, inside different **trees**.

```
all
|__Physical location tree
|____...
|________HostX
|__Cluster tree
|____...
|______HostX
```

This is perfectly fine and supported. Ansible allows one to combine different sources, static or dynamic, resulting in a rich inventory. 

## Benefits of using groups

Groups provide **flexibility**, let me illustrate.

Imagine because of customer's consumption we need to migrate capacity from one cluster to another.
This can be achieved only by moving hosts inside their cluster tree without update of their physical location.
But to achieve this flexibility you need to ensure variables are defined at the correct location.

This may seem obvious as you initialize an inventory but is easily put at risk because of many factors:

* inventory growing
* several people contributing  
* poor documentation
* reviewing yaml is boring

Imagine for example we had a definition for datacenter_contact inside the destination cluster group, all hosts migrated could end up with a false value for it.
This kind of misconfiguration can be completely hidden at a given point in time and, as inventory evolves in such way, be revealed and cause great damage. That's why we need a way to define constraints.

## Cartography

To answer this need and provide **automated** checks, we could set up a file containing for each level in the inventory what variables should be defined there (and only there).
We could also do it the other way around or include several levels where a definition is acceptable depending on your own use case.
This is what I call a **cartography**.

## Reading the inventory

I said __"for each level"__ but how can we actually read an inventory ? My actual inventory looked something like this:

```
all
|__equinix1
|____rack1
|____rack2
|__equinix2
|__cluster1
```

For a human reading this, it's kinda obvious that there are here 3 different levels corresponding to datacenter, rack and cluster. But this is not explicit enough for a computer.

If we rely on a list of groups to define a level, eg rack level = [rack1, rack2] , integrating a new group implies registering it inside the level definition which is not very convenient as the inventory grows.

One way to do so is relying on prefixes. Another way (my personal favourite) is to incorporate **descriptive groups** that act as metadata for groups under it. 

```
all
|__datacenter
|____equinix1
|______rack
|________rack1
|________rack2
|____equinix2
|__cluster
|____cluster1
```

Here datacenter, rack and cluster are fully descriptive groups, with no variables defined in them. They are just markers and what is below are the actual groups holding definitions.

With this, our cartography would look like:

```
datacenter : [datacenter_contact]
rack : [rack_id]
cluster : [ceph_version]
```

An example of implementation of this idea is available on [github](https://github.com/TC5027/ansible-mercator).

## Conclusion 

It seems to me the Ansible ecosystem lacks of support for such inventory format, more descriptive and allowing linkage between groups and variables. In my implementation I feel like I tweaked with yaml and ansible-inventory but I would prefer a standard approach to achieve the same.

Last thing : be very cautious using dict values for your variables as I have experienced part of the keys belonging to one level and other part to another. Use plain values as much as possible.
