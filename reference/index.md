---
title: Command reference
permalink: /reference/
---

# Command reference

The commands, split by who runs them.

| Page | Commands | Who runs them |
|---|---|---|
| [kvo commands]({{ '/reference/kvo/' | relative_url }}) | `kvo`: objects, silos, panes, sets, secrets, addresses, images, plugins for `kvo`, sign-in and contexts | Everyone who uses the cluster, with the role they were given |
| [kvoadm and kevasod commands]({{ '/reference/kvoadm/' | relative_url }}) | `kvoadm`: machines, ingress classes, people, keys, backups. `kevasod`: the program on the hosts | Administrators, over SSH and as the cluster admin |

`kvoadm kvo …` runs any `kvo` command with the admin certificate in the cluster folder: that is how
an administrator does the steps a user page marks as an admin's.

<script>
  // Old links pointed at sections of one page: send them to the page that has the section now.
  (function () {
    var id = location.hash.slice(1); if (!id) return;
    var admin = /^(kvoadm|kevasod|machines|ingress|people|keys|backups)/.test(id);
    location.replace("{{ '/reference/' | relative_url }}" + (admin ? "kvoadm/" : "kvo/") + location.hash);
  })();
</script>
