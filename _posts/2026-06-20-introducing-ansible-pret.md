# Introducing ansible-pret

I'm writing this blog post to describe 2 recurrent problems I have faced an how I solved it using [ansible-pret](https://github.com/TC5027/ansible-pret)

Ansible inventory is a powerful concept because it lets you model a server fleet and define groups which can be used to either scope actions (aka playbooks) or factorize redundant information. Note that these 2 facets are usually complementary. Users have the freedom to choose the way they represent their fleet. But as we all know, with freedom comes the reviewing cost and I prefer having a way to define constraints one time instead of having to review every change introduced, just like I would define a comprehensive struct in typed programming language. Bonus : your colleagues don't get angry at your for creating endless threads in PR reviews; they get angry at their computer before asking a review !

## Storytime

Imagine a storage infrastructure consisting of servers distributed in datacenters which are organized as Ceph clusters. Ansible inventory lets you refer to a host in different subtrees of the main tree, allowing you to effectively model the complementary definition of a server. We have its physical location + the Ceph cluster it belongs to. Our inventory model would look something like this

```
root
|_location
  |_...
        |_serverA
|_cluster
  |_...
      |_serverA
```

We would have a group representing a datacenter, another for a rack inside it, another for a Ceph cluster etc. Then we would incorporate variables that help us in our daily life like phone number to contact the datacenter, number of units for a rack, Ceph version etc. Each variable must be defined in the correct group ! Otherwise we may face issue if we imagine a server transferred from one Ceph cluster to another, transferred from one rack to another etc. Such flaw could go unnoticed as the end result considering a host could be perfectly fine when it is introduced, only noticed when a migration occur. What I'm describing may seem obvious when inventory is initialized but as it evolves knowledge may go lost and hidden flaw introduced. This would never happen if you make use of [ansible-pret](https://github.com/TC5027/ansible-pret) (like I do !)

[ansible-pret](https://github.com/TC5027/ansible-pret) allows you to define constraints scoping a variable to a concept like datacenters or Ceph clusters which correspond to a portion/level of your inventory. Once done you can use same tool inside your CI to perform automatic checks on variable definition based on your constraints. Let's see how we achieved that. First we need to help ansible-pret reading your inventory, so the tool can understand this part is for location and first level is datacenter, second level is rack etc. This concept is named 'anchor' in ansible-pret jargon. We need to update a bit our inventory to define explicit roots

```
root
|_anchor_location
  |_location
    |_...
          |_serverA
|_anchor_cluster
  |_cluster
    |_...
        |_serverA
```

And then we can describe the different levels/depths relative to a root, for example

```
ansible-pret version-anchor -dir {} anchor_location
ansible-pret version-anchor-depth -dir {} datacenter 0 anchor_location
```

This design has several strenghts. Changes required on the inventory itself are minimal, we only have to define the root of the subtrees to ensure the tool can effectively read the inventory. Members of a given depth are automatically inferred, no need to explicitly enumerate all of them. Finally every new member of a depth in the inventory is automatically caught by the tool, there is no update required on the anchor, same for removals.

From this we can now define actual constraints for variables

```
ansible-pret version-anchor-rule -dir {} datacenter_contact datacenter 0 anchor_location
```

Constraints are stored inside the directory -dir and using ansible-pret analyze referring to this same directory, we are able to check each iteration only contains definition for the datacenter_contact variable inside the groups representing datacenters.

## Lifecycles

Another cool feature implemented in [ansible-pret](https://github.com/TC5027/ansible-pret) is lifecycle. The use case I had was for NFS mounts deployed on Linux servers. These linux servers are converged daily so we have a daily appliance of the ansible.posix.mount module fed by our inventory. At some point we had to decomission one filesystem and we used inventory as the source of truth for servers needing umount. This turned out to be wrong because some people removed mention of this mount in the inventory without umount by hand in the mean time meaning it just got left out as in on the host.. Of course I could have check the git history but it feels overcomplicated and basically states that inventory is not trustable. To tackle the additive behavior of Ansible we solved this problem by forcing people defining the mount state as absent inside the inventory before being allowed to remove mention of it.

```
ansible-pret version-start -dir {} mounted mount_state
ansible-pret version-transition -dir {} absent mounted mount_state
ansible-pret version-end -dir {} absent mount_state
```

Paired with the daily converge mechanism and use of ansible-pret to analyze iteration inside the CI and register it once validated, it turned out to be quite effective, providing clean lifecycle for mounts. This idea could be applied to other configuration like packages etc. but requires a specific execution pattern.

## Closing

Ansible inventory is clearly not a first-class citizen when it comes to being a safe and scalable source of truth at scale, see all the non native mechanism (also impacting execution) that should be added to harden it! While it may be a decent solution for small scale projects there is a point where you should consider additional safety like ansible-pret or move your source of truth to a more robust and feature-rich tool, maybe Netbox ? an API powered by a relational database with well defined schema ? You could even consider several sources that would best fit each element of your configuration, merged together producing an ephemeral Ansible inventory safe to use for your playbooks. I'm still searching for such open source solution encapsulating features developed in ansible-pret...
