---
layout: post
title:  "A custom Ansible filter"
date:   2017-08-08 08:00:00 -0700
categories: ansible
---

Following up on my last post, I wanted to go into a little detail about the "camelize" filter I built for the template below. The [documentation][ansible_filters] on plugins in a bit sparse, but it's not too hard once you review a bit of how they do it in their [core module][ansible_core].

This particular filter is derived from the [inflection package][inflection]. I altered it slightly to handle dashes as well as underscores, as well as multiple consecutive separators.

The key bit here is the FilterModule, which must return the filter name in a "filters" method. In this case I built it as a method of the FilterModule, but you could just as easily create separate functions as they do in the core filters. Also, the filter needs to reside in a "filter_plugins" directory under the same location as your playbook:
```
~/ansible/playbook.yml
~/ansible/filter_plugins/camelize.py
```
The filter itself:
```
import re

class FilterModule(object):
    def filters(self):
        return {
            'camelize': self.camelize,
        }

    def camelize(self, s, uppercase_first=True):
        """ Converts string with dash and underscore characters to camel case

        Example:
        {{ 'my-string__of-chars' | camelize }} => MyStringOfChars
        {{ 'my-string-of-chars' | camelize(false) }} => myStringOfChars
        """
        if uppercase_first:
            return re.sub(r"(?:^|[_-]+)(.)", lambda m: m.group(1).upper(), s)
        else:
            return s[0].lower() + self.camelize(s)[1:]
```
Now if you had some variable like 'my-var_with_underscores', you could camel case it by passing in this filter. It would be written as 'myVarWithUnderscores'.

Also see [this helpful blog post][filter_blogpost] for some more on this.

[ansible_filters]: http://docs.ansible.com/ansible/latest/dev_guide/developing_plugins.html#filter-plugins
[ansible_core]: https://github.com/ansible/ansible/blob/devel/lib/ansible/plugins/filter/core.py
[filter_blogpost]: http://www.dasblinkenlichten.com/creating-ansible-filter-plugins/
[inflection]: http://inflection.readthedocs.io/en/latest/_modules/inflection.html
