---
title: "About"
permalink: "/about/"
layout: page
mathjax: true
---

These notes are on GitHub in [Jekyll][1] so I do not worry over a web server.

[1]: https://jekyllrb.com/

## Corrections

If you notice something wrong or outdated [please submit it as an issue][2]
(or even better as a pull request with the improvement you imagine).

[2]: https://github.com/AE1020/AE1020.github.io/issues

## Theme Used

The theme is called ["Contrast"][3] and it is by [Niklaus Buschmann][4]. It was
simple and looked good. Also it comes with support for equations, which you can
enable with just `mathjax: true` in the page.

$$ f^{(n)}(z) = \frac{n !}{2 \pi i} \int_{C}\! \frac{f(\zeta)}{(\zeta - z)^{n+1}} \mathrm{d}\zeta $$

Props.

[3]: https://jekyllthemes.io/theme/contrast
[4]: https://github.com/niklasbuschmann

The instructions suggest forking the repo, but I just cloned it and then pushed
to an empty repository. That way I can search it in the web UI (since at time of
writing GitHub does not search forks).

I'll mention this uses the [Liquid][5] templating system so I can find the
documentation later when I need it.

[5]: https://shopify.dev/docs/themes/liquid/reference

{% comment %}
         1         2         3         4         5         6         7         8
12345678901234567890123456789012345678901234567890123456789012345678901234567890
{% endcomment %}
